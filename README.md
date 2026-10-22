# Dublio — Game Localization, Dubbing & Community Platform

**Browser-Based Studio · BYOS Architecture · Native Engine Injection · Any Language**

Dublio is an end-to-end game localization platform that brings translation, dubbing, fonts, textures, mixing, quality control, and in-game deployment under a single roof. Instead of ZIP files drifting around Discord, spreadsheets, and manual file replacements inside game folders, it provides a structured, team-based pipeline with one-click installation. It supports localization into any language, and is intended for everything from independent community projects to studio-level requirements.

> **Note:** This repository is an architectural showcase. Core source code remains private to protect proprietary AI pipelines and enterprise (B2B) security boundaries.

---

## The Problem

Localizing a video game is fundamentally different from translating traditional software. A game project has to keep branching narratives, complex scene structures, character personalities, thousands of scattered audio files, textures, and font atlases consistent with one another. Most communities still work under the same constraints:

- **Disconnected workflows:** Translators work in spreadsheets or text editors without context, voice actors try to track timing in standalone audio editors, and mixers manually pair each line to its scene.
- **Storage and distribution costs:** Hundreds of gigabytes of raw video and tens of thousands of audio files place a heavy infrastructure burden on independent and non-profit teams.
- **No oversight or quality assurance:** Without a shared multiplayer working environment, version conflicts, lost lines, and synchronization errors are common.

Dublio's goal is to remove that overhead: **let creators focus purely on localization work while the infrastructure handles asset organization, processing, and delivery to players.**

---

## Any Language, Any Game

Dublio is not built around a single target language. A project can be localized into any language: text translation, voice dubbing, font support, and texture editing all work the same way regardless of which language a team is working in. Different teams can localize the same game into different languages side by side, each in their own workspace, without affecting one another.

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

A lightweight Node.js + PostgreSQL stack manages project data, localization assets, branching narrative structure, job queues, and package versions. With over eighty relational models, the platform keeps each game's text, audio, characters, and scene connections tightly linked. Long-running AI and file-processing jobs are tracked through a distributed job queue with clear states (pending, processing, completed, failed), so long-running jobs can run unattended and failures are visible.

### Master Catalog & Team Projects

When a game is first added to the platform, its raw text, characters, and audio assets are indexed into a global, read-only **Master Catalog**. When a team takes on the game, it spins up its own workspace from the catalog. This matters because multiple independent teams can work on the same game, with their own translation choices and dubbing vision, without overwriting each other's data. The catalog also acts as shared memory: translations approved earlier can be reused by later projects.

Dialogue is stored as a node-based graph rather than flat text rows, so branching conversations, player choices, and scene transitions are represented natively. This is what lets the platform preview a game's story flow the way a player would experience it.

### BYOS — Bring Your Own Storage

Instead of centralizing expensive compute and storage, Dublio distributes the workload to user-controlled resources: Google Drive, Google Colab, and local CPU/GPU. This lets the infrastructure scale horizontally at near-zero cost, which is what makes a free community tier possible.

- **Automatic workspace setup:** When a team connects its Google Drive, the platform builds the full folder structure for each project (raw videos, vocal references, dubbing takes, mixed stems, textures), so nobody has to organize folders by hand.
- **Direct-to-Drive uploads:** Large audio and video files are uploaded straight into the user's own storage through a serverless bridge. They never pass through Dublio's servers and are not limited by Drive's API quotas. Progress appears as live logs in the command panel.
- **Smooth streaming:** Large videos and audio files play in the browser without stalling, streamed from Drive on demand, so editors can seek through a multi-gigabyte recording without downloading it first.
- **Self-healing links:** If a file is moved or replaced in Drive, the platform finds it again and repairs the reference automatically, so a reorganized Drive does not break a project.
- **Guest protection:** Voice actors invited from outside a team can only access the audio of the lines assigned to them, never the team's full storage.

### Edge Delivery & Large-File Handling

Thousands of audio and texture files are presented to the game engine and the browser as individual files, while being stored in an efficiently packed form behind the scenes. This reduces storage and bandwidth costs.

- Very large files are uploaded in parts, and only failed parts are retried after a network interruption, so a dropped connection does not mean starting a multi-gigabyte upload over.
- Files travel directly between the user's device and storage through short-lived signed links, keeping file traffic off Dublio's servers and making the links expire.
- Temporary files, old logs, and leftovers of deleted projects are cleaned up automatically in bulk, so storage does not silently fill up.
- The database is backed up nightly into versioned cloud storage, so the database can be restored from an earlier backup.

### Local-First Studio Experience

The studio is built to stay responsive on weaker connections and with very large projects.

- Every project gets its own isolated local workspace inside the browser, so lists with tens of thousands of lines load from local storage.
- Project data loads in the background so the interface stays usable.
- An integrity checker compares local and server data periodically and repairs only the missing or damaged items, so the local copy stays in sync with the server.
- Heavy work such as matching transcripts to script lines and finding similar sentences runs in background workers, keeping the interface responsive.

---

## What the Platform Does

### 1. Source File Parsing

Games store their text in very different ways. Dublio reads the most common formats and turns them into a uniform, translation-ready structure, keeping each file's original organization so it can be rebuilt afterwards:

- **Ren'Py** (`.rpy`) — reads translate blocks, old/new text pairs, speaker tags, and voice file references, so each line arrives with its speaker and its audio.
- **Unity I2 Localization** (`.json`) — reads hierarchical keys and terms of type text, audio, or sprite, along with multi-language tables.
- **Unreal Engine** (`.locres`) — reads the binary localization files, including namespaces and string tables.
- **GNU Gettext** (`.po`, `.pot`) — reads context, source and target strings, plural forms, and translator comments.
- **Construct 2/3** — reads the three-dimensional data arrays that store dialogue in these engines.
- **INI / Config / Properties** — reads section and key-value pairs, including special escape characters.
- **Subtitle formats** (`.srt`, `.vtt`) — reads timecoded blocks and strips formatting tags.
- **Plain text** — reads line-based formats with character-name prefixes (such as `HERO: Welcome`) and custom delimiters.

The type of an uploaded file is detected automatically, so users do not have to choose a format by hand.

### 2. AI-Assisted Data Preparation

Even when a game ships without a text file, Dublio can extract the content. Heavy jobs are handed off to distributed workers running on teams' own GPU resources.

- **Prometheus Video OCR:** Reads on-screen subtitles and character names from gameplay videos and turns them into a clean line list with exact timecodes. It separates the subtitle area from the speaker-name area, merges repeated frames into single lines, and combines two OCR engines to improve accuracy on stylized game fonts. This covers games with no extractable text, and it gives translators the scene itself rather than audio alone.
- **Audio Classification:** Sorts game audio into seven categories so teams do not waste effort on the wrong files:
  - Dialogue (the priority for dubbing)
  - Non-dialogue vocals such as screams, breaths, and battle cries
  - Sound effects such as explosions, weapons, and doors
  - Ambience such as wind, rain, and room tone
  - Music
  - Whispers
  - Humming and melodic murmurs
- **Speech Transcription:** Audio is transcribed to text and automatically matched to the lines in the script. This links thousands of game audio files to their script lines without manual listening.
- **Stem Separation:** Original vocals are separated from music and ambience. Voice actors get a clean vocal reference to perform against, and the clean background (music, foley, environmental sound) is kept, so the new recording can replace the original voice without losing the soundtrack.
- **Video Ingest:** High-resolution gameplay videos are uploaded as unlisted videos to the team's own YouTube channel and played inside the platform. This avoids extra storage costs for the team.

### 3. Curation Studio

The hub where raw data is cleaned and prepared before translation. This is where lines are linked to characters, audio, and video, and where the story structure is built. It offers three working modes:

- **Wizard Mode:** A keyboard-driven, high-throughput screen for mass production. Source lines sit on one side, with video and waveform on the other, so large batches can be linked quickly.
- **Canvas Mode:** An infinite node canvas. Lines, audio, choices, scene sequences, map transitions, and video groups are connected visually, so branching stories and quest flows are modeled in space rather than in a flat list.
- **Demo Mode:** An interactive preview of how the prepared content will be experienced by a player, used to catch structural mistakes before translation begins.

**Source Explorer**
The main list of all source lines in a project.
- Browsing lists with more than 70,000 lines, as found in large RPGs
- Regex search, a folder-tree view of the source files, character filters, and bulk selection, to locate lines in large games
- Detail panels for lines (metadata), video segments, and audio waveforms, to inspect everything attached to a line in one place
- Bulk actions to assign characters, delete, or change the status of many lines at once

**Canvas Nodes**
- **Text node:** A line of dialogue. It glows and shows a progress bar while its line is playing.
- **Audio node:** A voice file. It syncs characters automatically and warns when several characters end up merged into one audio file, which usually needs splitting.
- **Sequence and video group nodes:** Containers that bundle connected lines or video segments into scenes.
- **Choice node:** Represents a player decision that branches the story.
- **Map node:** Represents transitions between locations in the game world.
- Automatic simplification of nodes when zoomed out, so large canvases stay responsive
- A searchable story map view for finding a scene anywhere in the graph
- Canvas export and import as JSON with a difference check, used for backups and for reviewing changes before they are applied

**Video & OCR Tools**
- **OCR overlay:** OCR boxes are shown on the video as it plays, either only those matched to script lines or everything detected in the video, so curators can see what was matched and what was missed.
- **Watermark eraser:** Draw a resizable area on the video and clear every OCR box inside it in one click. Channel logos, broadcaster marks, and UI elements would otherwise pollute the line list with fake dialogue.
- **Cinematic magnifier:** When a line is inspected, the video zooms into where its subtitle actually appears on screen, so small or distant text can be read comfortably.
- **Clean video player:** Hides YouTube's own title, logo, and controls and replaces them with Dublio's custom control bar, so teammates see only the game, not the platform around it.

**Timeline Editor**
Edits when each line starts and ends against the video.
- Overlapping lines are automatically distributed onto separate tracks, so nothing hides behind anything else.
- Line edges snap magnetically to other lines and to video scene cuts, to make precise timing easier.
- Dragging two consecutive OCR boxes together merges them with a ghost preview, closing the gap and unifying the text. This fixes subtitles that OCR split into pieces.

**Characters & Personas**
A character's different appearances or disguises are tied to one main character through sub-personas, for example "Bruce Wayne" with "Batman" and "Mysterious Man". In the Dubbing Studio, all personas are combined into a single voice actor's task list, so one actor naturally voices every identity of the same character.

**Speech-to-Text Studio**
A seven-tab workspace that connects transcribed audio to script lines:
- **Pending:** raw audio waiting to be processed
- **Processing:** recordings currently being transcribed on GPU workers
- **Processed:** successfully transcribed audio
- **Review:** matches that need a curator's confirmation
- **Other sounds:** effects and music that are not dialogue
- **Assign line:** unmatched audio that can be linked to a line by hand
- **Relink:** wrongly matched recordings that need to be reassigned

High-confidence matches are accepted automatically, medium-confidence ones go to the review tab, and low-confidence ones fall into the manual queue, so people only spend time on the uncertain cases.

### 4. Translation Studio

A team-driven workspace for translating large texts into any target language.

- **Four-column translation grid:** ID and character, source text, target translation, and status with actions. It is built for projects with 100,000+ lines.
- **Grouped lines:** Lines are grouped under sequence and video-group header strips, each with a completion percentage, so teams see which scenes are done and which are not.
- **Jump to selected:** After a search or an external link, the grid scrolls straight to the selected line.
- **Context drawer:** A resizable side drawer that remembers your preferred tab and gives the translator the full context of a line:
  1. **Video:** The exact video moment of the line, with a loop mode that replays the line's range until it is understood.
  2. **Audio:** The vocal reference played together with the background music, so tone, emphasis, and emotion can be felt before translating.
  3. **File content:** The raw source file, scrolled to the active line and highlighted. Translators see surrounding branches and neighboring dialogue and choose the right terminology instead of guessing.
- **Shared line management:** Standard lines like "Yes" or "System Error" repeat across hundreds of files. They get an "in X files" badge, sibling copies are listed, and one click jumps to any of them. With auto-sync enabled, saving a translation updates all identical lines in all files, so identical lines stay consistent and are not translated twice.
- **Master Catalog auto-fill:** Previously approved translations from the catalog can be applied to every matching empty line of a project in one click.
- **Community translation:** Any user can suggest an alternative translation for a line.
  - Votes carry weight according to the voter's experience level, so votes from experienced users count for more.
  - Users cannot vote on their own suggestions.
  - Votes update in the interface without waiting for the server.
  - The top suggestion automatically becomes the official translation. When another suggestion overtakes it, leadership changes hands, and the experience reward moves from the previous owner to the new leader.
- **Contribution analytics:** Ownership of a translation belongs to whoever actually wrote the words, not just whoever entered the line first. Changing only punctuation does not transfer ownership, and a breakdown (for example 70% one contributor, 30% another) is shown transparently. Meaningless edits do not transfer ownership.
- **Real-time multiplayer:** Teammates' avatars appear on the lines they are working on, a typing animation shows live activity, and unsaved drafts are visible to others. One click on a teammate jumps to their current line. This avoids two people translating the same line.
- **Support mode:** Admins can lock translation so only the approved core team can edit.
- **Keyboard shortcuts** for saving, approving and moving to the next line, moving between lines, and playing audio, so translators can work mostly from the keyboard.

### 5. Font Studio

Many game fonts do not contain the characters a target language needs, which turns text into empty boxes in-game. For example, a font may lack Turkish letters such as ç, ğ, ı, ö, ş, ü, or the accented characters of other languages. Font Studio fixes this.

- **Glyph injection:** Missing characters of the target language are added directly into the game's original font.
- **Automatic fitting:** Game fonts are stored as atlases with fixed-size slots. If a new glyph overflows its slot, it is scaled down step by step until it fits, so the atlas layout stays intact.
- **Fine-tuning controls:** Padding around each glyph, sharpness of edges, and font size can be adjusted for a result that matches the game's look.
- **Font DNA matrix:** A live, color-coded grid scans the whole character set of the font and shows at a glance which characters exist in the original, which were injected, which are critically missing, and which are simply not needed. It shows what still needs patching.
- **Export formats:** Unity TextMeshPro, with the coordinate conversion and cleanup Unity expects, and a simple JSON format with pixel coordinates for custom and web-based engines.
- **Atlas debugger:** A live viewer for inspecting the generated atlas at pixel level and checking character boxes with the mouse, with results savable to Google Drive.

### 6. Art & Texture Studio

Signs, maps, menu buttons, and UI graphics inside the game contain text that cannot be translated through a script. They are localized here.

- **Film strip:** A virtualized vertical strip for browsing thousands of textures.
- **Classification filters:** All, unclassified, text-based (contains text that needs translation), and purely graphical. This lets artists focus only on the textures that actually need work. Transparent images are shown over a checkerboard background, as in standard image editors.
- **Layer-based canvas:** The original texture stays locked at the bottom, while text boxes, brush strokes, and patches are added on top. Layers can be reordered by drag and drop, so the original art is not modified.
- **Zoom:** From a 25% overview up to 500% for precise alignment.
- **A/B comparison:** Switch between, or slide across, the original and localized texture to check that nothing was missed.
- **Regional crop and PNG download:** Cut out only the sign being translated and send it to external software such as Photoshop.
- **Version history and rollback:** Each save is versioned (v1.0, v1.1, …), and a mistaken edit can be undone by returning to an earlier version in one click.
- **Bink video hub:** In-game cutscenes are often stored in Bink (`.bk2`) format. A guided pipeline converts them to a normal video format, embeds subtitles, and converts them back.

### 7. Dubbing Studio

A browser-based recording studio; voice actors need only a microphone.

- **Lossless recording:** Microphone audio is captured unprocessed as 48 kHz, 16-bit PCM WAV, bypassing the browser's noise suppression and auto-gain.
- **Scene-synchronized recording:** A 3-2-1 countdown starts the recording, and the reference video and original audio play automatically at the same moment. Voice actors record while hearing the original performance in their headphones, matching its breathing and timing.
- **Live level metering:** A hardware-style VU meter with peak hold and a clipping warning shows if the voice is too quiet or too loud, and a live neon waveform is drawn while recording.
- **ADR timeline:** Shows the original vocal reference, the clean background music, and the new recording stacked together. Regions mark the original line's range, and zoom goes from an overview down to syllable-level alignment, to help match timing.
- **Overflow marking:** When a take runs longer than the original line, the overflow is flagged on the waveform, so actors can see where a take runs long.
- **In-place text editing:** If a translated sentence does not sit well in the mouth, the actor can revise it right from the text box above the waveform, instead of going back to the translators first.
- **Flexible line grouping:** Lines can be organized as sequences (consecutive speech in a scene), video sequences (lines tied to a clip), audio-file groups (sub-lines of the same file), or standalone lines. Sections can be merged into one recording, split apart, or added as custom sections for breaths and exclamations.
- **Unlimited take management:** Actors can record as many attempts as they want for one line, then star their best one as the preferred take. That take goes to the Mixing Studio as the official recording.
- **Loss prevention:** Recordings are saved locally first and then uploaded in the background, so a dropped connection does not lose a recording.
- **Voice actor discovery pool:** Team leaders can filter voice actors by gender and vocal tone (deep, young, cartoon, and so on) and shuffle through short random clips of their past recordings for quick casting.
- **Community voice acting:** In open projects, users can claim an unassigned character and leave a voice sample. The role is assigned when an admin approves.

### 8. Mixing Studio

A browser-based audio workstation where dubbed recordings are balanced against the game's original soundtrack.

- **Multi-channel timeline:** Unlimited channels for dialogue, group/foley, effects, music, and more, each with mute and solo, so any part of the mix can be isolated.
- **Two working modes:** Select mode (Q) for dragging, moving, and transferring blocks between channels, and blade mode (W) for splitting a block at any point.
- **Trim, resize, and fades:** Blocks can be lengthened, shortened, and given fade-in and fade-out.
- **Scrubbing and speed:** Dragging the playhead plays audio in real time like tape winding, and playback speed ranges from 0.5x to 2.0x for catching fine details.
- **Per-channel effects chain:** Each channel has a 4-band EQ (to shape tone), a compressor (to even out loud and quiet parts), a reverb (to place the voice in a room), and a delay (for echo), each with instant bypass and level indicators.
- **Stem channels:** One-click recall of the original music and vocal reference separated by stem separation, so the dub can be tested against the original soundtrack.
- **Master render:** The final mix is rendered as a 44.1 or 48 kHz WAV directly in the browser, without a server.
- **Timing sync:** Moving or trimming audio in the mixer updates the line's timing in the project, so everything stays consistent.
- **Retake requests:** If a recording is unsuitable (background noise, wrong emphasis), a Retake request with revision notes goes straight to the voice actor in the Dubbing Studio.
- **External DAW integration (Reaper, Pro Tools, Premiere):** For professionals who prefer desktop software.
  - Exports all audio and references in the folder layout of your choice: by character, by scene sequence, or flat.
  - On Chrome and Edge, files are written directly into a folder on the user's disk without zipping first.
  - Bulk re-import: hundreds of externally mixed files can be uploaded by choosing one folder, and file names are read to match each file to the correct line automatically.

### 9. Quality Control (QC) Studio

All components (text, audio, font, video) are tested together in the browser before they are packaged for the game.

- **Interactive GamePlayer:** Testers click through the dialogue like a player. Branching conversations work, and choice buttons appear when a choice node is reached, so every story branch can be tested. In auto-voice mode, the player waits for each line's audio to finish before advancing. The dubbed voice and background music play together in sync with the video.
- **Color-coded line markers:** The video progress bar marks lines with pins, so testers see review status at a glance: green for approved, red for revision requested, amber for not yet reviewed, and purple for the currently playing line.
- **Hierarchical issue reporting:** Testers pause the video and file a report in one of four categories, and the exact moment in the video is stamped automatically:
  - **Translation:** spelling errors, wrong meaning, text that does not fit
  - **Dubbing:** wrong pronunciation, lack of emotion, breath pops
  - **Mixing:** music drowning the voice, background noise, phase problems
  - **Curation:** wrong timecode, wrong character assignment
- **Waveform comparison:** The original and dubbed waveforms sit side by side, to check differences in amplitude and duration.
- **Approval and locking:** Lines without issues are approved individually or in bulk and locked as verified, so they cannot be changed by accident later.

### 10. Modding & Packaging

Once translation and dubbing are finished, assets must be put back into the game in the format its engine expects. Dublio does this through a configurable "Repack Recipe" for each project.

- **Text output modes:**
  - **Mirror:** recreates the original file structure exactly, in a target folder
  - **Single file:** merges all dialogue into one master JSON or binary dictionary
  - **Injection:** inserts only the changed lines into an existing game archive
- **Audio and folder rules:** Per-project choice of audio format (OGG, WAV, MP3) and bit rate, plus rules that decide where each sound goes in the game's folders and how it is named.
- **Build manifest:** A set of build instructions that the modding tools read, so packaging is repeatable.
- **Signed, purpose-built tools** for popular engines, triggered from the desktop client:
  - **Dublio Packer:** a general-purpose archive packer
  - **GameMakerDublio:** embeds text and fonts into GameMaker game files
  - **UnrealEngineDublioPacker & UECook:** archiving and cooking for Unreal Engine 5.6.1+ (`.pak`, `.utoc`, `.ucas`)
  - **FmodDublioPacker:** rebuilds FMOD sound banks with new audio
  - **TelltaleDublio:** integration with Telltale's engine archives
- **Incremental hotfix system:** If three audio files or one texture change, there is no need to rebuild a 10 GB package. Only the changed assets are converted and distributed as a small hotfix layer, which the player's client applies on top of the base package, so players download only the changed files.
- **Phase locks:** Discipline between project phases is enforced.
  - Completing a phase checks for unapproved lines or drafts and lists the people holding them, and the project manager can be nudged.
  - While a package is building, edits to lines and audio are frozen, so the package does not contain half-finished changes.
  - Users below a certain experience level are blocked from changing critical workshop settings or finished phases, protecting the project from accidents.

---

## Native Engine-Level Patcher

Traditional modding tools ask players to replace files manually, which is error-prone. Dublio includes a patcher that avoids this.

- **Direct asset replacement:** Replaces audio, texture, and font assets inside the game's own data files, so players do not need to touch game folders.
- **Engine support:** Supports Unity games, and integrates with packaging tools for Unreal Engine and other engines.
- **Large textures:** Processes big textures in a way that keeps memory use low, so patching works on ordinary computers.
- **Font safety:** Updates TextMeshPro font data without breaking the UI layout.

---

## Desktop Client (Rust / Tauri)

The desktop application is a state-aware patching client that delivers the finished localization to players with a single click.

- **One-click install:** When a player presses **"Install Patch"** on the website, the `dublio://install` protocol opens the client, which detects the installed game automatically and starts the installation.
- **Install options per language:** Full localization (text and dubbing), dubbing only, or subtitles only, so players choose how much of the game they want changed.
- **Three-layer installation:**
  1. **Base Layer:** the main patch package
  2. **Version Patches (Differential):** only the delta between the current version and the target version, so updates stay small
  3. **Hotfixes:** last-minute fixes such as a late retake, applied over everything else
- **Safe updates:** Before an update, the client compares against previous installation records, removes leftover files from older versions, and restores original game files from backups. This keeps the game directory consistent and lets players return to the original game.
- **Modding tools inside the studio:** Signed packaging tools can be triggered directly from the client.

---

## Teams, Projects & Community

### Teams
- **Team showcase:** A public profile with completed projects, the active roster, and achievement badges
- **Studio management tabs:**
  - **Identity:** logo, biography, and social links
  - **Recruitment:** an open listing board for roles the team is looking for, such as translators and voice actors
  - **Specialties:** how members are distributed across workshops
  - **Danger Zone:** ownership transfer and disbanding the team
- **Studio vitals dashboard:** In-team MVP ranking, the stage distribution of projects, and weekly word and audio production metrics, so leaders see who is carrying the project and where it is stuck
- **Activity center and project launch pad:** Teams start a new project in one click, and rosters are built through applications and invitations

### Roles & Permissions
Every action in the platform passes through a fine-grained permission check across three levels:
- **Platform roles** (Super Admin, Admin, Observer, User) decide access to platform administration
- **Team roles** (Leader, Admin, Member) decide what someone can do inside a team
- **Workshop specialties** decide which workshops a member works in: Voice Actor, Translator, Editor, Audio Engineer, Artist, Developer, QA Tester, Curator

Multiple teams can collaborate on the same project, sharing rights and visibility. For example, one team handles translation while another runs dubbing production.

### Projects
- **Three workflow types:**
  - **Full production:** curation → translation → dubbing → mixing → testing → release
  - **Dialogue only:** productions focused only on voice acting
  - **Text only:** classic subtitle patch projects with no dubbing
- **Stage progression:** Curation → Translation → Dubbing → Mixing → Final Review → Completed
- **Smart time estimator:** Remaining word count, the team's daily completion pace, and audio recording durations are analyzed to calculate a projected completion date that updates dynamically
- **Production hub:** A single cockpit summarizing the progress of all five workshops
- **Volunteer pool:** Independent users who want to help on a project can apply, and teams can find extra hands
- **Steam catalog:** Entering a game's AppID pulls its description, screenshots, genre tags, and release date in 11 languages, so catalog pages fill themselves
- **3D game cards:** Interactive cards with a perspective tilt that follows the mouse
- **Dynamic project covers:** Where the game's dialogue boxes appear on screen is marked on the cover through an ROI selector, with letterbox and pillarbox compensation, so covers reflect the real game layout
- **Community proposals and game requests:** Users open proposals for games they want localized and vote on them. The most requested games land on teams' radar.

### Real-Time Communication
- **Floating messaging dock:** Chat windows that can be dragged, resized, minimized, or turned into Picture-in-Picture, so chat stays available while working
- **Peer-to-peer group calls:** Voice and video calls directly between participants, with no media load on Dublio's servers
- **Screen sharing with system audio:** Translators and testers can broadcast the game screen or video player live to teammates, with sound
- **Adaptive quality:** Stream resolution adjusts automatically with network quality, from 480p to 1080p, with live latency and packet loss counters
- **Speaking indicator:** A green neon ring lights up around the avatar of whoever is speaking
- **Global presence:** Shows who is online and which workshop they are in
- **In-browser media compression:** Images and videos are converted to WebP or compressed MP4 before sending, and voice messages go out with a waveform preview

### Contextual Sticky Notes
- **Anchored to a page:** Notes are attached to a URL (even a specific line), so everyone who visits that page sees the same note, which makes them handy for handoffs and reminders
- **Editor:** Five color themes and a dual-pane Markdown editor with live preview
- **Permissions:** Owner, editor, and viewer levels
- **Global announcements:** Admins can pin notes that appear across the whole platform
- **Notes explorer:** A file-explorer-style tree view of all project notes, with folders, drag and drop, and media attachments

### Social Ecosystem
- **Artist portfolios:** Voice actors' demos, tone tags (such as "Monster" or "Young Lover"), roles played, and completed line counts, which also feed the casting pool
- **Reputation record:** A transparent record for every user. It lists commendations (completed projects, thanks from team leaders, translation awards) and penalties or warnings (abandoning projects without cause, sabotage attempts, inappropriate language), so teams know who they are inviting.
- **Community forums:** Four channels: general discussion and game worlds, translation and terminology, voice acting and dubbing craft, and technical support and modding guides
- **A hidden easter egg page:** A secret 3D space where the names of everyone who contributed to the platform float as stars

---

## Gamification

Dublio offers an experience system that rewards localization effort and does not tolerate abuse.

- **Ten levels,** each with its own title: Apprentice, Intern, Senior Translator/Voice Actor, Master Artist, Workshop Chief, Studio Architect, Grand Master, Legendary Localizer, Dublio Council Member, and Immortal Localization God. The progression is steep, so higher levels reflect sustained work.
- **Rewarded work:** Approved translations, taking the lead on a suggestion, starred dubbing takes, approved mixes, verified audio-to-text matches, approved textures, successfully tested font atlases, and confirmed QC reports each earn experience.
- **Penalties:** Sabotage, such as deleting approved lines or entering meaningless text, is met with heavy penalties, including direct loss of rank.
- **Level-based trust:** Higher levels carry more weight in translation votes, and Level 5 unlocks access to critical workshop settings.
- **Achievement badges:** Metallic 3D badges that react to mouse movement, in Bronze, Silver, Gold, Platinum, and Legendary tiers

---

## Administration & Statistics

### Ecosystem Dashboard
- Nightly snapshots of platform-wide totals: words translated, dubbing time recorded, fonts and textures produced, and active teams and users
- Interactive timeline charts showing how the platform has grown over the months
- An infrastructure health monitor covering Google Drive quotas, cloud storage volume, GPU worker response times, and database connection pool usage, to spot problems before users do

### Admin OS
An operating-system-style control center for platform administrators.
- **Master Data Wizard:** Adds new games to the Master Catalog, triggers Steam sync, and parses raw game files
- **Approvals:** Reviews projects opened by teams, publishes completed patches, and approves community proposals
- **User and team management:** Freezes accounts, assigns roles, and resolves ownership disputes between team members
- **Report inspector:** A drawer for reviewing user complaints and bug reports, with a diff lens that shows the earlier and later versions of a disputed translation or audio side by side
- **Client update hub:** Publishes new versions of the desktop client
- **Audit logs:** A second-by-second record of who changed what, when, and from which IP
- **Backups:** One-click manual database backup and restore from a previous backup
- **GPU queue control:** Shows worker status and lets admins route queued OCR and stem separation jobs manually
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
