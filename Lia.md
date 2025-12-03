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

---

## 🚀 Next Steps
- [ ] **Blender Bake:** Perform the rotation fix in Blender to resolve the "Moonwalking" issue once and for all.
- [ ] **Roaming Logic:** Re-enable the State Machine (Idle -> Roaming -> Hiding) once animations are stable.
- [ ] **UI Polish:** Add a "Settings" context menu to change AI models or Hitbox size.

---

## 💡 Lessons Learned
> "Wayland security is strict. If you want a window to control itself (move/resize), you often have to fall back to X11 (XWayland) drivers."

> "Retargeting animations at runtime is messy. It is almost always better to fix the animation data in a DCC tool (Blender) than to hack it in the game engine."
