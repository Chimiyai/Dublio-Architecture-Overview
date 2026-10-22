# Dublio — Next-Generation Game Localization, Dubbing & Community Ecosystem

**Browser-Based Super Studio · BYOS Architecture · Native Engine Injection**

Dublio is an end-to-end game localization platform that brings translation, dubbing, fonts, textures, mixing, quality control, and in-game deployment under a single roof. Instead of ZIP files drifting around Discord, spreadsheets, and manual file replacements inside game folders, it provides a structured, team-based pipeline with one-click installation. It is designed to scale from independent community projects to studio-level requirements.

> **Note:** This repository is an architectural showcase. Core source code remains private to protect proprietary AI pipelines and enterprise (B2B) security boundaries.

---

## The Problem

Localizing a video game is fundamentally different from translating traditional software. A game project demands flawless harmony between branching narratives, complex scene structures, character personalities, thousands of scattered audio files, textures, and font atlases. Yet most communities still struggle with the same constraints:

- **Disconnected workflows:** Translators work in Excel or Notepad without context, voice actors try to track timing in standalone audio editors, and mixers manually pair each line to its scene.
- **Storage and distribution costs:** Hundreds of gigabytes of raw video and tens of thousands of audio files place a heavy infrastructure burden on independent and non-profit teams.
- **No oversight or quality assurance:** Without a shared multiplayer working environment, version conflicts, lost lines, and synchronization errors are inevitable.

Dublio's goal is to remove that overhead: **let creators focus purely on localization work while the infrastructure handles asset organization, processing, and delivery to players.**

---

## Architecture Overview

Dublio operates across four main layers: a browser-based studio interface, an edge delivery network, a lightweight backend orchestrator, and a native desktop patching client. Heavy AI workloads are pushed out to user-controlled resources rather than centralized servers.

```
[Game Engine] <══(Chunked Streaming)══> [Edge Gateway] <══(Direct Read)══> [Cloud Object Storage]
                                              ^
                                       (Signed URLs)
                                              |
[Dublio Desktop] <══(Manifest & Jobs)══> [Next.js / Node.js Backend] <══(Queue)══> [GPU Workers / Drive (BYOS)]
 (Rust / Tauri)                             (PostgreSQL)                              (AI Processing)
```

### Backend Orchestrator

A lightweight Node.js + PostgreSQL stack manages project data, localization assets, branching narrative structure, job queues, and package versions. With over eighty relational models, the platform keeps each game's text, audio, characters, and scene connections tightly linked. Long-running AI and file-processing jobs are tracked through a distributed job queue with clear states (pending, processing, completed, failed).

### Master Catalog & Team Projects

When a game is first added to the platform, its raw text, characters, and audio assets are indexed into a global, read-only **Master Catalog**. When a team takes on the game, it spins up its own workspace from the catalog. This allows multiple teams to work on the same game with their own translation and dubbing vision, without overwriting each other's data.

Dialogue is stored as a node-based graph rather than flat text rows, so branching conversations, player choices, and scene transitions are represented natively.

### BYOS — Bring Your Own Storage

Instead of centralizing expensive compute and storage, Dublio distributes the workload to user-controlled resources: Google Drive, Google Colab, and local CPU/GPU. This lets the infrastructure scale horizontally at near-zero cost.

- **Automatic workspace setup:** When a team connects its Google Drive, the platform builds the full folder structure for projects (raw videos, vocal references, dubbing takes, mixed stems, textures) automatically.
- **Direct-to-Drive uploads:** Large audio and video files are uploaded straight into the user's own storage through a serverless bridge, so they never pass through Dublio's servers and are not limited by Drive's API quotas. Progress appears as live logs in the command panel.
- **Smooth streaming:** Large videos and audio files play in the browser without stalling, streamed from Drive on demand.
- **Self-healing links:** If a file is moved or replaced in Drive, the platform finds it again and repairs the reference automatically.
- **Guest protection:** Voice actors invited from outside a team can only access the audio of the lines assigned to them, never the team's full storage.

### Edge Delivery & Large-File Handling

Thousands of audio and texture files are presented to the game engine and the browser as individual files, while being stored in an efficiently packed form behind the scenes. This approach sharply reduces storage and bandwidth costs while giving players a smooth experience.

- Very large files are uploaded in parts, and only failed parts are retried after a network interruption.
- Files travel directly between the user's device and storage through short-lived signed links, keeping server bandwidth near zero.
- Temporary files, old logs, and leftovers of deleted projects are cleaned up automatically in bulk.
- The database is backed up nightly into versioned cloud storage.

### Local-First Studio Experience

Dublio feels like desktop software even on weak connections or very large projects.

- Every project gets its own isolated local workspace inside the browser.
- Project data with tens of thousands of lines streams in the background without ever freezing the interface.
- An integrity checker compares local and server data periodically and repairs only the missing or damaged items.
- Heavy work such as matching transcripts to script lines and finding similar sentences runs in background workers.

---

## What the Platform Does

### 1. Source File Parsing

Dublio understands the text and data formats of many game engines and prepares them for localization:

- **Ren'Py** (`.rpy`) — including translate blocks, speaker tags, and voice references
- **Unity I2 Localization** (`.json`) — including text, audio, and sprite term types and multi-language tables
- **Unreal Engine** (`.locres`)
- **GNU Gettext** (`.po`, `.pot`) — including context, plural forms, and translator comments
- **Construct 2/3**
- **INI / Config / Properties**
- **Subtitle formats** (`.srt`, `.vtt`)
- **Plain text** files with character-name prefixes and custom delimiters

The type of an uploaded file is detected automatically. Character names, context information, and plural forms are preserved on import.

### 2. AI-Assisted Data Preparation

Even when a game ships without a text file, Dublio can extract the content. Heavy jobs are handed off to distributed workers running on teams' own GPU resources.

- **Prometheus Video OCR:** Reads on-screen subtitles and character names from gameplay videos and turns them into a clean line list with exact timecodes. Repeated frames are merged into single lines, and a hybrid of two OCR engines improves accuracy on stylized game fonts. Translators work from the scene, not from audio alone.
- **Audio Classification:** Sorts game audio into seven categories: dialogue, non-dialogue vocals (screams, breaths, battle cries), sound effects, ambience, music, whispers, and humming. Dialogue is prioritized for dubbing.
- **Speech Transcription:** Audio is transcribed and automatically matched to the lines in the script.
- **Stem Separation:** Original vocals are separated from music and ambience. Voice actors work against a clean vocal reference, while the original music and environmental sound are preserved for mixing.
- **Video Ingest:** High-resolution gameplay videos are uploaded as unlisted videos to the team's own YouTube channel and played inside the platform, creating no extra storage cost for the team.

### 3. Curation Studio

The hub where raw data is cleaned and prepared for translation. It offers three working modes:

- **Wizard Mode:** A keyboard-driven, high-throughput screen. Source lines sit on one side, with video and waveform on the other.
- **Canvas Mode:** An infinite node canvas. Lines, audio, choices, scene sequences, map transitions, and video groups are connected visually, so branching stories are modeled in space.
- **Demo Mode:** An interactive preview of how the prepared content will be experienced from the player's perspective.

**Source Explorer**
- Smooth browsing of source lists with more than 70,000 lines
- Regex search, a folder-tree view, character filters, and bulk selection
- Detail panels for lines, video segments, and audio waveforms
- Bulk actions: assign characters, delete, or change status for many lines at once

**Canvas**
- Text nodes that glow and show a progress bar while their line is playing
- Audio nodes that sync characters automatically and warn when several characters merge into one audio file
- Sequence and video group containers, plus choice and map nodes for branching
- Automatic simplification of nodes when zoomed out, keeping huge canvases light
- A searchable story map view, and canvas import/export as JSON with a diff check

**Video & OCR Tools**
- Viewing OCR boxes by second while the video plays, either only those matched to lines or all text in the video
- A watermark eraser: draw a resizable area on the video and clear all OCR boxes inside it in one click, perfect for channel logos and broadcaster marks
- A cinematic magnifier that zooms into a line's on-screen position as it is inspected
- A clean video player that hides YouTube's own title, logo, and controls, replaced by Dublio's custom control bar

**Timeline Editor**
- Overlapping lines are automatically distributed onto separate tracks
- Line edges snap magnetically to other lines and to video scene cuts
- Dragging two consecutive OCR boxes together merges them, with a ghost preview, and closes the gap between them

**Characters & Personas**
A character's different appearances or disguises are tied to one main character through sub-personas (for example, "Bruce Wayne" with "Batman" and "Mysterious Man"). In the Dubbing Studio, all personas are synchronized into a single voice actor's task list.

**Speech-to-Text Studio**
A seven-tab workspace that connects transcribed audio to script lines: pending, processing, processed, review, other sounds, assign to line, and relink. High-confidence matches are accepted automatically, medium-confidence ones go to a curator for review, and low-confidence ones fall into a manual matching queue.

### 4. Translation Studio

A team-driven workspace for translating huge texts smoothly.

- **Four-column translation grid:** ID/character, source text, target translation, and status/actions. It handles 100,000+ lines with stable memory use.
- **Grouped lines:** Lines are grouped under sequence and video-group header strips, each with a completion percentage.
- **Jump to selected:** After a search or an external link, the grid animates straight to the selected line.
- **Context drawer:** A resizable side drawer with three tabs that remembers your preferred tab:
  1. **Video:** The exact video moment of the line, with a loop mode over the line's range.
  2. **Audio:** Vocal reference played together with the background music, so tone and emphasis can be felt.
  3. **File content:** The raw source file, scrolled to the active line and highlighted, so translators see surrounding branches and neighboring dialogue and choose the right terminology.
- **Shared line management:** Lines that repeat across many files get an "in X files" badge. Sibling copies are listed, and one click jumps to any of them. With auto-sync enabled, saving a translation updates all identical lines across all files.
- **Master Catalog auto-fill:** Previously approved translations can be pulled into every matching empty line of a project in one click.
- **Community translation:** Any user can suggest an alternative translation, but not vote on their own. Votes update instantly in the interface, and the experience level of voters is taken into account. The top suggestion automatically becomes the official translation, and leadership changes hands when another suggestion overtakes it, with the experience reward moving to the new leader.
- **Contribution analytics:** Who actually authored a translation is measured word by word. Changing only punctuation does not transfer ownership, and the contribution split (for example, 70% one contributor, 30% another) is shown transparently.
- **Real-time multiplayer:** Teammates' avatars appear on the lines they are working on, a typing animation shows live activity, drafts are visible before they are saved, and one click jumps to a teammate's current line.
- **Support mode** to restrict a translation to the core team
- **Keyboard shortcuts** for saving, approving and advancing, moving between lines, and playing audio

### 5. Font Studio

Many game fonts lack Turkish characters (ç, ğ, ı, ö, ş, ü and their capitals). Font Studio solves that problem.

- Injecting missing Turkish characters into the game's original font
- Automatic fitting: if a new glyph overflows its slot in the original atlas, it is scaled down until it fits
- Fine-tuning controls for padding, sharpness, and font size
- **Font DNA matrix:** A live, color-coded matrix shows at a glance which characters exist in the original, which were injected, which are critically missing, and which are not needed
- Export to Unity TextMeshPro and a simple JSON format for custom engines
- A live font and atlas debugger for inspecting the generated atlas at pixel level, with results savable to Google Drive

### 6. Art & Texture Studio

Signs, maps, menu buttons, and UI graphics inside the game are localized here.

- A virtualized vertical film strip for browsing thousands of textures
- Classification filters: all, unclassified, text-based (needs translation), and purely graphical, with a checkerboard background for transparent images
- A layer-based canvas: the original texture stays locked at the bottom, while text boxes, brush strokes, and patches are added on top. Layers can be reordered by drag and drop.
- Zoom from a 25% overview up to 500% for pixel-precise alignment
- A/B comparison between the original and localized texture, by toggle or slider
- Regional crop and PNG download, to send a single sign to external software such as Photoshop
- Version history (v1.0, v1.1, …) with one-click rollback
- A guided Bink (`.bk2`) video hub for converting in-game cutscenes, embedding subtitles, and converting back

### 7. Dubbing Studio

Turns the browser into a professional recording studio.

- **Lossless recording:** Microphone audio is captured unprocessed at studio quality (48 kHz, 16-bit PCM WAV), bypassing the browser's noise suppression and auto-gain.
- **Scene-synchronized recording:** A 3-2-1 countdown starts the recording, and the reference video and original audio play automatically, so voice actors record while hearing the original performance in their headphones.
- **Live level metering:** A hardware-style VU meter with peak hold and a clipping warning, plus a live neon waveform drawn while recording.
- **ADR timeline:** Original vocal reference, clean backing music, and the new recording are stacked together, with highlighted regions marking the original line's range and a zoom from overview down to syllable-level alignment.
- **Overflow marking:** When a take exceeds the original line's duration, the overflow is flagged on the waveform, so actors see they need to speak faster.
- **In-place text editing:** If a Turkish sentence does not sit well in the mouth, the actor can revise the line right from the text box above the waveform.
- **Flexible line grouping:** Sequence, video sequence, audio-file, and standalone groups, with merging, splitting, and adding custom sections for breaths or exclamations.
- **Unlimited take management:** Any number of attempts per line, with the best one starred as the preferred take and sent to the Mixing Studio as the official recording.
- **Loss prevention:** Recordings are saved locally first and then uploaded in the background, so no recording is lost even if the connection drops.
- **Voice actor discovery pool:** Team leaders can filter voice actors by gender and vocal tone (deep, young, cartoon, and so on) and shuffle through short random clips of their past recordings for quick casting.
- **Community voice acting:** In open projects, users can claim unassigned characters and leave a voice sample, and the role is assigned once an admin approves.

### 8. Mixing Studio

A complete digital audio workstation running in the browser.

- **Multi-channel timeline:** Unlimited channels (dialogue, group/foley, effects, music, and more), each with mute and solo
- **Two working modes:** Select mode (Q) for dragging and moving blocks between channels, and blade mode (W) for splitting a block at any point
- Trimming, resizing, fade-in and fade-out
- Real-time scrubbing while dragging the playhead, and 0.5x–2.0x playback speed
- **Per-channel effects chain:** 4-band EQ, compressor, reverb, and delay, each with instant bypass and level indicators
- **Stem channels:** One-click recall of the original music and vocal reference separated by Demucs, so the Turkish dub can be tested against the original soundtrack immediately
- **Master render:** The final mix is rendered as a 44.1 or 48 kHz WAV directly in the browser, without a server
- **Timing sync:** Moving or trimming audio in the mixer updates the line's timing in the project
- **Retake requests:** Send a Retake request with revision notes straight to the voice actor in the Dubbing Studio when a recording is unsuitable
- **External DAW integration (Reaper, Pro Tools, Premiere):**
  - Export in character-first, sequence-first, or flat folder hierarchies
  - On Chrome and Edge, files are written directly into a folder on the user's disk without zipping
  - Bulk re-import of externally mixed audio, automatically matched back to the right lines by file name

### 9. Quality Control (QC) Studio

All components are tested end to end in the browser before they reach the game.

- **Interactive GamePlayer:** Click through branching dialogues like a player, pick options when choices appear, and use auto-voice mode, which waits for each line's audio before advancing. Turkish dubbing and background music play together in sync.
- **Color-coded line markers:** The video progress bar marks lines with pins: green for approved, red for revision requested, amber for not yet reviewed, and purple for the currently playing line.
- **Hierarchical issue reporting:** Testers pause the video and file a report in one of four categories: translation, dubbing, mixing, or curation. The exact moment in the video is stamped automatically.
- Side-by-side comparison of original and Turkish audio waveforms
- Approving and locking clean lines individually or in bulk

### 10. Modding & Packaging

Translated and voiced assets are packaged in the way each game engine expects.

- **A "Repack Recipe" system:** Text can be output by mirroring the original file structure exactly, by merging everything into a single master file, or by injecting only the changed lines into an existing game archive.
- Per-project definition of audio format (OGG, WAV, MP3), bit rate, and placement and naming rules inside the game's folders
- A build manifest that modding tools read
- **Signed, purpose-built tools** for popular engines, triggered from the desktop client:
  - **Dublio Packer** — general-purpose archive packer
  - **GameMakerDublio** — embeds text and fonts into GameMaker games
  - **UnrealEngineDublioPacker & UECook** — archiving and cooking for Unreal Engine 5.6.1+
  - **FmodDublioPacker** — rebuilds FMOD sound banks
  - **TelltaleDublio** — Telltale engine integration
- **Incremental hotfix system:** A small fix does not require rebuilding the entire package. Only changed assets are converted and distributed as a small hotfix layer, which the player client applies on top of the base package.
- **Phase locks:** When a phase is marked complete, the people still holding pending work are listed and the project manager can be nudged. The project is temporarily frozen while a package is being built, and users below a certain experience level cannot accidentally change critical workshop settings or completed phases.

---

## Native Engine-Level Patcher

Unlike traditional modding tools, Dublio provides an intelligent patching system that does not require manual file replacement.

- Fully supports Unity architectures, and integrates with packaging tools for Unreal Engine and other engines.
- Safely replaces audio, texture, and font assets directly inside the game's own data files.
- Handles large textures without creating memory pressure.
- Updates TextMeshPro font data without breaking UI layout.

---

## Desktop Client (Rust / Tauri)

The desktop application is a state-aware patching client that delivers the patch to players with a single click.

- When a player presses **"Install Patch"** on the website, the `dublio://install` protocol opens the client, detects the game automatically, and starts the installation.
- **Three install options:** Full Turkish (text + dubbing), dubbing only, subtitles only
- **Three-layer installation structure:**
  1. **Base Layer** — the main patch package
  2. **Version Patches (Differential)** — the delta between the current version and the target version
  3. **Hotfixes** — last-minute fixes (e.g., late-arriving retakes)
- Before an update, the client compares against previous installation records, cleans up leftover files, and restores original game files from backups. The game directory is never left in a corrupted state.
- Signed modding tools can be triggered directly from inside the studio

---

## Teams, Projects & Community

### Teams
- **Team showcase:** A public profile with completed projects, active roster, and achievement badges
- **Studio management tabs:** Identity (logo, bio, social links), Recruitment (open roles such as translator or voice actor), Specialties (workshop permissions of members), and Danger Zone (ownership transfer and disbanding)
- **Studio vitals dashboard:** In-team MVP ranking, stage distribution of projects, and weekly word and audio production metrics
- **Activity center and project launch pad:** Teams start new projects in one click, and rosters are built through applications and invitations

### Roles & Permissions
Every action passes through a fine-grained permission check across three levels:
- **Platform roles:** Super Admin, Admin, Observer, User
- **Team roles:** Leader, Admin, Member
- **Workshop specialties:** Voice Actor, Translator, Editor, Audio Engineer, Artist, Developer, QA Tester, Curator

Multiple teams can collaborate on the same project, sharing rights and visibility. For example, one team handles translation while another runs dubbing production.

### Projects
- **Three workflow types:** Full production (curation → translation → dubbing → mixing → testing → release), dialogue only, and text only for classic subtitle patch projects
- **Stage progression:** Curation → Translation → Dubbing → Mixing → Final Review → Completed
- **Smart time estimator:** Remaining word count, the team's daily completion pace, and audio recording durations are analyzed to calculate a projected completion date dynamically
- **Production hub:** A single cockpit summarizing the progress of all five workshops
- **Volunteer pool:** Independent users who want to help on a project can apply
- **Steam catalog:** Entering a game's AppID pulls descriptions, screenshots, genre tags, and release dates in 11 languages
- **3D game cards:** Interactive cards with a perspective tilt that follows the mouse
- **Dynamic project covers:** Dialogue box positions from the game are marked on the cover through an ROI selector, with letterbox and pillarbox compensation
- **Community proposals and game requests:** Users open proposals for games they want localized and vote on them, and the most requested games land on teams' radar

### Real-Time Communication
- **Floating messaging dock:** Draggable, resizable, minimizable chat windows, with Picture-in-Picture mode
- **Peer-to-peer group calls:** Voice and video calls directly between participants, with no media load on Dublio's servers
- **Screen sharing with system audio:** Translators and testers can broadcast the game screen or video player live to teammates
- **Adaptive quality:** Stream resolution adjusts automatically with network quality, from 480p to 1080p, with live latency and packet loss counters
- **Speaking indicator:** A green neon ring lights up around the avatar of whoever is speaking
- **Global presence:** See who is online and which workshop they are in
- **In-browser media compression:** Images and videos are converted to WebP or compressed MP4 before being sent, and voice messages go out with a waveform preview

### Contextual Sticky Notes
- Notes are anchored to a page URL (even a specific line), so everyone who visits that page sees the same note
- Five color themes and a dual-pane Markdown editor with live preview
- Owner, editor, and viewer permission levels
- Global announcement notes that admins can pin across the whole platform
- A file-explorer-style tree view of all project notes, with folders, drag and drop, and media attachments

### Social Ecosystem
- **Artist portfolios:** Voice actors' demos, tone tags (such as "Monster" or "Young Lover"), roles played, and completed line counts
- **Reputation record:** A transparent record for every user, with commendations (completed projects, thanks from team leaders, translation awards) and penalties or warnings (abandoning projects without cause, sabotage attempts, inappropriate language)
- **Community forums:** Four channels for general discussion and game worlds, translation and terminology, voice acting and dubbing craft, and technical support and modding guides
- **A hidden easter egg page:** A secret 3D space where the names of everyone who contributed to the platform float as stars

---

## Gamification

Dublio offers an experience system that rewards localization effort and does not tolerate abuse.

- **Ten levels,** each with its own title: Apprentice, Intern, Senior Translator/Voice Actor, Master Artist, Workshop Chief, Studio Architect, Grand Master, Legendary Localizer, Dublio Council Member, and Immortal Localization God.
- **Rewarded work:** Approved translations, taking the lead on a suggestion, starred dubbing takes, approved mixes, verified audio-to-text matches, approved textures, successfully tested font atlases, and confirmed QC reports each earn experience.
- **Penalties:** Sabotage, such as deleting approved lines or entering meaningless text, is met with heavy penalties, including direct loss of rank.
- **Level-based trust:** Higher levels carry more weight in translation votes, and Level 5 unlocks access to critical workshop settings.
- **Achievement badges:** Metallic 3D badges that react to mouse movement, in Bronze, Silver, Gold, Platinum, and Legendary tiers

---

## Administration & Statistics

### Ecosystem Dashboard
- Nightly snapshots of platform-wide totals: words translated, dubbing time recorded, fonts and textures produced, active teams and users
- Interactive timeline charts showing growth over the months
- An infrastructure health monitor covering Google Drive quotas, cloud storage volume, GPU worker response times, and database connection pool usage

### Admin OS
- **Master Data Wizard:** Adding new games to the Master Catalog, triggering Steam sync, and parsing raw files
- **Approvals:** Reviewing team projects, publishing completed patches, and approving community proposals
- **User and team management:** Freezing accounts, role assignments, and resolving ownership disputes
- **Report inspector:** A drawer for reviewing user complaints and bug reports, with a diff lens that compares the earlier and later versions of a disputed translation or audio
- **Client update hub:** Publishing new versions of the desktop client
- **Audit logs:** A second-by-second record of who changed what, when, and from which IP
- **Backups:** One-click manual database backup and restore from a previous backup
- **GPU queue control:** Worker status and manual routing of queued OCR and stem separation jobs
- **Maintenance mode:** A global shield that temporarily pauses user access during critical updates

---

## Interface

### Curation Canvas

Instead of spreadsheets, users build the localization narrative visually in an infinite node editor (`ReactFlow`). Audio blocks, text nodes, and video timings are connected via edges.

- **Wizard Mode** — guided setup for rapid batch linking
- **Canvas Mode** — spatial node editor with real-time sync indicators
- **Demo Mode** — instant preview environment before deployment

https://github.com/user-attachments/assets/bfead1e2-17e6-412f-8826-5bfd4b00a15a

### Role-Specific Workspaces

Every contributor role gets a dedicated, focused environment:

- **Translators** — side-by-side video context, reference audio playback, community reputation and badge system
- **Voice Actors** — in-browser recording synced to the scene, or batch upload from an external DAW

<img width="720" height="354" alt="dubbing workshop" src="https://github.com/user-attachments/assets/cfe915e7-7a7d-4381-8f4a-6b54382c1595" />

### Discovery Portal

The homepage is designed as a gaming portal rather than a standard SaaS dashboard. Projects are assigned RPG-style rarities based on translation volume, and active translation teams and their members are mapped on a visual network.

https://github.com/user-attachments/assets/c40de33f-916c-429c-8a2c-08fc62b5a82f

---

## Security & B2B Boundary

**The Double Pool Model:**

- **Pool A (Community)** — a free environment where independent teams use their own BYOS storage and Colab resources. It builds the talent network and validates the technology at scale.
- **Pool B (Enterprise)** — a commercial pipeline for game studios requiring NDA-protected asset handling. Enterprise assets never touch user-controlled storage; they are processed in isolated, secure storage with short-lived signed URLs.

Strict separation between the two pools is a core architectural constraint, not an afterthought. Keeping the source code private protects this boundary as well as the proprietary AI pipelines.

Additionally:

- Users' Drive access tokens are stored encrypted.
- Guest voice actors invited from outside a team can only access the audio files of the lines assigned to them.
- Fine-grained permission checks at platform, team, and workshop level, with all critical actions written to audit logs.
- The database is backed up every night.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | React, Next.js (App Router), TypeScript, Tailwind CSS, ReactFlow, Zustand, TanStack Virtual, Konva, Three.js |
| Backend | Node.js, PostgreSQL, Prisma ORM |
| Local Workspace | Dexie.js (IndexedDB), Web Workers |
| Desktop Client | Rust, Tauri |
| Media Processing | FFmpeg (WASM + CUDA), PaddleOCR, YAMNet, Whisper, Demucs, Web Audio API, Wavesurfer.js, opentype.js |
| Storage & Delivery | Google Drive, Google Apps Script, Cloudflare R2, Cloudflare Workers |
| Real-Time Communication | Pusher Channels, WebRTC |
| AI Processing | Google Colab (CUDA) |
| Engine Patching | Unity, Unreal Engine, GameMaker, FMOD, Telltale |

---

## Current Status

Dublio is a solo-built project, architected and developed over roughly one year. It is currently in active use, progressing from a community-scale proof of concept toward industry-scale beta testing.

---

*Architected and developed by Chimiya.*
