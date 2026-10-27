# 🎭 Dublio — Game Localization, Dubbing & Community Platform

**Browser-Based Studio · BYOS Architecture · Zero-Copy VFS · Native Engine Injection · Any Language**

Dublio is an end-to-end game localization platform that brings translation, dubbing, fonts, textures, mixing, quality control, and in-game deployment under a single roof. It replaces the chaotic industry standard of Discord ZIP files, spreadsheets, and manual file replacements with a structured, team-based pipeline and one-click installation for players. It is scalable from independent community projects to studio-level requirements.

> **Note:** This repository is an architectural showcase. Core source code remains private to protect proprietary AI pipelines and enterprise (B2B) security boundaries.

---

## At a Glance

| | |
|---|---|
| **Scope** | 80+ relational models, 10 production workshops, 3 permission layers |
| **Scale** | Source lists of 70,000+ lines, translation grids of 100,000+ lines, tens of thousands of audio and texture files per game |
| **Video OCR** | 6-hour gameplay recording processed in about 1 hour |
| **Storage** | 10 GB of raw assets reduced to 2.5 GB (75%) with no audible quality loss |
| **Compute cost** | $0 server-side transcoding, near-zero egress, AI workloads on user-controlled GPUs |
| **Status** | Solo-built over roughly one year, in active use |

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
| **AI Data Prep** | Video OCR, audio classification, speech transcription with automatic line matching, and stem separation, run on the team's own GPU |
| **Curation Studio** | Links lines to characters, audio, and video on a node canvas that models branching dialogue as a graph |
| **Translation Studio** | Real-time multiplayer grid with video/audio context, shared-line sync, Master Catalog auto-fill, and weighted community voting |
| **Font Studio** | Injects missing glyphs into the game's original font and exports Unity TextMeshPro atlases |
| **Art & Texture Studio** | Layer-based texture localization with A/B comparison, versioning, and a Bink video pipeline |
| **Dubbing Studio** | In-browser lossless recording synced to the scene, ADR timeline, unlimited takes |
| **Mixing Studio** | Multi-channel browser DAW with per-channel effects chain and in-browser master render |
| **QC Studio** | Interactive game player with branch testing, timestamped issue reports, and line locking |
| **Modding & Packaging** | Configurable "Repack Recipe" per project, engine-specific packers, incremental hotfix layers |

The full feature catalog lives in **[docs/FEATURES.md](docs/FEATURES.md)**.

---

## Architecture Overview

Dublio operates across four layers: a browser-based studio, an edge delivery network, a lightweight backend orchestrator, and a native desktop patcher.

```
[Game Engine] <══(Byte-Range Stream)══> [Cloudflare Worker] <══(Direct Read)══> [Backblaze B2]
                                               ^
                                          (Presigned URLs)
                                               |
[Dublio Desktop App] <══(Manifest & Jobs)══> [Next.js / Node.js Backend] <══(Queue)══> [Colab / Drive (BYOS)]
(Rust / Tauri)                                (PostgreSQL)                               (AI Processing)
```

### Backend Orchestrator

A Node.js + PostgreSQL (Prisma) stack manages project data, localization assets, narrative branching, job queues, and package versions. More than eighty relational models keep each game's text, audio, characters, and scene connections tightly linked. Long-running AI and file-processing jobs run through a distributed queue with explicit states (pending, processing, completed, failed), so they can run unattended and failures stay visible.

### Master Catalog & Team Projects

Every game is first indexed into a global, read-only **Master Catalog**. Each team spins up its own workspace from the catalog, so multiple teams can localize the same game with their own translation choices without overwriting each other. Previously approved translations can be reused as shared memory. Dialogue is stored as a **node-based graph**, not flat rows, so choices, branches, and scene transitions are modeled natively.

### BYOS — Bring Your Own Storage

Instead of centralizing expensive compute and storage, Dublio distributes the workload to user-controlled resources: Google Drive, Google Colab, and local CPU/GPU. Infrastructure scales horizontally at near-zero cost, which is what makes a free community tier possible.

- Automatic Drive workspace setup per project
- Direct-to-Drive uploads through a serverless bridge, bypassing Dublio's servers
- On-demand streaming of multi-gigabyte media from Drive
- Self-healing links when files are moved or replaced
- Guest voice actors can only access the audio of lines assigned to them

---

## Engineering Challenges & Solutions

### 1. Context-Aware Video OCR ("Prometheus Engine")

Translators need visual context; raw audio alone leads to blind, error-prone translations.

**Challenge:** Frame-by-frame processing of a 6-hour gameplay recording is computationally prohibitive.

**Solution:**
- GPU-accelerated ROI cropping via `ffmpeg -hwaccel cuda`, so OCR runs only on the region of interest.
- Adaptive frame skipping with a temporal similarity loop (`thefuzz`). If consecutive frames exceed 60% text similarity, the engine triggers an `ADAPTIVE_SKIP`, drastically reducing PaddleOCR inferences.
- Spatial clustering of bounding boxes to separate multi-line subtitles from disconnected UI elements.
- Separate subtitle and speaker-name regions, with repeated frames merged into single timecoded lines.

**Result:** A 6-hour video processed in approximately 1 hour.

---

### 2. Client-Side WASM Transcoding

Audio mixing requires specific formats (48 kHz, mono, 16-bit PCM). Server-side transcoding at scale would be a significant recurring cost.

**Solution:** FFmpeg WebAssembly runs entirely in the browser, so server compute cost is $0.

- The WASM instance is terminated and re-initialized every 5 files to prevent memory leaks during bulk operations.
- The FileSystem Access API builds a hierarchical `/Workspace` folder structure directly on the user's disk with no server round-trip.
- The same approach powers the in-browser master render in the Mixing Studio.

---

### 3. Smart Batch Uploader & Conflict Resolution

When a voice actor uploads 500 recorded `.wav` files at once, manual assignment is not feasible.

**Solution:** A 4-tier matching waterfall:
1. **Strict ID Match:** explicit tags (e.g., `_L88210.wav`)
2. **Sequence Match:** maps `seq-05` prefixes to project scenes
3. **Contextual Match:** analyzes cinematic video names linked to the project
4. **Fuzzy Fallback:** breaks the filename down and compares it against character names and dialogue keys

Collisions are never auto-resolved. They go to a manual review queue to prevent silent errors. The speech-to-text pipeline applies the same philosophy: high-confidence matches are accepted automatically, medium ones go to review, and low ones go to a manual queue.

---

### 4. Zero-Copy VFS & Byte-Offset Streaming

**Challenge:** Storing and serving tens of thousands of individual audio and texture files creates high storage overhead and egress costs.

**Solution:** Files are packed with `ZIP_STORED` (uncompressed). The backend calculates exact `start` and `end` byte offsets per asset. A **Cloudflare Worker** intercepts game engine requests, performs HTTP Range reads directly against a large archive in **Backblaze B2**, injects the correct `Content-Type` header, and serves the byte chunk. Because traffic is routed through Cloudflare, egress from B2 is free, which makes player downloads cost nothing.

The game engine receives what looks like a standalone file, unaware it is streaming from inside a multi-gigabyte archive.

**Result on a real-world game:** 10 GB of raw assets reduced to 2.5 GB (75%) with no audible quality loss, using `.opus` conversion.

---

### 5. Local-First Studio for 100,000+ Line Projects

**Challenge:** Browser UIs choke on huge line lists, especially on weak connections.

**Solution:**
- Each project gets an isolated IndexedDB workspace (Dexie.js), so large lists load from local storage.
- Virtualized rendering (TanStack Virtual) and Web Workers for heavy work such as transcript matching and similar-sentence search.
- An integrity checker periodically diffs local and server data and repairs only missing or damaged items.
- Real-time collaboration (Pusher Channels): live presence, typing indicators, and visible drafts prevent two people from translating the same line.

---

### 6. Incremental Hotfix Delivery

**Challenge:** Rebuilding and redistributing a 10 GB package because three audio files changed is wasteful for everyone.

**Solution:** Only changed assets are converted and published as a small hotfix layer. The desktop client applies it on top of the base package, so players download only what changed. Build phases are locked during packaging so a package never contains half-finished edits.

---

## Native Engine-Level Patcher

Unlike traditional modding tools that require manual file replacement, Dublio includes a custom-built patcher for **Unity (Mono and IL2CPP)**, with packaging tools for other engines.

- Reads `.assets` and `.bundle` files directly, locating the exact Byte/PathID of target audio, texture, or font assets.
- For textures above 16 MB, a strict `TEXTURE_STREAMING_THRESHOLD` bypasses RAM accumulation by streaming raw pixel data (RGBA32 flipped) directly into `.resource` blocks.
- Automatically detects IL2CPP builds and dynamically reconstructs DLL headers.
- Custom C# routines parse TextMeshPro font metadata and replace `m_FaceInfo` glyph structures without breaking UI layout.
- Purpose-built, signed packers for **Unreal Engine 5.6.1+** (`.pak`, `.utoc`, `.ucas`), **GameMaker**, **FMOD** sound banks, and **Telltale** archives.

---

## Desktop Client (Rust / Tauri)

A state-aware differential patcher built in Rust.

- **One-click install:** the website's "Install Patch" button opens the client through the `dublio://install` protocol, which detects the installed game and starts installation.
- **Install options per language:** full localization, dubbing only, or subtitles only.

**3-tier installation matrix:**
1. **Base Layer:** foundational asset mappings
2. **Version Patches (Differential):** only the delta between current and target version
3. **Hotfixes (Global Overwrite):** critical late-stage overrides (e.g., last-minute retakes)

Before applying any update, the client cross-references the previous `manifest.json`, hunts down orphaned files, and restores original game files from `.dublio.bak` backups, so the game directory is never left corrupted and players can always return to the original game.

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

### Discovery Portal

The homepage is designed as a gaming portal rather than a standard SaaS dashboard. Projects are assigned RPG-style rarities based on translation volume, and active teams and their members are mapped on a visual network.

https://github.com/user-attachments/assets/c40de33f-916c-429c-8a2c-08fc62b5a82f

---

## Security & B2B Boundary

**The Double Pool Model:**

- **Pool A (Community):** a free environment where independent teams use their own BYOS storage and Colab resources. It builds the talent network and validates the technology at scale.
- **Pool B (Enterprise):** a commercial pipeline for game studios requiring NDA-protected asset handling. Enterprise assets never touch user-controlled storage; they are processed in isolated, secure storage with short-lived presigned URLs.

Strict separation between the two pools is a core architectural constraint, not an afterthought. Source code stays private to protect this boundary and the proprietary AI pipelines.

Additionally:
- Users' Drive access tokens are stored encrypted.
- Fine-grained permissions at platform, team, and workshop level, with all critical actions written to audit logs.
- Short-lived presigned URLs keep file traffic off Dublio's servers.
- Nightly database backups into versioned cloud storage, with one-click restore.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | React, Next.js (App Router), TypeScript, Tailwind CSS, ReactFlow, Zustand, TanStack Virtual, Konva, Three.js |
| Backend | Node.js, PostgreSQL, Prisma ORM |
| Local Workspace | Dexie.js (IndexedDB), Web Workers |
| Desktop Client | Rust, Tauri |
| Media Processing | FFmpeg (WASM + CUDA), PaddleOCR, YAMNet, Whisper, Demucs, Web Audio API, Wavesurfer.js, opentype.js |
| Storage & Delivery | Backblaze B2, Cloudflare Workers, Google Drive, Google Apps Script |
| Real-Time | Pusher Channels, WebRTC |
| AI Processing | Google Colab (CUDA) |
| Engine Patching | Unity (.assets / .bundle, IL2CPP, C#), Unreal Engine, GameMaker, FMOD, Telltale |

---

## Current Status

Dublio is a solo-built project, architected and developed over roughly one year. It is in active use, progressing from a community-scale proof of concept toward industry-scale beta testing.

---

## Contact

I am currently open to roles in **Backend Engineering, Cloud Architecture, and DevOps.** If you are looking for an engineer who can design and build complex distributed systems from zero to production while maintaining strict cost-efficiency, I would be happy to connect.

- [LinkedIn](https://www.linkedin.com/in/emre-bulut-0a1827315/?profileFormEntryPoint=IntroForm)
- [Email](mailto:005emreebulutt005@gmail.com)

---

*Architected and developed by Chimiya.*
