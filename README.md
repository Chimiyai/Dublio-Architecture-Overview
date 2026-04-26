
https://github.com/user-attachments/assets/d7b8352d-9f64-4661-b797-e7413569a767
<div align="center">
  <h1>🎭 Dublio: The Distributed Localization Orchestra</h1>
  <p><b>Zero-Copy VFS, BYOS (Bring Your Own Storage), and Native Engine Injection for Next-Gen Game Localization.</b></p>
  
  <a href="https://reactjs.org/"><img src="https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB" alt="React"></a>
  <a href="https://nextjs.org/"><img src="https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white" alt="Next.js"></a>
  <a href="https://www.typescriptlang.org/"><img src="https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white" alt="TypeScript"></a>
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python"></a>
  <a href="https://postgresql.org/"><img src="https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL"></a>
  <a href="https://unity.com/"><img src="https://img.shields.io/badge/Unity_C#-000000?style=for-the-badge&logo=unity&logoColor=white" alt="Unity"></a>
  <br><br>
</div>

> **Note:** This repository serves as an architectural showcase and technical overview of the Dublio ecosystem. The core source code remains private to protect proprietary AI pipelines and enterprise (B2B) security models.

## 📖 The Vision: End of the "Chaos Era"
For decades, game localization and modding have been plagued by fragmented workflows: lost `.zip` files in Discord channels, manual unzipping nightmares, and destructive file replacements. 

Dublio is not just a platform; it is a **Distributed Orchestra Philosophy**. It turns localization into an organized, frictionless pipeline where creators focus on art, and the machines handle the deployment.

* **The Brain (The Orchestrator):** A lightweight, centralized Node.js/PostgreSQL backend that tracks branching narratives, manages metadata, and dispatches asynchronous jobs.
* **The Muscle (The Orchestra):** A distributed network utilizing **BYOS (Bring Your Own Storage)**. Audio files, raw video tracks, and AI workloads (Whisper, Demucs) are executed directly on the users' connected Google Drive and Colab instances, resulting in an infinitely scalable infrastructure with near-zero egress costs.

## ⚙️ Beyond "Download & Unzip": Native Engine Injection
Unlike traditional modding tools that require users to manually replace files, Dublio features a custom-built, intelligent **Native Game Engine Patcher** (currently supporting 100% of Unity architectures, expanding to Unreal Engine).

**How the CLI Patcher Works (Under the Hood):**
* **Direct Asset Manipulation:** It does not overwrite raw game files. It dynamically reads `.assets` and `.bundle` files, locating the exact Byte/PathID of the target audio, texture, or font.
* **Memory-Safe Streaming:** Implements a strict `TEXTURE_STREAMING_THRESHOLD`. For textures larger than 16MB, it bypasses RAM accumulation by dynamically flipping (RGBA32) and streaming raw pixel data directly into `.resource` blocks.
* **Automated Decryption:** Automatically detects IL2CPP architectures and dynamically invokes memory dumpers to reconstruct DLL headers on the fly.
* **Format Restoration:** Custom C# routines parse TextMeshPro (TMP) font metadata, replacing `m_FaceInfo` glyph structures dynamically without breaking the game's UI layout.

## 🧠 Engineering Challenges & Solutions

Building a distributed, browser-first ecosystem required overcoming significant bottlenecks in memory management, cloud egress costs, and AI processing times. Here is how Dublio solves them:

### 1. "Prometheus" Video OCR Engine (Context Mapping)
Translators need context. Providing just audio files leads to blind translations. Dublio features a proprietary OCR engine that maps in-game text to video timelines.
* **The Challenge:** Processing a 6-hour gameplay video frame-by-frame takes immense compute power and time.
* **The Solution:** 
  * **GPU-Accelerated ROI:** Uses `ffmpeg -hwaccel cuda` to crop the user's defined Region of Interest (ROI) directly on the GPU before OCR begins.
  * **Adaptive Frame Skipping:** Implements a temporal memory loop using `thefuzz`. If consecutive frames yield >60% text similarity, the engine triggers an `ADAPTIVE_SKIP`, drastically reducing PaddleOCR inferences. A 6-hour video is processed in ~1 hour.
  * **Spatial Clustering:** Raw bounding boxes are intelligently merged to distinguish between multi-line subtitles and disconnected UI elements.

### 2. Client-Side WASM Transcoding & BYOS
Audio mixing requires specific formats (e.g., 48kHz, Mono, 16-bit PCM). Converting millions of assets on a centralized server would incur massive AWS/Cloudflare costs.
* **The Solution (Browser as an OS):** 
  * Dublio loads **FFmpeg WebAssembly (WASM)** dynamically via CDN. All transcoding happens strictly on the user's CPU, reducing server compute costs to $0.
  * **WASM Memory Management:** To prevent browser memory leaks during bulk operations, the WASM instance is programmatically terminated and re-initialized every 5 files.
  * **FileSystem Access API:** Downloads uncompressed ZIP payloads, decrypts the manifest, and automatically generates a hierarchical `/Workspace` folder structure directly on the user's local disk.

### 3. Smart Batch Uploader & Conflict Resolution
When a Voice Actor drops 500 recorded `.wav` files into the browser, manually assigning them to script lines is impossible.
* **The Solution:** A 4-tier Regex & Fuzzy Matching waterfall algorithm. 
  1. **Strict ID Match:** Checks for explicit tags (e.g., `_L88210.wav`).
  2. **Sequence Match:** Maps `seq-05` prefixes to project scenes.
  3. **Contextual Match:** Analyzes cinematic video names linked to the project.
  4. **Fuzzy Fallback:** Breaks down the filename and compares it against character names and dialogue keys.
  * If a collision occurs (two files match one line), the system halts and pushes the conflict to a manual resolution queue.

### 4. Zero-Copy VFS & The "AI-Mode" Ingestion
* **High-Velocity Pipeline:** When a game's master asset ZIP is uploaded, the "AI Mode Toggle" determines the pipeline. When disabled for speed, 40,000 files can be parsed, converted to `.opus` (for legal/fair-use safety), and metadata-extracted in just 50 minutes.
* **Byte-Offset Mapping:** Media files are packed using `ZIP_STORED` (uncompressed). The Python backend calculates the exact `start` and `end` byte offsets for each file, enabling direct HTTP Range-Request streaming from Backblaze B2 without extracting the archive.
## 🏗️ System Architecture & Data Flow

Dublio's infrastructure is divided into three distinct operational layers: The Edge Delivery Network, the Backend Orchestrator, and the Native Desktop Client.

```text
[Game Engine] <══(Byte-Range Stream)══> [Cloudflare Worker] <══(Direct Read)══> [Backblaze B2 Vault]
                                               ^                                        |
                                               |                                    (Uploads)
                                          (Presigned URLs)                              |
                                               |                                        |
[Dublio Desktop App] <══(Manifest & Jobs)══> [Next.js / Node.js Backend] <══(Queue)══> [Colab / Drive (BYOS)]
(Rust/Tauri)                                   (PostgreSQL)                              (AI Processing)
```

### 1. The Edge V2 Proxy (Zero-Copy Delivery)
A traditional file server sends whole files. Dublio uses a **Cloudflare Worker Edge Proxy** to perform "Data Illusions."
* **Byte-Offset Routing:** When a game engine requests an audio file, it hits the Cloudflare Worker. The worker does not download the file. Instead, it reads the `start` and `end` byte offsets from the PostgreSQL database and sends an HTTP `Range` request directly to a massive uncompressed `.zip` file sitting in Backblaze B2.
* **Header Spoofing:** As the specific byte chunk returns from B2, the Edge Worker intercepts it, dynamically injects the correct `Content-Type` (e.g., `audio/ogg`, `video/webm`), and serves it. The game engine believes it is reading a standalone file, unaware that it is dynamically streaming from inside a 10GB archive.

### 2. The Native Desktop Patcher (Rust / Tauri)
The Dublio Desktop application is not a simple downloader; it is a **State-Aware Differential Patcher** built in Rust for absolute bare-metal performance.
* **3-Tier Installation Matrix:**
  1. **Base Layer:** Injects the foundational asset mappings.
  2. **Version Patches (Differential):** Applies only the specific changes between the user's current version and the target version.
  3. **Hotfixes (Global Overwrite):** Applies critical, late-stage overrides (e.g., last-minute Voice Actor retakes) directly over the existing installation.
* **Garbage Collection & Rollback:** Before applying a new version, the Rust engine cross-references the previous `manifest.json`. It automatically hunts down "orphaned" files (files added in V1 but removed in V2) and deletes them. It then perfectly restores the original game files from `.dublio.bak` backups, ensuring the game directory never becomes corrupted.
* **System-Level API Access:** Bypasses standard browser sandbox limitations to deeply hook into the OS, safely manipulating protected game directories and tracking background Game PIDs.

### 3. S3/R2 Backend Integration & B2B Boundary
All data ingested into the system passes through a secure AWS SDK pipeline. 
* Implements robust `MultipartUpload` protocols for massive asset archives.
* Generates short-lived, time-sensitive Presigned URLs for edge delivery.
* **The Iron Wall (Constitution Article 2):** Ensures strict compartmentalization. Enterprise (B2B) assets are processed in isolated, NDA-protected S3 buckets, completely decoupled from the community (BYOS) storage nodes.

## 🎨 The Interface: A Studio in the Browser

A powerful distributed backend is useless if the frontend creates friction. Dublio's UI/UX philosophy is built on **"Productivity, Precision, and Spatial Awareness,"** wrapped in a sleek, minimalist dark theme. It transforms the chaotic localization process into a visual, gamified experience.

### 1. The Curation Canvas (React Flow)
The heart of the workflow. Instead of staring at endless spreadsheets, users build the narrative logic visually.
* **Hybrid Workflow Modes:**
  * **Wizard Mode:** An automated, guided setup for rapid batch linking.
  * **Canvas Mode:** An infinite, spatial node editor built with `ReactFlow`. Audio blocks, text nodes, and video timings are visually connected via edges, allowing users to literally "see" the branching narrative of the game.
  * **Demo Mode:** An instant preview environment to test the curated scene before deployment.
* **Pro-Level UX:** 
  * High-performance Drag & Drop powered by `@dnd-kit/core`.
  * **State Awareness:** Real-time sync indicators and an amber-400 "Dirty State" light for unsaved spatial changes.
  * Context menus, keyboard shortcuts (e.g., Space to play/pause), and a floating video player synced perfectly with the timeline editor.
  * **AI Assistant Launcher:** Integrated directly into the canvas to auto-resolve broken links or missing metadata.

https://github.com/user-attachments/assets/bfead1e2-17e6-412f-8826-5bfd4b00a15a

### 2. Gamified Discovery (The Homepage Portal)
The homepage abandons the traditional "SaaS Dashboard" look in favor of a "Worlds within Worlds" gaming portal concept.
* **The Nexus:** A visual network mapping the ecosystem of translation teams, their active projects, and members.
* **The Carnival:** Game genres are presented as interactive "Loot Boxes." Projects are assigned RPG-style rarities based on their translation volume:
  * ⚪ **Common:** Standard mods.
  * 🔵 **Rare:** Medium-scale projec

ts (30,000+ lines).
  * 🟡 **Legendary:** Massive AAA-scale undertakings (80,000+ lines).
* **Cinematic Gallery:** A seamless background mosaic of 50+ game covers with dynamic WebM video previews injected upon hover.

https://github.com/user-attachments/assets/c40de33f-916c-429c-8a2c-08fc62b5a82f

### 3. Role-Specific Workshops
Every role (Translator, Voice Actor, Audio Mixer) gets a dedicated, distraction-free environment.
* **Translation Workshop:** Features side-by-side video context, reference audio playback, and a community-driven reputation/badge system to gamify translation accuracy.
* **Dubbing & Mixing Workshop:** Voice actors can record directly in the browser while watching the scene, or use the "Batch Sync" modal to upload external DAW recordings. 

<img width="720" height="354" alt="dublaj_kucuk" src="https://github.com/user-attachments/assets/cfe915e7-7a7d-4381-8f4a-6b54382c1595" />

## 🔒 Security, Legal Strategy & B2B Boundary

A system of this scale cannot rely solely on technical architecture; it requires an ironclad legal and operational strategy. This is why the core repository remains closed-source.

**The "Double Pool" Ecosystem:**
* **Pool A (The Community Workshop):** A completely free, zero-friction environment where independent modding teams use their own Google Drive storage (BYOS) and Colab limits. This builds the talent network and proves the technology at scale.
* **Pool B (The Enterprise / B2B Studio):** The commercial future of Dublio. Game studios require absolute secrecy (NDAs). An unreleased script or audio asset cannot touch a user's personal Google Drive. 
* **The Iron Wall:** The codebase is designed with strict compartmentalization. Enterprise assets are processed in deeply isolated, highly secure S3 vaults. Keeping the source code private protects this proprietary B2B security boundary and our customized AI ingestion pipelines from reverse-engineering.

## 🚀 The Road Ahead & Contact

Dublio was architected and built as a solo-founder endeavor over the course of a year. It proves that with the right distributed architecture, a single Node.js backend can orchestrate a global network of community-driven computing power.

The system is currently in active deployment, transitioning from the "Community Proof-of-Concept" phase toward industry-scale beta tests.

**Interested in the architecture or discussing professional opportunities?**
I am currently exploring roles in **Cloud Architecture, Backend Engineering, and DevOps.** If you are looking for an engineer who understands how to scale complex systems from zero to production while maintaining strict cost-efficiency, let's connect.

📫 **Reach out to me via:**
* [LinkedIn Profile](https://www.linkedin.com/in/emre-bulut-0a1827315/?profileFormEntryPoint=IntroForm) 
* [Email Address](mailto:005emreebulutt005@gmail.com) 
* Or check out my other open-source snippets and gists on this GitHub profile.

---
*Architected and developed with passion by Chimiya.*
