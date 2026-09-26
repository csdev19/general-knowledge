# Local speech-to-text in a desktop app: the engine shape decides the architecture

Two engine families dominate offline transcription, and they are not interchangeable
components behind one interface. They fail differently, expose different confidence
signals, and need different defences. Choosing one is an architectural decision, not a
model swap.

Everything below is measured on one machine (Apple M4 Pro, 24 GB) with corpora and
hashes stated, because published WERs are measured on whole files and an interactive app
never sends a whole file.

## Both engines must be child processes

An N-API speech addon cannot run inside Electron. `sherpa-onnx-node` throws

```
External buffers are not allowed
```

and `utilityProcess` runs the same Node, so it fails identically. The addon path is not
merely slower — it is unavailable.

This has a consequence worth stating early, because it invalidates benchmarks: **measure
through the transport the app will actually use.** An in-process addon measurement is a
number your product can never reproduce. Measured across the same weights, transport was
worth ~0.12 pp WER — small, but it is not zero, and it sits inside the margin most engine
comparisons are decided by.

So both families run as sidecars, and the practical difference is their protocol:

| | Encoder-decoder (whisper.cpp) | Transducer (sherpa-onnx + Parakeet) |
| --- | --- | --- |
| Sidecar | `whisper-server`, HTTP multipart | `sherpa-onnx-offline-websocket-server`, binary frames |
| Readiness | `/health` answers 503 while loading | no health endpoint — a log line, then a real decode probe |
| Bad input | closes the connection | **the process exits** |
| Language | a request parameter you must send | **no parameter exists** |

Two of those rows are load-bearing.

**No health endpoint means readiness must be a decode.** An open port proves the socket
is listening, not that weights are loaded. Send 100 ms of silence and wait for a reply.

**A malformed payload killing the process** means the caller must treat an exit as the
engine dying, and cannot distinguish "bad frame" from "crash". Get the wire format right
with a golden fixture rather than by probing a running server.

### The binary frame, since it is undocumented in prose

Per utterance: a 4-byte little-endian sample rate, a 4-byte little-endian **audio byte
count** (`samples.length * 4`, excluding the 8-byte header), then float32 samples. Frame
it in chunks and wait for the JSON reply before closing. A one-second golden fixture —
16,000 samples, 64,000 audio bytes — catches every field-order and endianness mistake at
once.

## The hallucination problem is shaped by the engine family

An encoder-decoder model has a language-model head. Given audio with no speech in it, it
still produces its most likely sentence. In one real meeting, **77 of 116 microphone
segments were "Thank you."** — from typing and knocks on a desk.

Three defences, and only one works:

1. **An energy VAD** (RMS over a frame) opens a segment for a keystroke. A keystroke has
   plenty of energy. Useless here.
2. **The model's own non-speech probability.** whisper.cpp exposes `no_speech_prob`, and
   it is the obvious fix. **Measured inert:** ~1e-10 for silence *and* for real speech,
   so no threshold separates them. Do not "fix" hallucination by lowering it.
3. **A neural VAD upstream of the model** (Silero). This works, because it stops the
   audio from ever reaching the model — there is no text to filter because none was
   generated.

A junk-phrase list is not a fourth defence. "Thank you." is something people say, and a
list that drops it deletes real speech.

### Transducers invent less, not never — and they tell you when

A transducer has no LM head, so it has less to hallucinate *from*. Given the same eight
non-speech clips with no gate, one produced text on four of eight; the other checkpoint
of the same model produced text on three of eight. Milder failures — a filler word, not a
sentence — but `"Okay."` from a keyboard is a word a downstream LLM reads as agreement.

What the family *does* give you is a per-token log probability, and its mean over a
segment separates invented text from real speech:

| Input | Text | Mean log prob |
| --- | --- | --- |
| Quiet noise | "Okay." | **-0.474** |
| Typing, 0.5 s | "Okay." | **-0.547** |
| Single knock | "Uh" | **-1.015** |
| Real "Yeah." | correct | -0.091 |
| Real 10 s sentence | correct | -0.047 |

Everything invented below -0.45, everything real above -0.10. A threshold near -0.3 sits
in the gap with roughly 3× headroom on both sides.

**The finding that makes this the right defence:** the engine invented `"Yeah."` from a
breath and from a knock, and a real `"Yeah."` is the same string. No text filter, pattern
or LLM judge can separate them. Only the number can.

Four rules for implementing it:

- **Absent is "no opinion", never zero.** An engine without the signal must pass through
  untouched; zero reads as perfect confidence, and a low default deletes everything.
- **Average every token, punctuation included** — or change the threshold in the same
  commit. Punctuation genuinely scores lower (a comma measured -0.365 among word tokens
  at -0.0001, because the audio does not contain it), so excluding it is defensible and
  it silently invalidates any threshold measured the other way.
- **A non-finite entry voids the mean**, rather than being skipped. One `-Infinity` drags
  an average to nonsense and deletes real speech on a parsing artefact.
- **Log every drop**, with the reason and the score. A threshold calibrated on twelve
  clips must be observable; a transcript with a hole in it says nothing about why.

### A gate is not portable between sidecars

The neural VAD above is a *flag on the encoder-decoder's server* (`--vad --vad-model`).
The transducer's server has **no VAD option at all**, and the project ships its VAD only
in file-in/file-out demo binaries. So "we already have a speech gate" is false the moment
the engine changes. The options are a process launch per segment (against a ~150 ms
decode, the gate costs more than the transcription), a second resident sidecar you build
yourself, or the confidence signal the engine already returns for free.

## Language is a property of the engine, not a setting

An encoder-decoder takes a language parameter. A transducer of this family takes **none** —
it decodes whatever it hears among the languages it knows, and reports no detected
language (`"lang"` comes back empty).

This is a UI contract, not a detail. A "spoken language" control that shows "English" for
an engine that was never told English, and may not answer in it, is lying. Show the lock
only for the engine that honours it, and "Automatic" for the one that decides.

Two more consequences:

- **Per-segment automatic detection is a trap on the parameterised engine.** Short or
  noisy clips get misread, and Spanish speech comes back as an English *translation*.
  Pinning the language is the safer default.
- **A benchmark must send the right language.** A Spanish run with `language=en` measures
  translation, not transcription — it understates that engine and flatters whatever it is
  compared against.

## Coverage claims: enumerate, never `"*"`

A 25-language model is not an "all languages" model. Speech outside its set does not
error — it returns plausible text in one of the languages it knows. Store the tags, so a
per-language question can be answered honestly and the UI can say "25 languages".

An English-only checkpoint fed Spanish is the same failure at full volume: **82% WER**, and
not as errors:

> ref: *El elemento del determinismo cultural se encontraba muy presente en el romanticismo…*
> out: *The elemento del determinismo cultural determinism contramo y present in romantímo…*

This kills the obvious optimisation. If one checkpoint is better at English and another
covers many languages, routing by declared language looks free — but the engine is
typically fixed for a session's lifetime, so a session declared "English" turns any other
language spoken inside it into that. For a code-switching user the arithmetic is
inverted: you trade a fraction of a point of English WER for a catastrophic failure
whenever they switch.

## Benchmark discipline

The shape of the measurement matters more than the corpus.

**Cut segments the way the app cuts them.** A whole-file benchmark measures something the
product never does, and hides both the per-segment wait a person feels and the text an
engine invents from a noise-only segment.

**Use a parallel corpus for cross-language questions.** FLEURS holds the same sentence ids
in every language, so two prepared corpora differ by language and not by content. That is
what makes "this engine hears Spanish better than English" a statement about the engine —
measured at 4.54% against 9.22% on the same 300 translated sentences, which is not what
anyone expects from a multilingual model and is invisible without parallel text.

**Pin identity, not names.** Three failures, all observed:

- A benchmark script held a *copy* of the model directory name and kept pointing at the
  previous checkpoint after the app moved on. It would have reported old weights under the
  new name. Read the path from the application's own source.
- A tool reported `vad=1` while starting the server **without** the gate, because the
  weights were missing and the check was a silent `&& -f`. A hallucination measurement
  would have compared a gated engine against an ungated one and called the difference a
  model property. **A missing dependency must fail loudly**; an ungated control arm is
  something you ask for explicitly.
- Two runs of the same engine and model differed by 0.15 pp with nothing in the record
  able to explain it, because the machine, thread counts and model bytes were never
  stored.

So store, beside every number: machine and OS, tool versions, thread counts, and the
**SHA-256 of each model file**. A version string is not identity.

**Make the corpus verifiable, not merely rebuildable.** Audio does not belong in git, and
it does not need to: public corpora come from their publishers and synthetic fixtures from
a seed. What is missing without a manifest is proof that a rebuild produced the *same*
corpus — otherwise "engine X scores 2.45%" is a claim nobody can contest, because nobody
can confirm which clips produced it. Hash every segment plus the reference into one
`corpusDigest`, ~24 KB for 300 clips against ~68 MB of audio, and quote the digest wherever
the number is quoted.

Pin the download too. Fetch with a publisher-published hash where one exists — Hugging
Face serves git-LFS oids in its file listing, which is a stronger claim than a locally
computed digest — and pin a dataset *revision*, never `main`, because a corpus that moves
under a benchmark invalidates every number measured against it.

**Write down what a run costs.** A three-engine, two-language comparison at 300 segments
is about 30 minutes and 1 GB of downloads, most of it the GPU engine saturating the
machine. A suite that expensive gets re-run to recall a number unless the number is
written down — which is the actual reason the results belong in a durable document. And
cut the sample before cutting the rigour: if the gaps you are looking for are larger than
a point, a third of the clips will show them.

## Choosing, and what the numbers looked like

| Corpus | Encoder-decoder (large, GPU) | Transducer, English-only | Transducer, 25-language |
| --- | --- | --- | --- |
| LibriSpeech, English | 3.75% | **1.85%** | 2.45% |
| FLEURS, English (paired) | — | **6.00%** | 9.22% |
| FLEURS, Spanish (paired) | 6.91% | 82.18% | **4.54%** |
| Median latency / segment | ~1150 ms | ~200 ms | ~230 ms |

Read with its caveats: the encoder-decoder ran **without** its neural gate (weights absent
on the machine), and its 182 Spanish insertions against the transducer's 39 are exactly
where a gate would help, so its numbers are a ceiling on its error rate, not a floor.

What generalises:

- **A transducer on CPU beat a large encoder-decoder on GPU, in both languages, at a
  third of the latency.** For interactive transcription the latency difference is felt and
  the accuracy difference was not a trade.
- **A multilingual checkpoint can regress on its predecessor's single language** — here on
  two independent corpora, worse on the harder one, plus two whole utterances lost where
  the older checkpoint transcribed correctly. A lost utterance is worse than a misheard
  word: there is no wrong text to notice.
- **Coverage is bought, not free.** Decide explicitly whether the languages are worth the
  regression, and write the number down rather than discovering it later.
- **Keep the replaced engine installed through one release and a rehearsed rollback.** Its
  cost while unused is disk, because only one engine runs per session.

## Checklist

- [ ] Run the engine as a sidecar; an N-API addon is unavailable inside Electron.
- [ ] Benchmark through the transport the app uses, not an in-process addon.
- [ ] Make readiness a decode probe where there is no health endpoint.
- [ ] Treat a sidecar exit as engine death; pin the wire format with a golden fixture.
- [ ] Put the speech gate upstream of the model, or use a measured confidence signal —
      never a junk-phrase list, never the model's own non-speech probability without
      measuring it first.
- [ ] Absent confidence means "no opinion"; log every drop with its score.
- [ ] Never claim `"*"` languages; enumerate the tags.
- [ ] Show a language lock only for an engine that is actually told the language.
- [ ] Send the right language in a benchmark, or you measure translation.
- [ ] Read model paths from the app's source, never a copy in the benchmark.
- [ ] Fail loudly on a missing gate model; an ungated arm is requested explicitly.
- [ ] Store machine, tools, thread counts and model file hashes beside every number.
- [ ] Ship a corpus manifest with one digest; quote it wherever the number is quoted.
- [ ] Pin corpus downloads by revision and publisher hash.
- [ ] Record what a run costs, and prefer the cheapest arm that answers the question.
