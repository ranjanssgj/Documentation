# Lia - AI Desktop Companion (Linux/Godot)

**[Lia](https://github.com/ranjanssgj/Lia)** is an intelligent, offline desktop companion built for Linux (Arch/Hyprland). Unlike standard desktop pets, Lia uses a local Large Language Model (Llama 3.2 via Ollama) to converse, react to emotions, and roam the screen freely while maintaining transparency and click-through capabilities.

## 🛠 Tech Stack
* **Engine:** Godot 4.3 (Compatibility Mode / OpenGL)
* **Language:** GDScript
* **AI Backend:** Ollama (Llama 3.2:1b)
* **Format:** VRM (Anime Character) & Mixamo Animations
* **OS Target:** Linux (Arch + Hyprland), compatible with X11/Wayland.

---

## 🏗 Architecture Layers
The project is built on four distinct layers:
1.  **The Stage:** Transparent window management & OS integration.
2.  **The Body:** VRM rendering & Smart Hitboxes.
3.  **The Brain:** HTTP Bridge to local AI & Emotion Parsing.
4.  **The Behavior:** Finite State Machine (Idle, Roaming, Hiding).

---

## 📜 Development Log & Challenges

### Phase 1: The Transparent Stage
**Goal:** Create a borderless, transparent window that floats over the wallpaper but under active windows.

* **Challenge 1: The Black Screen.**
    * *Issue:* Godot rendered a black skybox instead of transparency.
    * *Fix:* Enabled **Transparent Background** (Viewport) AND **Transparent** (Window) in Project Settings. Removed the default `WorldEnvironment` node.
* **Challenge 2: The Hyprland Conflict.**
    * *Issue:* Hyprland (Tiling WM) tried to tile the character window, adding borders and shadows.
    * *Fix:* Added custom Window Rules in `hyprland.conf`:
      ```bash
      # Force floating, remove decorations, and pin to all workspaces
      windowrulev2 = float,title:^(Lia)$
      windowrulev2 = noborder,title:^(Lia)$
      windowrulev2 = pin,title:^(Lia)$
      ```
* **Challenge 3: The Movement Lock.**
    * *Issue:* Error `Embedded window can't be moved`. Wayland security prevents windows from moving themselves via script.
    * *Fix:* Forced the app to run via XWayland using the launch flag:
      ```bash
      --display-driver x11
      ```

### Phase 2: The Smart Hitbox (Interaction)
**Goal:** Allow the user to click the character (to chat) but click *through* the empty air (to use the desktop).

* **The "Ghost Paradox":** If `mouse_passthrough` is ON, we can't click Lia. If OFF, we can't click the desktop behind her.
* **The Solution:** Implemented a **Dynamic Polygon Hitbox**.
    * We project the 3D character's position to 2D screen coordinates.
    * We draw a rectangle (`Rect2`) around her body + the Chat Bubble.
    * We send this polygon to the OS:
      ```gdscript
      DisplayServer.window_set_mouse_passthrough(polygon, get_window().get_window_id())
      ```
    * *Result:* Only the pixels occupied by Lia block the mouse; everything else is transparent air.

### Phase 3: The Brain (Local AI Integration)
**Goal:** Enable offline conversation with personality.

* **Implementation:** Used `HTTPRequest` node to send JSON payloads to `localhost:11434` (Ollama).
* **The System Prompt:** Configured the AI to reply with "Action Tags" to drive animations.
    * *Prompt Strategy:* "If happy, start with `[HAPPY]`. If leaving, use `[HIDE]`."
* **The Parser:** Wrote a string parser in GDScript that extracts these tags, strips them from the visible text, and triggers the `AnimationPlayer`.

### Phase 4: Animation & Behavior (Current Focus)
**Goal:** A "State Machine" that allows Lia to roam, hide, and idle organically.

* **The Asset Crisis (Mixamo vs. VRM):**
    * *Issue 1: Z-Flip (Moonwalking).* Mixamo uses +Z Forward, Godot uses -Z Forward. The character walked backwards.
    * *Issue 2: A-Pose vs. T-Pose.* VRM models rest with arms down (A-Pose), Mixamo assumes arms out (T-Pose). Result: Lia walked with hands stuck in the air.
* **Attempted Fixes (Failed):**
    * Rotating Root Node (Flipped the mesh, but animation data remained reversed).
    * Godot Retargeting "Rest Fixer" (Inconsistent results).
    * Manual Bone Rotation Script (Complex quaternion math).
* **The Definitive Solution (In Progress):** The Blender Pipeline.
    1. Import FBX to Blender.
    2. Rotate 180° on Z-Axis.
    3. Apply Transform (`Ctrl+A`).
    4. Export as GLB.
    * This "bakes" the correct orientation permanently before Godot even sees the file.
 
# Phase 5: The Asset Pipeline Fix (Blender Integration)

**Goal:** Fix the "Moonwalking" and "Broken Wrists" issues permanently by standardizing the character rig.

## The Workflow:

- **Clean:** Imported original VRM to Blender -> Cleaned Mesh -> Exported as FBX.

- **Rig & Animate:** Uploaded FBX to Mixamo -> Applied Auto-Rigger -> Downloaded Animations (Walk, Dance, Salute, Sit).

- **Combine:** Merged all animations into a single .glb file using an online glTF combiner.

## The Code Update:

- Refactored `main.gd` to target the new node hierarchy (`Lia/Node/Armature/Skeleton3D`) instead of the old `GeneralSkeleton`.

- **Result:** Animations now play with correct orientation (No more Z-Flip) and correct rest pose (No more T-Pose arms).

# Phase 6: Intelligence & Memory (The "Brain" Upgrade)

**Goal:** Move from "Random Behavior" to "Context-Aware Behavior."

## Feature 1: Persistent Memory (`memory_manager.gd`)

- **Implementation:** Created a JSON database at `user://lia_memory.json`.

- **Function:** Stores User Name, Mood, and Current Task.

- **Effect:** On startup, Lia checks this file. If the name is "User" (default), she enters Setup Mode to ask for your name. If known, she greets you personally.

## Feature 2: Activity Monitoring (`activity_monitor.gd`)

- **Logic:** A background node tracks mouse velocity and idle time.

- **States:**
  - **Working:** Low mouse movement for >5s → Triggers Salute.
  - **Bored:** No movement for >30s → Triggers Yawn or Sit.
  - **Active:** High movement → Wakes up to Idle.

## Feature 3: Priority Tag Parsing

- **Issue:** The AI would sometimes output multiple tags (e.g., `[WAVE] [WORK]`) causing conflicting animations.

- **Fix:** Implemented a Priority Dictionary in `chat_controller.gd`. `[WORK]` is checked before `[WAVE]`, ensuring she Salutes instead of waving goodbye when the user says "I'm busy."

# Phase 7: Cross-Platform & Architecture Experiments

**Goal:** Solve the Linux Wayland movement restriction.

## Experiment A: Windows Port

- **Action:** Exported project with Embed PCK enabled.

- **Result:** Confirmed movement logic works perfectly on Windows 10/11. This isolated the "Stuck Window" bug specifically to Arch Linux/Wayland security policies.

## Experiment B: Fullscreen Overlay (The "Glass Pane")

- **Concept:** Instead of moving the window, make the window fullscreen/static and move the character inside it.

- **Implementation:** Switched Camera to Orthogonal mode and Window to Fullscreen Transparent.

- **Outcome:** Technically functional but introduced "Input Trapping" issues (blocking clicks on the desktop).

- **Decision:** Shelved for later. Current focus returned to standard window mode with DisplayServer movement logic.


---
# Major Changes

**Date:** January 06, 2026
**Version:** 0.4.0 (The Voice Update)
**Status:** Functional Hybrid System (Godot + Python)

---

### 🚀 Summary
This update marks the transition from a "Local-Only Text Chatbot" to a **"Hybrid Cloud Voice Assistant."** We moved the heavy processing (Audio & Intelligence) to a dedicated Python backend and cloud APIs, resulting in near-instant response times and expressive voice capabilities.

---

### ✨ New Features

#### 1. Hybrid Audio Architecture (The "Split Brain")
* **Concept:** Godot handles the body (Visuals/Animation), while Python handles the senses (Ears/Mouth).
* **Communication:** Implemented a local UDP network for fast data transfer.
    * `Port 4242`: User Voice Text (Python → Godot)
    * `Port 4243`: AI Response Text (Godot → Python)

#### 2. Cloud Intelligence Upgrade (Groq)
* **Switched from Ollama to Groq:**
    * **Model:** `llama-3.3-70b-versatile` (Primary) with fallback to `llama-3.1-8b-instant`.
    * **Performance:** Inference time reduced from ~4.0s (Local) to ~0.4s (Cloud).
    * **Proactive Triggers:** Fixed logic to ensure the AI responds to system events (e.g., "User is bored") by formatting them as pseudo-user prompts.

#### 3. Voice I/O Implementation
* **Ears (STT):** Integrated **Groq Whisper (Large-v3)**.
    * Replaced `SpeechRecognition` (Google) for faster, higher-accuracy transcription.
* **Mouth (TTS):** Integrated **Microsoft Edge TTS** (`edge-tts`).
    * Provides high-quality neural voices (`en-US-AnaNeural`) without API costs.
* **"Walkie-Talkie" Logic:**
    * Implemented a **Microphone Lock** to prevent Lia from hearing her own voice and entering an infinite loop.
    * Added a **Cooldown Timer (2s)** after user speech to prevent accidental double-sending.

#### 4. Interaction Refinements
* **Typing Mode Protocol:**
    * When the user clicks the Chat Input Box, Godot sends a `__SYS__PAUSE_MIC` command to Python.
    * When the user submits text, Godot sends `__SYS__RESUME_MIC`.
    * *Result:* Eliminates conflict between voice input and keyboard input.
* **Latent Masking:**
    * Added immediate "Thinking..." UI feedback and animations to mask the 1-2 second latency caused by TTS audio generation.

---

### 🛠 Technical Changes

#### **Python Backend (`audio_server.py`)**
* **Cross-Platform Playback:**
    * **Windows:** Uses `pydub` + `simpleaudio`.
    * **Linux (Arch):** Uses `subprocess` to call `ffplay` externally, fixing the `SIGSEGV` crash caused by internal audio libraries.
* **Security:** Integrated `python-dotenv` to load API keys from a local `.env` file, keeping secrets out of source code.

#### **Godot Controller (`chat_controller.gd`)**
* Refactored `parse_and_animate` to support concurrent animation and voice triggers.
* Added `_send_sys_command` to handle low-level control of the Python backend.

---

### 🐛 Bug Fixes

| ID | Issue | Solution |
| :--- | :--- | :--- |
| **B-01** | `SIGSEGV` Crash on Linux | Replaced `pydub.play` with external `ffplay` subprocess call. |
| **B-02** | Infinite Voice Loop | Added `is_ai_speaking` flag to sleep the microphone while TTS is active. |
| **B-03** | Text Input Ignored | Fixed signal connection logic in `_ready` and `_on_text_submitted`. |
| **B-04** | API Rate Limit (429) | Implemented automatic fallback from Llama-70b to Llama-8b upon exhaustion. |
| **B-05** | Proactive Triggers Ignored | Reformatted system events as `[SYSTEM EVENT: ...]` user messages to force AI acknowledgement. |

---

### 🔮 Next Steps
* [ ] **Visuals:** Overhaul UI (Chat Bubbles, Fonts, Transparency).
* [ ] **Model:** Improve texture quality and shading for Lia.
* [ ] **Deployment:** Finalize `LifecycleManager` to bundle the Python executable inside the Godot Export.

## 💡 Lessons Learned
> "Wayland security is strict. If you want a window to control itself (move/resize), you often have to fall back to X11 (XWayland) drivers."
> "Retargeting animations at runtime is messy. It is almost always better to fix the animation data in a DCC tool (Blender) than to hack it in the game engine."
