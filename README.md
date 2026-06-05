# 🎭 Dublio — Distributed Game Localization Platform

**Zero-Copy VFS · BYOS Architecture · Native Engine Injection**

Dublio is a full-stack localization platform built to replace the chaotic industry standard of Discord ZIP files and manual file replacements. It provides a structured, distributed pipeline for translation, dubbing, and engine-level asset deployment — scalable from indie community projects to AAA studio pipelines.

> **Note:** This repository is an architectural showcase. Core source code remains private to protect proprietary AI pipelines and enterprise (B2B) security boundaries.

---

## The Problem

Game localization workflows are surprisingly fragmented. Most teams still rely on manually shared ZIP files, Discord history threads, spreadsheets, and destructive file replacements inside game directories. As projects scale to tens of thousands of lines and hundreds of voice actors, keeping translations, audio, and assets synchronized becomes operationally unsustainable.

Dublio was built to eliminate that overhead. The goal: **let creators focus on localization work while the infrastructure handles asset organization, processing, and deployment.**

---

## Architecture Overview

Dublio operates across three distinct layers — an edge delivery network, a lightweight backend orchestrator, and a native desktop patcher.

```
[Game Engine] <══(Byte-Range Stream)══> [Cloudflare Worker] <══(Direct Read)══> [Backblaze B2]
                                               ^
                                          (Presigned URLs)
                                               |
[Dublio Desktop App] <══(Manifest & Jobs)══> [Next.js / Node.js Backend] <══(Queue)══> [Colab / Drive (BYOS)]
(Rust / Tauri)                                (PostgreSQL)                               (AI Processing)
```

### Backend Orchestrator

A lightweight Node.js + PostgreSQL stack manages project metadata, localization assets, narrative branching, job scheduling, and manifest/version tracking. Heavy AI processing is intentionally pushed outside centralized infrastructure to reduce cloud costs.

### BYOS — Bring Your Own Storage

Instead of centralizing expensive compute, Dublio distributes workloads to user-controlled resources: Google Drive, Google Colab, and local CPU/GPU. This makes the infrastructure horizontally scalable at near-zero egress cost.

---

## Engineering Challenges & Solutions

### 1. Context-Aware Video OCR ("Prometheus Engine")

Translators need visual context. Providing raw audio alone leads to blind, error-prone translations.

**Challenge:** Frame-by-frame processing of a 6-hour gameplay recording is computationally prohibitive.

**Solution:**
- GPU-accelerated ROI cropping via `ffmpeg -hwaccel cuda` — OCR runs only on the region of interest, not full frames.
- Adaptive frame skipping using a temporal similarity loop (`thefuzz`). If consecutive frames exceed 60% text similarity, the engine triggers an `ADAPTIVE_SKIP`, drastically reducing PaddleOCR inferences.
- Spatial clustering of raw bounding boxes to distinguish multi-line subtitles from disconnected UI elements.

**Result:** A 6-hour video processed in approximately 1 hour.

---

### 2. Client-Side WASM Transcoding

Audio mixing requires specific formats (48kHz, Mono, 16-bit PCM). Server-side transcoding at scale would incur significant infrastructure costs.

**Solution:** FFmpeg WebAssembly runs entirely in the browser. All transcoding happens on the user's CPU — server compute cost is $0.

**Implementation details:**
- WASM instance is programmatically terminated and re-initialized every 5 files to prevent memory leaks during bulk operations.
- FileSystem Access API generates a hierarchical `/Workspace` folder structure directly on the user's local disk without any server round-trip.

---

### 3. Smart Batch Uploader & Conflict Resolution

When a voice actor uploads 500 recorded `.wav` files at once, manual assignment is not feasible.

**Solution:** A 4-tier matching waterfall algorithm:
1. **Strict ID Match** — explicit tags (e.g., `_L88210.wav`)
2. **Sequence Match** — maps `seq-05` prefixes to project scenes
3. **Contextual Match** — analyzes cinematic video names linked to the project
4. **Fuzzy Fallback** — breaks down the filename and compares against character names and dialogue keys

Collisions are never auto-resolved. They are escalated to a manual review queue to prevent silent errors.

---

### 4. Zero-Copy VFS & Byte-Offset Streaming

**Challenge:** Storing and serving tens of thousands of individual audio/texture files creates high storage overhead and egress costs.

**Solution:** Files are packed using `ZIP_STORED` (uncompressed). The backend calculates exact `start` and `end` byte offsets for each asset. A Cloudflare Worker intercepts game engine requests, performs HTTP Range reads directly from a massive `.zip` archive in Backblaze B2, injects the correct `Content-Type` header, and serves the byte chunk.

The game engine receives what appears to be a standalone file — completely unaware it is streaming from inside a multi-gigabyte archive.

**Real-world result on Oxenfree:** 10GB of raw assets optimized to 2.5GB (75% reduction) with no audible quality loss, using `.opus` conversion for legal fair-use compliance.

---

## Native Engine-Level Patcher

Unlike traditional modding tools that require manual file replacement, Dublio includes a custom-built intelligent patcher — currently supporting 100% of Unity architectures, with Unreal Engine support planned.

**How it works:**
- Reads `.assets` and `.bundle` files directly, locating the exact Byte/PathID of target audio, texture, or font assets.
- For textures exceeding 16MB, implements a strict `TEXTURE_STREAMING_THRESHOLD`: bypasses RAM accumulation by streaming raw pixel data (RGBA32 flipped) directly into `.resource` blocks.
- Automatically detects IL2CPP architectures and dynamically reconstructs DLL headers.
- Custom C# routines parse TextMeshPro font metadata, replacing `m_FaceInfo` glyph structures without breaking UI layout.

---

## Desktop Client (Rust / Tauri)

The desktop application is a state-aware differential patcher built in Rust.

**3-tier installation matrix:**
1. **Base Layer** — foundational asset mappings
2. **Version Patches (Differential)** — only the delta between current and target version
3. **Hotfixes (Global Overwrite)** — critical late-stage overrides (e.g., last-minute retakes)

Before applying any update, the Rust engine cross-references the previous `manifest.json`, hunts down orphaned files, and restores original game files from `.dublio.bak` backups — ensuring the game directory is never left in a corrupted state.

---

## Interface

### Curation Canvas

The primary workflow interface. Instead of spreadsheets, users build the localization narrative visually using an infinite node editor (`ReactFlow`). Audio blocks, text nodes, and video timings are connected via edges.

- **Wizard Mode** — guided automated setup for rapid batch linking
- **Canvas Mode** — spatial node editor with real-time sync indicators and dirty-state tracking
- **Demo Mode** — instant preview environment before deployment

https://github.com/user-attachments/assets/bfead1e2-17e6-412f-8826-5bfd4b00a15a

### Role-Specific Workspaces

Every contributor role gets a dedicated, focused environment:

- **Translators** — side-by-side video context, reference audio playback, community reputation/badge system
- **Voice Actors** — in-browser recording synced to scene, or batch upload from external DAW

<img width="720" height="354" alt="dubbing workshop" src="https://github.com/user-attachments/assets/cfe915e7-7a7d-4381-8f4a-6b54382c1595" />

### Discovery Portal

The homepage is designed as a gaming portal rather than a standard SaaS dashboard. Projects are assigned RPG-style rarities based on translation volume, with a visual network (The Nexus) mapping active translation teams and their members.

https://github.com/user-attachments/assets/c40de33f-916c-429c-8a2c-08fc62b5a82f

---

## Security & B2B Boundary

**The Double Pool Model:**

- **Pool A (Community)** — free environment where indie teams use their own BYOS storage and Colab limits. Builds the talent network and validates the technology at scale.
- **Pool B (Enterprise)** — commercial pipeline for game studios requiring NDA-protected asset handling. Enterprise assets never touch user-controlled storage; they are processed in isolated, secure S3 buckets via robust `MultipartUpload` protocols with short-lived presigned URLs.

Strict compartmentalization between these two pools is a core architectural constraint, not an afterthought. The source code remains private to protect this boundary and the proprietary AI ingestion pipelines from reverse-engineering.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | React, Next.js, TypeScript, ReactFlow, dnd-kit |
| Backend | Node.js, PostgreSQL |
| Desktop Client | Rust, Tauri |
| Media Processing | FFmpeg (WASM + CUDA), PaddleOCR, Demucs, Whisper |
| Storage & Delivery | Backblaze B2, Cloudflare Workers, AWS S3 |
| Engine Patching | Unity (.assets / .bundle), IL2CPP, C# |
| BYOS | Google Drive, Google Colab |

---

## Current Status

Dublio is a solo-built project, architected and developed over roughly one year. It is currently in active deployment, transitioning from community proof-of-concept toward industry-scale beta testing.

---

## Contact

I am currently open to roles in **Backend Engineering, Cloud Architecture, and DevOps.** If you are looking for an engineer who can design and build complex distributed systems from zero to production while maintaining strict cost-efficiency, I would be happy to connect.

- [LinkedIn](https://www.linkedin.com/in/emre-bulut-0a1827315/?profileFormEntryPoint=IntroForm)
- [Email](mailto:005emreebulutt005@gmail.com)

---

*Architected and developed by Chimiya.*
