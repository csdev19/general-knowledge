# WebCodecs export encoding for screen recordings

*How to get small, readable exports of screen recordings out of the browser's encoder (WebCodecs,
usually through a library such as mediabunny) before reaching for a bundled FFmpeg: the two
settings that matter, the ones that do nothing, and how to measure it.*

Screen recordings are unlike camera video: most of the frame is static, and the content that
matters is small, high-contrast text. Encoder defaults are tuned for camera video and waste most of
their bits on a screen. The rules below came from one measured benchmark (see the example at the
end); treat the numbers as an example, and re-measure on your own content and machines.

## The rules

### 1. Raise the keyframe interval

A keyframe is a full picture; the frames between keyframes store only what changed. On a static
screen, the frames between keyframes are almost free, so **the keyframes are the file**.

- mediabunny's `VideoEncodingConfig.keyFrameInterval` defaults to **2 s** (its conversion API
  defaults to 5 s). WebCodecs itself has no interval; the caller decides per frame
  (`encode(frame, { keyFrame: true })`), and mediabunny does that for you.
- For a shared screen recording, **10 s** is a reasonable start. x264's default (`keyint=250`)
  is ~10 s at 24 fps.
- The cost of a longer interval is coarser seeking, which rarely matters for a short clip watched
  start to finish.

### 2. Use constant quantizer instead of a target bitrate

A target bitrate (`bitrateMode: 'variable'`, the WebCodecs default) asks the encoder to spend
roughly N bits per second whether the screen changed or not. **Quantizer mode**
(`bitrateMode: 'quantizer'`) fixes the quality per frame instead — the same idea as x264's CRF —
so a static screen costs almost nothing and motion gets the bits it needs.

- In WebCodecs: `configure({ ..., bitrateMode: 'quantizer' })` with no `bitrate`, then pass the
  quantizer per frame: `encode(frame, { avc: { quantizer: 30 } })` (`hevc`, `vp9`, `av1` keys for
  the other codecs). Lower means better and bigger; AVC/HEVC take 0–51.
- mediabunny ≥ 1.52 wraps this: `quality: new Quality({ quantizer: 30, bitrate: 2_000_000 })`,
  where the bitrate is the fallback when the platform encoder cannot do quantizer mode
  ([announcement](https://mediabunny.dev/blog/quantizer-support)). Older versions need
  `onEncoderConfig` to rewrite the config plus the per-frame option.
- **Support is per encoder, not per browser.** Probe it with
  `VideoEncoder.isConfigSupported({ ..., bitrateMode: 'quantizer' })` and fall back to a bitrate
  (keep the long keyframe interval) when it says no.
- **Quantizer mode does not cap size.** If a destination has a hard limit, you need an estimate
  per content type, a short probe encode, or the bitrate fallback.

### 3. Prefer the hardware encoder, and check what you got

`hardwareAcceleration: 'prefer-hardware'` is a hint. If no hardware H.264 encoder is reachable,
Chromium falls back to its software H.264 encoder, **OpenH264**, which is tuned for real-time
calls: on small text it produces garbled glyphs, it may ignore the requested bitrate, and it does
not support quantizer mode. Log the final config (`onEncoderConfig` in mediabunny) and treat an
OpenH264 fallback as a degraded path that deserves its own decision.

### 4. Do not count on hints

On macOS VideoToolbox through Chromium, `contentHint: 'text'` / `'detail'` and
`latencyMode: 'quality'` produced no measurable change (`quality` is already the default). Other
platforms may differ; measure before relying on them.

### 5. Pick the frame rate for the motion, not for the text

At a fixed bitrate, halving the frame rate doubles the bits per frame, so 15 fps looks sharper
than 30 fps in a still crop. But anything animated — zooms, cursor highlights, scrolling — visibly
stutters at 15 fps. **24 fps** is a safer floor for demo-style recordings. In quantizer mode the
extra frames are cheap when little changes.

### 6. Do not lower the resolution to save size

Downscaling a 1440p screen to 720p blurs small text more than any bitrate saving recovers.
Keep 1080p and raise the quantizer instead.

### 7. Ship H.264 unless you control the player

HEVC is ~20% smaller at the same quantizer on hardware encoders, but Chrome and Firefox only play
it with hardware decode. VP9 and AV1 software encoders in the browser are no better than hardware
H.264 on screen content and have their own playback gaps (check caniuse for
[HEVC](https://caniuse.com/hevc), [WebM](https://caniuse.com/webm), [AV1](https://caniuse.com/av1)).

## Before bundling FFmpeg

The usual reason to bundle FFmpeg is x264's better compression. Try the settings above first and
compare at **equal frame rate and equal keyframe interval** — most of the gap is the defaults, not
the encoder. Bundling FFmpeg costs, in summary:

- **license** — x264 is GPL; your app's license must be compatible, and you must provide source
  for what you ship. An LGPL FFmpeg build without x264 usually has no better H.264 encoder than
  the platform one;
- **patents** — distributing your own H.264 encoder may bring the AVC patent pool into scope,
  where the platform encoder did not (get legal advice);
- **size** — tens of MB per architecture for a full build;
- **signing** — every bundled executable must be signed and notarized on macOS;
- **ffmpeg.wasm** avoids binaries but is 25–50× slower than native.

## How to measure it

- Run the encodes **inside the real runtime** (for Electron, a hidden `BrowserWindow` in the
  shipped Electron), not in Node or with ffmpeg's wrapper of the same hardware encoder: the code
  paths differ (for example, WebCodecs' VBR on VideoToolbox undershot its target where ffmpeg's
  `h264_videotoolbox` hit it).
- Feed every encoder **identical pixels**: render one near-lossless intermediate, encode all
  variants from it, and build the metric reference through the same canvas path.
- Use VMAF/SSIM/PSNR for ranking, but **VMAF is not trained on text**: also crop small text right
  after a screen change and zoom it 2× (nearest-neighbour). Then watch the full files: stutter
  never shows in a still crop.
- Report sizes in decimal MB and say which: `ls -lh` prints MiB, and a 4% gap between them is
  larger than some of the differences you are measuring.
- Write down the machine and OS. Hardware encoders differ by vendor and generation.

## Measured example: Kaipu, October 2026

Kaipu (an Electron screen recorder exporting through mediabunny 1.49 → WebCodecs → VideoToolbox)
measured this on an Apple M5, macOS 26.6.2, Electron 39.8.10, with two 2560×1440 recordings: a 78 s
web-app demo with zooms and a 171 s terminal session.

| Recording, 1080p, video only | Old settings (2 Mbps VBR, 2 s keyframes, 15 fps) | Quantizer 30, 10 s keyframes, 24 fps | x264 `-preset slow -crf 28 -g 240`, 24 fps |
| --- | --- | --- | --- |
| Web demo, 78 s | 9.59 MB | 6.96 MB | 7.89 MB |
| Terminal, 171 s | 18.70 MB | 5.95 MB | 6.82 MB |

- The 10 s keyframe interval alone made the terminal clip 2.7× smaller at the same quantizer.
- At quantizer 28 the hardware path was 5–7% larger than x264 with no visible text difference;
  at 30 it was ~12% smaller and very slightly softer.
- Unmeasured: Windows (Media Foundation: NVENC, Quick Sync, AMF), Linux, Intel Macs, and
  fast-motion content.

Full record: Kaipu's
[encoding benchmark 2026-10](https://github.com/niway-dev/kaipu-record/blob/main/apps/documentation/src/content/docs/architecture/encoding-benchmark-2026-10.md).

## Related

- [media-pipeline.md](./media-pipeline.md) — the capture side of the same pipeline, including the
  hardware-encode hint and the OpenH264 caveat for live capture.
- [electron-vs-tauri.md](./electron-vs-tauri.md) — why the encode pipeline, not the framework, is
  the real performance question.
