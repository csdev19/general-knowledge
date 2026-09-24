# App bundle identifiers

_How Niway apps pick their bundle identifier, why the prefix is `dev.niway.*`, and why an already-shipped identifier never changes._

## The convention

Bundle identifiers use reverse-DNS of a domain the org controls. Niway's domain
is **niway.dev**, so every new app is:

```text
dev.niway.<product>        e.g. dev.niway.rik-stay-awake
```

This applies to macOS/iOS bundle IDs, Android application IDs, and Electron
`appId` alike. Pick the identifier once, before the first public release, and
treat it as frozen from then on.

## Why reverse-DNS, and what it does not do

The convention exists to guarantee global uniqueness without a central
registry: if you own the domain, nobody else will use your prefix. Nothing
verifies it — Apple and Google never check domain ownership — so an identifier
under a domain you do not own still works; it just risks (theoretical)
collision and reads as sloppy.

Functionally, `com.niway.x` and `dev.niway.x` behave identically: signing,
notarization, sandboxing, preferences, and updaters only care that the string
is unique and stable.

## Why frozen after first release

The identifier IS the app's identity to the OS and the update system.
Changing it after shipping means:

- macOS/Android treat it as a different app: preferences, granted permissions,
  keychain items, and login items are lost.
- The updater (Sparkle, electron-builder, store) no longer recognizes existing
  installs — old users are stranded on the last old-ID version.
- Deep-link protocol registrations and single-instance locks break.

Before the first release — or while an app has no real users — a rename is a
one-line change plus a sweep for literal references (userData paths, protocol
registration, update feed). That window closes at launch.

## Users never see it

The identifier is metadata: it lives in `Info.plist`, container paths under
`~/Library/Containers/`, and `defaults` domains. Regular users see the app
name and the signing identity (the developer name in the Gatekeeper prompt),
never the bundle ID. Consistency here is for the org's own hygiene, not for
users.

## Recorded exceptions

- **Kaipu** shipped as `com.niway.kaipu-record` before this convention was
  written. It stays: even with few users, the migration buys nothing users can
  see, and grandfathering is cheaper than coordinating a rename across the
  update feed and existing installs. It is unique regardless of who owns
  niway.com.
- If Niway ever buys **niway.com**, existing identifiers still do not migrate;
  at most, new apps could adopt `com.niway.*` from that point on. Mixed
  prefixes across apps are harmless.

## Related

- Signing identity is separate from the identifier: Niway apps sign under the
  personal Apple Developer team `K9TKC5GG76` (Cristian Sotomayor); "Niway" is
  a naming/GitHub-org convention, not a legal entity Apple knows about.
