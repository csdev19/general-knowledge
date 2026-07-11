# Product Philosophy — Local-First Media App

*Core principles for a local-first desktop app — the artifact is the product, cloud as a capability, and the vault model.*

> Living document. Updated as decisions are made.

---

## Core Idea

Think of it as an "Obsidian for recordings/media artifacts."

The user's artifacts are theirs. They live on the machine first. The cloud exists as a capability — not the product.

---

## The Three Principles

### 1. The artifact is the product

Not the upload. Not the cloud. Not the pipeline.

An artifact exists the moment it's saved to disk. Everything after that — transcoding, uploading, sharing — is infrastructure. Infrastructure serves the artifact. The artifact does not serve the infrastructure.

### 2. Cloud is a capability. Not a category.

Successful infrastructure is invisible.

If an upload worked: show nothing.
If it failed: show something — because it requires user action.

Cloud sync status is not primary information. It belongs to the margins, not the card title.

### 3. The library is a collection of artifacts. Not a processing queue.

Users open the library to find their artifacts.
Not to monitor pipeline states.
Not to act as upload operators.

---

## Core User Flow

```
Create → Review → Share
```

Nothing else should dominate the experience.

---

## Navigation

```
● Create
□ Library
⚙ Settings
```

### Library filters

```
All  ·  Shared  ·  Failed
```

`Recent` is a default sort, not a filter.

---

## What the User Sees

### An artifact card

```
┌─────────────────────┐
│                     │
│      Preview        │
│                     │
└─────────────────────┘

Sprint Review

28 min • Yesterday
```

Nothing else. No badge. No status. No cloud language.

### A shared artifact

```
┌─────────────────────┐
│                     │
│      Preview     ↗  │
│                     │
└─────────────────────┘

Sprint Review

28 min • Yesterday
```

The ↗ icon means: has a public link. Nothing more.

### An artifact with an error

```
⚠ Upload failed — [Retry]
```

Errors are visible. Successful infrastructure is not.

---

## What the User Does NOT See

All of these badges are removed:

- `LOCAL` · `UPLOADED` · `PROCESSING` · `AUTO-UPLOAD PENDING` · `UPLOADING` · `READY` · `ARCHIVED`

These are pipeline states. They belong in logs, not in the UI.

---

## What Drives User Actions

| Situation             | What user sees                      | Action available      |
| --------------------- | ----------------------------------- | --------------------- |
| Artifact saved locally | Thumbnail · title · duration · date | Play · Share · Delete |
| Upload failed         | ⚠ small warning                     | Retry                 |
| Has cloud link        | ↗ icon on thumbnail                 | Copy link             |
| Everything working    | Nothing                             | —                     |

---

## Positioning

| App          | Philosophy                                          |
| ------------ | --------------------------------------------------- |
| **Loom**     | Cloud-first. The recording is a network event.      |
| **iMovie**   | File-first. Export when ready.                      |
| **Obsidian** | Files are yours. Sync is a plugin, not the product. |
| **This app** | The artifact is yours. Cloud is a capability.       |

---

## Internal Architecture (not user-facing)

### Three independent axes

```typescript
// Is the artifact file itself valid and usable?
type RecordingState = 'working' | 'ready' | 'error'

// What is the cloud sync status?
type SyncState = 'none' | 'pending' | 'syncing' | 'synced' | 'error'

// Is there a shareable link? (cloud-only, premium feature)
type ShareState = 'private' | 'shared'
```

**RecordingState:**

- `working` — file is being created (operation in progress, composition running)
- `ready` — file exists on disk, is playable/usable
- `error` — creation failed; file may be unusable. Always show to user.

**SyncState:**

- `none` — cloud sync never attempted
- `pending` — queued for upload; waiting for user trigger (manual) or connectivity
- `syncing` — actively uploading right now
- `synced` — confirmed in cloud
- `error` — upload failed; show ⚠ to user

**ShareState:**

- `private` — no public link
- `shared` — public link exists (`shareUrl` set). Sharing is cloud-only, premium.

---

### Storage fields — timestamps, not booleans

**Decision:** Use nullable ISO timestamps instead of boolean flags.

```typescript
localSavedAt: string | null   // ISO 8601, null = no local file on disk
cloudSyncedAt: string | null  // ISO 8601, null = never confirmed in cloud
```

**Why timestamps beat booleans:**

- More auditable: you know _when_ the local copy was saved and _when_ it synced
- The boolean is derivable: `hasLocal = localSavedAt !== null`
- Enables future features: sort by sync date, "synced X ago", etc.
- Same pattern as soft deletes (`deletedAt`), widely understood

**Electron-store note:** Store as ISO strings, not `Date` objects — JSON doesn't serialize `Date`.

---

### Restart recovery — always try to recover

**Decision:** On app restart, always attempt to recover artifacts in `working` state.

Recovery logic:

1. Find all artifacts with `recordingState: 'working'`
2. Check if the local file exists on disk
3. If file exists → mark as `ready` (the file survived the crash)
4. If file does not exist → mark as `error` (nothing to recover)

**Never** mark an artifact as `error` just because the app closed. Only mark `error` if the file is missing or unplayable.

---

### Vault model — one folder, one library

**Decision:** The library registry belongs to the active save folder. Changing the save folder changes the vault.

Inspired by Obsidian's vault model:

- Each save folder has its own artifact registry
- The registry lives in `.app/` inside the save folder
- Changing `savePath` opens a new vault — the old artifacts are still on disk but not shown
- **The user is explicitly warned** when changing save folder: "Changing this folder will show a different library. Your previous files remain at the old location."

**Why not a global registry:**

- Avoids orphaned entries pointing to deleted/moved files
- The registry and files are always colocated — copy the folder, you have everything
- No reconciliation needed when files are moved externally

---

### Sharing

- Sharing requires a cloud copy (the file must be synced first)
- Sharing is a premium feature (not available on free tier)
- `ShareState` is derived from whether `shareUrl` is set
- If `cloudSyncedAt` is null, sharing is not available — no UI affordance shown
