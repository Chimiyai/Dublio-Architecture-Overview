# 🎭 Dublio — Game Localization, Dubbing & Community Platform

**Browser-Based Studio · BYOS Architecture · Byte-Offset Asset Vaults · Signed Thin-Client Patching · Any Language**

Dublio is an end-to-end game localization platform that brings translation, dubbing, fonts, textures, mixing, quality control, and in-game deployment under a single roof. It replaces the chaotic industry standard of Discord ZIP files, spreadsheets, and manual file replacements with a structured, team-based pipeline and one-click installation for players. It scales from independent community projects to studio-level requirements.

> **Note:** This repository is an architectural showcase. Core source code remains private to protect proprietary pipelines and enterprise (B2B) security boundaries.

---

## At a Glance

| | |
|---|---|
| **Scope** | 80+ relational models, 10 production workshops, 3 permission layers |
| **Live usage** | 3,500+ registered users · 1 completed and 4 ongoing projects · Poppy Playtime Chapter 5 (Unreal Engine 5.6) localization downloaded 2,000+ times |
| **Scale** | Source lists of 70,000+ lines, translation grids of 100,000+ lines, tens of thousands of audio and texture files per game |
| **Video OCR** | 6-hour gameplay recording processed in about 1 hour |
| **Storage** | 10 GB of extracted game data reduced to 2.5 GB (75%) by filtering and compression |
| **Infrastructure cost** | Whole platform on a single VDS, roughly 200 TL/month including domain; no server-side transcoding; storage cost is negligible at current scale |
| **Patcher reliability** | 20/20 simulated-crash recoveries with 0 corrupted bytes on a real GameMaker data file; 19 Rust tests passing |
| **Status** | Solo-built since April 2025, in active use and development |

---

## The Problem

Localizing a game is fundamentally different from translating traditional software. A project must keep branching narratives, scene structures, character personalities, thousands of scattered audio files, textures, and font atlases consistent with one another. Most communities still work with:

- **Disconnected workflows:** translators in spreadsheets without context, voice actors tracking timing in standalone editors, mixers pairing every line to its scene by hand.
- **Heavy infrastructure costs:** hundreds of gigabytes of video and tens of thousands of audio files are a serious burden for independent and non-profit teams.
- **No oversight or QA:** without a shared multiplayer environment, version conflicts, lost lines, and sync errors are routine.

The goal: **let creators focus purely on localization work while the infrastructure handles asset organization, processing, and delivery to players.**

---

## Platform Overview

Dublio supports localization into **any target language**. Teams can localize the same game into different languages side by side, each in an isolated workspace. The pipeline is made of ten workshops:

| Workshop | What it does |
|---|---|
| **Source Parsing** | Auto-detects and parses Ren'Py, Unity I2, Unreal `.locres`, Gettext, Construct, INI, SRT/VTT, and plain text into one uniform structure |
| **AI Data Prep** | Video OCR, audio classification, speech transcription with automatic line matching, and stem separation on GPU |
| **Curation Studio** | Links lines to characters, audio, and video on a node canvas that models branching dialogue as a graph |
| **Translation Studio** | Real-time multiplayer grid with video/audio context, shared-line sync, Master Catalog auto-fill, and weighted community voting |
| **Font Studio** | Injects missing glyphs into the game's original font and exports Unity TextMeshPro atlases |
| **Art & Texture Studio** | Layer-based texture localization with A/B comparison, per-artist version history, rollback, and sabotage reporting |
| **Dubbing Studio** | In-browser lossless recording synced to the scene, ADR timeline, unlimited takes |
| **Mixing Studio** | Multi-channel browser DAW with per-channel effects chain and in-browser master render |
| **QC Studio** | Interactive game player with branch testing, timestamped issue reports, and line locking |
| **Modding & Packaging** | Per-project packaging recipe, engine-specific packers, layered updates and hotfixes |

The full feature catalog lives in **[docs/FEATURES.md](docs/FEATURES.md)**.

---

## Architecture Overview

Dublio operates across four parts: a browser-based studio, an edge delivery path, a backend orchestrator, and a native desktop client.

```
[Dublio Desktop Hub] <══ signed recipes / manifests ══> [Next.js / Node.js Backend on VDS] <══> [PostgreSQL]
 (Rust / Tauri)                  |                                  |
        |                  (engine parsing,                  (presigned URLs)
        |                   recipe signing)                         |
        └══════ byte-range / vault download ══> [Cloudflare] <══> [Backblaze B2]
```

### Backend

A Node.js + PostgreSQL (Prisma) stack manages project data, localization assets, narrative branching, package versions, and permissions. More than eighty relational models keep each game's text, audio, characters, and scene connections tightly linked. Heavy media work is pushed away from the server: transcoding runs in the user's browser (FFmpeg WASM), and AI workloads run on GPU resources outside the platform's own server.

### Master Catalog & Team Projects

Every game is first indexed into a global, read-only **Master Catalog**. Each team spins up its own workspace from the catalog, so multiple teams can localize the same game with their own translation choices without overwriting each other. Previously approved translations are reused as shared memory. Dialogue is stored as a **node-based graph**, not flat rows, so choices, branches, and scene transitions are modeled natively.

### BYOS — Bring Your Own Storage

Instead of centralizing expensive compute and storage, Dublio distributes workloads to user-controlled resources: Google Drive, Google Colab, and local CPU/GPU. Infrastructure scales horizontally at near-zero cost, which is what makes a free community tier possible.

- Automatic Drive workspace setup per project
- Direct-to-Drive uploads through a serverless bridge, bypassing Dublio's servers
- On-demand streaming of multi-gigabyte media from Drive
- Self-healing links when files are moved or replaced
- Guest voice actors can only access the audio of lines assigned to them

### Real-Time Collaboration

- **Pusher Channels** for presence, notifications, messaging, and metadata sync across the platform.
- **Yjs CRDT over WebRTC** (with a self-hosted signaling server) for conflict-free, peer-to-peer text editing in shared notes and the curation canvas, with live remote cursors.
- Text streams travel peer-to-peer; the database receives debounced saves, and Pusher only carries metadata, so remote updates never overwrite what someone is typing.

---

## Engineering Challenges & Solutions

### 1. Ingestion Pipeline: Filter First, Compress Second

**Challenge:** Raw data extracted from a game is mostly irrelevant to localization, yet tens of thousands of files must be stored and served.

**Solution:**
- Extracted files that need no localization (no text, no speech, no readable imagery) are discarded at ingestion. They never occupy storage.
- File metadata is captured when an asset first enters the system, **before** any conversion. Metadata differs from game to game and file to file, so nothing is assumed.
- Audio is compressed aggressively with FFmpeg (lossy), with the target format driven by each file's captured metadata. Text, images, video, and other assets are compressed losslessly.
- At packaging time, the stored metadata is applied to the users' localized files, so the engine-side properties of each replacement match the original.

**Result on a real-world game:** 10 GB of extracted data reduced to 2.5 GB (75%). Most of the saving comes from discarding non-localizable files; the rest comes from compression.

---

### 2. Byte-Offset Vaults with Adaptive Retrieval

**Challenge:** Storing and serving tens of thousands of individual audio and texture files creates high object-count overhead, request latency, and egress costs.

**Solution:**
- Assets are packed into uncompressed ZIP vaults (`ZIP_STORED`). The ZIP is only a container, so every asset sits at an exact, known byte range.
- The database stores `startByte` / `endByte` for each asset (and separate offsets for mixed audio).
- **Two-tier retrieval:** when a build needs **more than 100** files from a vault, the client downloads the vault once and slices it locally with `seek`. When it needs fewer, it fetches each asset individually by HTTP byte-range, without downloading the whole archive.
- Temporary vault downloads are deleted as soon as the build finishes, leaving no persistent disk footprint.
- Storage is Backblaze B2, served through Cloudflare, so egress from B2 is free.

This architecture replaced an earlier design that used Google Apps Script workers to zip and unzip files. That version was measurably slower, and byte-offset slicing made it obsolete.

---

### 3. Context-Aware Video OCR ("Prometheus Engine")

Translators need visual context; raw audio alone leads to blind, error-prone translations.

**Challenge:** Frame-by-frame processing of a 6-hour gameplay recording is computationally prohibitive.

**Solution:**
- GPU-accelerated ROI cropping via `ffmpeg -hwaccel cuda`, so OCR runs only on the region of interest.
- Adaptive frame skipping with a temporal similarity loop (`thefuzz`). If consecutive frames exceed 60% text similarity, the engine triggers an `ADAPTIVE_SKIP`, drastically reducing PaddleOCR inferences.
- Spatial clustering of bounding boxes to separate multi-line subtitles from disconnected UI elements.
- Separate subtitle and speaker-name regions, with repeated frames merged into single timecoded lines.

**Result:** A 6-hour video processed in approximately 1 hour.

---

### 4. Client-Side WASM Transcoding

Audio mixing requires specific formats (48 kHz, mono, 16-bit PCM). Server-side transcoding at scale would be a significant recurring cost.

**Solution:** FFmpeg WebAssembly runs entirely in the browser, so server-side transcoding cost is $0.

- The WASM instance is terminated and re-initialized every 5 files to prevent memory leaks during bulk operations.
- The FileSystem Access API builds a hierarchical `/Workspace` folder structure directly on the user's disk with no server round-trip.
- The same approach powers the in-browser master render in the Mixing Studio.

---

### 5. Smart Batch Uploader & Conflict Resolution

When a voice actor uploads 500 recorded `.wav` files at once, manual assignment is not feasible.

**Solution:** A 4-tier matching waterfall:
1. **Strict ID Match:** explicit tags (e.g., `_L88210.wav`)
2. **Sequence Match:** maps `seq-05` prefixes to project scenes
3. **Contextual Match:** analyzes cinematic video names linked to the project
4. **Fuzzy Fallback:** breaks the filename down and compares it against character names and dialogue keys

Collisions are never auto-resolved. They go to a manual review queue to prevent silent errors. The speech-to-text pipeline applies the same philosophy: high-confidence matches are accepted automatically, medium ones go to review, and low ones go to a manual queue.

---

### 6. Local-First Studio for 100,000+ Line Projects

**Challenge:** Browser UIs choke on huge line lists, especially on weak connections.

**Solution:**
- Each project gets an isolated IndexedDB workspace (Dexie.js), so large lists load from local storage.
- Virtualized rendering (TanStack Virtual) and Web Workers for heavy work such as transcript matching and similar-sentence search.
- An integrity checker periodically diffs local and server data and repairs only missing or damaged items.
- Live presence, typing indicators, and visible drafts prevent two people from translating the same line.

---

### 7. Data Integrity Rules in the Backend

- **Reference-counted deletion:** a file is physically deleted only after a scan of all five tables that can point to it (line audio, raw recordings, hotfixes, package versions, package files) finds no remaining reference. Database changes run in a transaction.
- **Field-level dirty tracking:** text/texture changes and dubbing changes use separate `isDirty` flags, so uploading a voice take never triggers a text hotfix. A voice actor's new take produces a small, actor-specific voice pack instead.
- **Mix immutability:** in team projects, a finished studio mix is never overwritten or cleared by an actor's later takes. Sequences of lines are treated as a single audio unit and synchronized in one transaction.
- **Notification discipline:** active users get in-app notifications only; email (via Resend) goes only to users who have been away for more than 10 days, with per-user preferences and a silent mode for admin actions.

---

## Layered Delivery: Base, Patches, Hotfixes, Voice Packs

Updates are layered so players download only what changed. The desktop client installs up to four layers in order:

1. **Base (multi-part):** the foundation of a localization, split into 1.5–2 GB parts for large games.
2. **Version patches:** chained differential updates between the installed version and the target.
3. **Hotfixes:** small delta packages (tens of KB up to about 25 MB) carrying text, font, and global fixes, plus tombstones for deleted files. When micro-hotfixes accumulate, they are merged into a consolidated block; a new Base release clears older hotfixes.
4. **Actor-based voice packs:** the player picks a cast, and only the chosen voice actor's audio per character is installed. Updates only fetch voice packs created since the last install, except for characters whose actor changed, which get the full pack.

Each layer is applied in two passes (tombstones first, then files), so a deleted-and-replaced file never conflicts. Before applying an update, the client cross-references the previous manifest, removes orphaned files, and restores originals from `.dublio.bak` backups, so players can always return to the unmodified game.

---

## Signed Thin-Client Patcher

Dublio's desktop client is a Rust / Tauri application. For modern packages it is deliberately a **"dumb byte applicator"**.

- **Engine logic stays on the server.** Parsing engine-specific formats (GameMaker, Unity, and others) runs on the Linux VDS. The client never downloads engine-specific parsers; it receives a declarative recipe of atomic byte operations (`APPEND_PAYLOAD`, `WRITE_BYTES`, `PATCH_POINTERS`).
- **Signed recipes.** Recipes are Ed25519-signed. The client verifies the signature over the raw bytes of the payload before parsing anything, and rejects unknown or revoked keys. Online recipes carry a client-generated one-time nonce to prevent replay.
- **Windows path jail.** Rejects directory traversal, NTFS alternate data streams, reserved device names, and 8.3 short names/symlinks (resolved from an open handle). Files are opened with exclusive sharing, and only an allowlist of data extensions can be patched. Executables are never targets.
- **Three-tier integrity verification.** (1) file size and header SHA-256 before touching anything; (2) SHA-256 of every byte range about to be overwritten; (3) final size and header hash after patching. A mismatch aborts before a single byte is written.
- **Crash-safe journal.** A state machine (`PREPARED → PAYLOADS_APPENDED → POINTERS_PATCHED → COMMITTED`) is persisted atomically. After a crash or power loss, the next launch rolls the files back to their original bytes. If the game was updated by Steam in the meantime, the journal is kept and the state is flagged as stale instead of overwriting the new files.

### Engine support

| Engine | Strategy |
|---|---|
| **GameMaker** | Append-and-repoint: new audio appended to the data file, chunk sizes and pointers updated via recipe |
| **Unity** | Audio appended to stream resources, serialized object pointers updated via recipe (validated on a multi-file pilot) |
| **Unreal Engine 5.x** | Pre-baked containers (`.pak` / `.utoc` / `.ucas`) are built in the packaging stage and copied into the mods folder, never touching original files. Used for Poppy Playtime Chapter 5 (UE 5.6) |
| **FMOD** | Pre-built sound banks distributed after validation |

A legacy C# packer path remains as a fallback for older packages without recipes.

### Measured reliability

- **Crash recovery:** on a real GameMaker `data.win` (about 2.9 MB), 20 simulated crashes across 6 failure points were recovered 20/20 byte-for-byte, with 0 corrupted bytes and an average recovery time of about 15 ms.
- **Unity multi-file transaction:** a `resources.assets` + `.resS` pair was locked, patched, and rolled back correctly.
- **Test suite:** 19 Rust recipe tests passing, TypeScript type-check with 0 errors, plus end-to-end scripts for the GameMaker and Unity flows.

Moving engine parsing off the client does not make reverse engineering impossible, but it keeps proprietary parsers off players' machines and avoids shipping executables entirely.

---

## Desktop Client

- **One-click install:** the website's "Install Patch" button opens the client through the `dublio://install` protocol, which detects the installed game and starts installation.
- **Install options per language:** full localization, dubbing only, or subtitles only.
- **Update detection:** the client shows whether the installed mod is current, and applies only the missing patches.
- **Mixing and packaging tools:** the same client hosts the desktop mixing studio and the packaging ("kitchen") pipeline.

---

## Interface

### Curation Canvas

Instead of spreadsheets, users build the localization narrative visually in an infinite node editor (`ReactFlow`). Audio blocks, text nodes, choices, and video timings are connected via edges.

- **Wizard Mode:** keyboard-driven, high-throughput batch linking
- **Canvas Mode:** spatial node editor with real-time sync indicators
- **Demo Mode:** instant preview of the player experience before translation begins

https://github.com/user-attachments/assets/bfead1e2-17e6-412f-8826-5bfd4b00a15a

### Role-Specific Workspaces

Every contributor role gets a dedicated, focused environment:

- **Translators:** side-by-side video context, reference audio playback, community voting and reputation
- **Voice Actors:** in-browser lossless recording synced to the scene, or batch upload from an external DAW

<img width="720" height="354" alt="dubbing workshop" src="https://github.com/user-attachments/assets/cfe915e7-7a7d-4381-8f4a-6b54382c1595" />

### Discovery Portal & Community

The homepage is designed as a gaming portal rather than a standard SaaS dashboard. Projects are assigned RPG-style rarities based on translation volume, and active teams and their members are mapped on a visual network. Contributors earn XP and levels; advanced workshops unlock at higher levels. A reputation record with admin audit trail tracks both commendations and violations, and community translation voting blocks self-voting.

https://github.com/user-attachments/assets/c40de33f-916c-429c-8a2c-08fc62b5a82f

---

## Security & B2B Boundary

**The Double Pool Model:**

- **Pool A (Community):** a free environment where independent teams use their own BYOS storage and Colab resources. It builds the talent network and validates the technology at scale.
- **Pool B (Enterprise, by design):** a commercial pipeline for game studios requiring NDA-protected asset handling. Enterprise assets are designed never to touch user-controlled storage; they are processed in isolated storage with short-lived presigned URLs.

Strict separation between the two pools is a core architectural constraint, not an afterthought.

Additionally:
- Users' Drive access tokens are stored encrypted.
- Role-based access control at platform, team, and workshop level through a single permission helper; critical actions and moderation decisions record who performed them.
- Workshops of projects awaiting review are frozen for everyone except platform administrators.
- User-generated HTML is sanitized before rendering.
- Short-lived presigned URLs keep file traffic off Dublio's servers.
- Nightly database backups into versioned cloud storage, with one-click restore.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | React, Next.js (App Router), TypeScript, Tailwind CSS, ReactFlow, Zustand, TanStack Virtual, Konva, Three.js, next-intl |
| Backend | Node.js, PostgreSQL, Prisma ORM, Resend |
| Local Workspace | Dexie.js (IndexedDB), Web Workers |
| Desktop Client | Rust, Tauri |
| Media Processing | FFmpeg (WASM + CUDA), PaddleOCR, YAMNet, Whisper, Demucs, Web Audio API, Wavesurfer.js, opentype.js |
| Storage & Delivery | Backblaze B2, Cloudflare, Google Drive, Google Apps Script (Drive bridge) |
| Real-Time | Pusher Channels, WebRTC, Yjs (CRDT) |
| AI Processing | Google Colab (CUDA) |
| Engine Patching | Ed25519-signed byte recipes, GameMaker, Unity, Unreal Engine, FMOD |
| Hosting | Single Linux VDS |

---

## Current Status

Dublio is a solo-built project, developed since April 2025. It is in active use by a small, growing community: 3,500+ registered users, one completed project (Poppy Playtime Chapter 5, 2,000+ downloads) and four in progress. It is progressing from a community-scale proof of concept toward industry-scale beta testing.

---

## Contact

I am currently open to roles in **Backend Engineering and Cloud Infrastructure.** If you are looking for an engineer who can design and build complex systems from zero to production while keeping costs under control, I would be happy to connect.

- [LinkedIn](https://www.linkedin.com/in/emre-bulut-0a1827315/)
- [GitHub](https://github.com/Chimiyai)
- [Email](mailto:ebulutdev@gmail.com)

---

*Architected and developed by Emre Bulut (Chimiya).*
