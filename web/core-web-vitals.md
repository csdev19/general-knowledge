# Core Web Vitals: measuring honestly

Optimizing a page you have not measured correctly is guessing with extra steps. This page
is about the instrument and the conditions — what a number has to be taken under before it
means anything, and what makes a series of numbers comparable over time.

For what to *do* once the numbers point somewhere, see
[bundle splitting](./bundle-splitting.md).

## Never diagnose performance from the dev server

A dev server and a production build are different programs. Vite serves CSS as JavaScript
modules in development, so a page can flash unstyled for half a second there and never
flash in production, where the same CSS is a render-blocking `<link>` in the `<head>`.

The reverse trap is worse: a dev server feels fast because everything is local, uncompiled
and uncompressed, and hides the cost a visitor pays.

So the rule is simple and absolute: **build, serve the build, measure the build.** If the
symptom only reproduces against the dev server, the bug is in the dev tooling, and fixing
production will not make it go away. [Local preview](./local-preview.md) covers how to get
a production-equivalent build running locally and why the two differ.

## Use Lighthouse, not a hand-rolled script

It is tempting to collect `PerformanceObserver` entries yourself — it is twenty lines and
gives exactly the fields you want. Resist it for anything you intend to track, for three
reasons:

- **A single run has no error bar.** Load metrics swing easily by ±15% between runs.
  Lighthouse runs the page under a controlled profile and is the thing other tools agree
  with; a one-shot script reports noise with the same confidence as signal.
- **The audits are the actual product.** Raw metrics tell you the page is slow. Lighthouse
  tells you *"render-blocking requests — est. savings 320 ms"* and names the files. The
  diagnosis is what you were trying to buy.
- **Its numbers are comparable to the outside world.** A score has shared meaning;
  your script's milliseconds do not.

Keep a custom script only for assertions Lighthouse does not make — "the first paint uses
the theme the visitor chose", "both locales render" — and label it a correctness check, not
a performance instrument.

## Always read both presets

Lighthouse's **mobile** preset (Slow 4G, 4× CPU) is the default and is the honest number.
The **desktop** preset has no throttling and mostly describes your build and your server.

The gap between them is itself information: a page scoring 87 desktop and 57 mobile is not
"mostly fine", it is a page whose cost is in bytes and main-thread work, which only shows
up when either is scarce.

## Localhost understates the network and overstates the bytes

Two corrections to apply before concluding anything from a local run:

**A local preview usually sends no compression.** Check it rather than assume:

```bash
curl -s -o /dev/null -H "Accept-Encoding: br, gzip" \
  -w "%{size_download} %header{content-encoding}\n" http://localhost:4173/assets/<bundle>.js
```

If `content-encoding` comes back empty, every byte count in that run is the decompressed
size. A JavaScript bundle typically compresses 3–4× (one measured case: 943 KB raw,
293 KB with `gzip -9`), and a CDN serves brotli, which does better. A local measurement can
therefore report a download cost three times what a visitor pays.

**But compression does not excuse bundle size.** JavaScript is parsed and executed from its
decompressed form. Compression changes how long the bytes take to arrive, not how long they
take to run — so blocking time and interactivity stay expensive no matter how well the file
gzips. Treat the compressed size as the network cost and the raw size as the CPU cost; they
are two different problems that happen to share a file.

One measured case of the whole correction, same commit, same instrument: mobile 57 against
the uncompressed localhost preview, **71** against the CDN serving brotli over a real edge —
fourteen points that belong to the serving conditions, not to any code change.

**What a local run can never tell you:** edge TTFB, cache behaviour, HTTP/2 or /3
multiplexing, TLS cost, and real-user field data. Those need a deployed URL. Deploying a
known-imperfect page to a staging URL purely to measure it is a reasonable thing to do —
the lab number and the field number answer different questions, and you need both.

## FCP ≈ LCP means the page is gated on its stylesheets

A useful signature to recognize. When First Contentful Paint and Largest Contentful Paint
land within a couple of hundred milliseconds of each other, nothing painted *progressively*:
the browser waited, then drew the finished page in one go. That points at render-blocking
resources in the `<head>`, not at a heavy hero element.

When LCP trails FCP by a lot, the opposite — something painted early and a large element
arrived late. Those are different bugs with different fixes, and the two numbers side by
side tell you which one you have without opening a waterfall.

## The first paint has to agree with the theme

If a page supports light and dark and resolves the choice server-side, the inline critical
CSS and the `<html>` class must follow that choice too. A hardcoded dark background on a
page that is about to render light is not a flicker — it is the wrong theme for however long
the stylesheets take, which the measurements above routinely put in whole seconds.

## Validate the instrument before trusting a number that surprises you

A measurement that moves the wrong way is usually the measurement, not the page. Two real
failure modes of hand-written audit scripts:

- a colour parser that handles `rgb(0-255)` but not `color(srgb 0-1)`, reading every light
  surface as black;
- a contrast check that reads an element's own translucent background instead of compositing
  it over what is behind it.

Both produce confident, specific, wrong numbers — and a change made to satisfy a wrong
number makes the page worse. When a result contradicts the direction of the change you just
made — you darkened the text and the contrast ratio *fell* — stop and audit the instrument
before touching the page again. A check that fails falsely is as expensive as a check that
cannot fail: one hides a real defect, the other invents one and charges you a change to
"fix" it.

## Confirm which page the number describes

An unauthenticated measurer follows redirects and scores whatever it lands on, without
complaint. Point PageSpeed Insights or Lighthouse at a URL behind an auth wall — a
Cloudflare Access login, an SSO gate — and it returns confident numbers *for the login
page*. Measured once: a PSI run against a protected preview URL reported the access
provider's domain as the analyzed page, and nothing in the score table said so out loud.

Two checks before believing any number taken against a URL you do not fully control:

- the report's final/displayed URL must match the URL you asked for — a redirect to
  another host means the wrong page was scored;
- `curl -I` the URL first: a `3xx` to an auth provider means every unauthenticated tool
  will measure the gate, and the fix is a measurement window with the gate off (or a
  service token), not a different tool. Which preview mechanism to measure against in the
  first place is [deploy environments](../infra/deploy-environments.md)' question.

The same discipline applies to category scores on preview infrastructure: a platform that
stamps `x-robots-tag: noindex` on preview URLs (Cloudflare does, by design) tanks the SEO
category through its crawlability audit. Before treating a category drop as a regression,
open the category and read **which audit** failed — an environment-caused failure is a
property of the URL, not of the page.

## Record a baseline before optimizing, and pin the method to it

The value of a performance number is almost entirely in the one before it. So the first
measurement is worth taking deliberately:

- take it on a build, with Lighthouse, on both presets;
- write down the **commit**, the instrument **version**, and the serving conditions
  (compressed or not, local or deployed);
- keep the rows append-only — a corrected number is a new row with a note, never an edit,
  because the shape of the curve is the artifact;
- when the instrument changes, say so in the row. A series measured two ways is two series.

A baseline taken with the instrument you are about to abandon is worse than no baseline:
later improvements will mix real progress with the change of method, and you will not be
able to separate them.
