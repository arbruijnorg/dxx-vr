# agents.md — OpenVR Virtual Reality Enablement (OpenGL)

This document defines the implementation plan, architecture, and agent responsibilities for adding **Virtual Reality (VR)** to an existing application using **OpenVR** and **OpenGL**.

## Goals

1. **Virtual Cinema Mode (2D UI/Video on Curved Screen)**
   - All non-3D content (**cutscenes, menus, briefings, UI screens, videos**) must render into an **OpenGL texture** (FBO).
   - That texture is displayed on a **curved polygon/screen** in a VR “virtual cinema” environment.

2. **In-Game Stereoscopic 3D Mode**
   - In-game 3D content is rendered **stereoscopically**:
     - Render **Left Eye** into a left eye texture (FBO).
     - Render **Right Eye** into a right eye texture (FBO).
   - Submit those eye textures to OpenVR for presentation on the headset.

3. **Unified Rendering Strategy**
   - Everything renders into textures first.
   - The VR compositor receives eye textures.
   - Virtual Cinema renders as a world-space curved mesh with the 2D texture applied.

## Non-Goals

- Not implementing OpenXR (OpenVR only).
- Not rewriting the whole renderer; we wrap/extend existing pipelines.
- Not requiring a specific engine framework (works with custom OpenGL).

---

## High-Level Architecture

### Modes
- **MODE_CINEMA**: 2D content rendered into a texture → mapped onto a curved screen mesh in VR.
- **MODE_VR_3D**: in-game scene rendered into stereo textures → submitted to OpenVR.
- Optional: **MODE_FLAT** (non-VR) preserved for desktop fallback/debug.

### Core Components
- `VrSystem`  
  - Initializes OpenVR.
  - Obtains recommended render target size per eye.
  - Polls events, updates poses.
  - Submits frames to the compositor.

- `VrRenderer`
  - Owns GL resources: eye FBOs/textures, cinema FBO/texture, mirror output (optional).
  - Produces:
    - Eye textures for stereo mode.
    - Cinema texture for virtual cinema.
  - Renders curved cinema mesh (world-space) using a standard shader.

- `SceneRenderer` (existing)
  - Renders the 3D world given view/projection matrices.
  - Must support “render to FBO” instead of directly to the window.

- `UiRenderer` / `VideoRenderer` (existing)
  - Must support “render to FBO” producing a RGBA texture.

---

## Data Flow

### Virtual Cinema (Menus, Cutscenes, Briefings)
1. Bind `cinemaFbo`
2. Render the 2D screen/cutscene into `cinemaTex` (RGBA)
3. For VR presentation:
   - Render the VR environment once per eye (or minimal skybox/room)
   - Render a **curved mesh** with `cinemaTex` applied, positioned in front of the user
   - Output into `leftEyeFbo` / `rightEyeFbo`
4. Submit `leftEyeTex` and `rightEyeTex` to OpenVR

### In-Game Stereo 3D
1. Update headset/controller poses
2. For each eye:
   - Compute `view = HMDPose * eyeOffset * gameCamera`
   - Compute `projection` from OpenVR
   - Bind `eyeFbo`
   - Render 3D scene into `eyeTex`
3. Submit both eye textures to OpenVR

---

## Rendering Requirements

### OpenGL Texture/FBO Standard
- Use sRGB correctly if you already render with sRGB.
- Preferred formats:
  - Color: `GL_RGBA8` or `GL_SRGB8_ALPHA8` (if using sRGB pipeline)
  - Depth: `GL_DEPTH24_STENCIL8` (or separate depth)
- Each render target is an FBO with:
  - color texture attachment
  - depth (and optionally stencil) attachment

### Resolution & Size
- Stereo eye target size should come from:
  - `vr::IVRSystem::GetRecommendedRenderTargetSize(&w,&h)`
- Cinema texture size can be:
  - Match window UI resolution (e.g., 1920x1080) OR
  - Scale to a performance/quality target (configurable)

---

## Coordinate Systems & Matrices

OpenVR provides:
- Eye-to-head transforms (`GetEyeToHeadTransform`)
- Projection matrices (`GetProjectionMatrix`)
- Tracked device poses (`WaitGetPoses`)

You must define consistent conventions:
- Right-handed vs left-handed
- Row-major vs column-major
- OpenGL clip space expectations

**Rule:** Convert OpenVR matrices into your engine’s math types once, and keep everything consistent.

---

## Curved Cinema Screen

### Mesh
- A curved “cylinder segment” is recommended:
  - Radius: 1.5–3.0 meters (configurable)
  - Arc: 60–110 degrees (configurable)
  - Height: sized for aspect ratio (16:9 typical)
- UV mapping:
  - U across arc
  - V across height

### Placement
- Screen anchored in front of the user:
  - Center at ~2.0m forward, slightly downward (optional)
- Optionally allow:
  - Recenter / reposition screen
  - Scale up/down
  - Curve amount

### Rendering
- Render screen in linear space; apply gamma correctly.
- Use a simple unlit shader (no lighting), optionally with subtle vignette/borders.

---

## OpenVR Frame Submission

- Submit each eye as a `vr::Texture_t`.
- The texture must remain valid until submission finishes (do not delete/overwrite early).
- Call `vr::VRCompositor()->Submit(vr::Eye_Left, &tex)` and similarly for right.
- Call `vr::VRCompositor()->WaitGetPoses(...)` each frame to sync with compositor timing.

**Optional:** Mirror output to desktop window:
- Blit one eye texture or a composited view to the main window for spectators/debug.

---

## Input & Interaction

- Virtual Cinema mode may need:
  - Laser pointer cursor onto curved screen
  - Controller ray → UV mapping → UI mouse events
- In-game mode:
  - Standard VR controller binding (OpenVR actions recommended)
  - Head pose drives camera orientation (or hybrid with mouse/gamepad)

---

## Performance Targets

- Avoid re-rendering cinema texture if static UI:
  - Update only when UI changes
- Use MSAA carefully:
  - MSAA in eye FBO improves edges but costs performance
- Keep GPU copies minimal:
  - Render directly into textures used for Submit
- Use late-latching style updates where possible:
  - Get poses as late as possible before rendering

---

## Agent Responsibilities

### Agent: `vr-platform`
**Owns:** OpenVR init/shutdown, device events, poses, action bindings  
**Deliverables:**
- `VrSystem::init()`, `shutdown()`
- `VrSystem::pollEvents()`
- `VrSystem::waitGetPoses()`
- Eye projection + eye offset transforms
- Controller input abstraction (minimal initially, extend later)

### Agent: `vr-render-targets`
**Owns:** FBO/texture creation, resizing, sRGB correctness  
**Deliverables:**
- Create/recreate:
  - `leftEyeFbo + leftEyeTex + depth`
  - `rightEyeFbo + rightEyeTex + depth`
  - `cinemaFbo + cinemaTex + depth` (depth optional for 2D)
- Resize policy:
  - Eye size from OpenVR recommended size
  - Cinema size from config

### Agent: `cinema-mesh`
**Owns:** curved mesh generation and rendering  
**Deliverables:**
- Curved cylinder segment mesh generator
- UV-correct mesh
- Simple textured shader (unlit)
- Screen placement controls (radius/arc/scale)

### Agent: `engine-integration`
**Owns:** integrating VR matrices with existing renderer  
**Deliverables:**
- Modify `SceneRenderer` to render into arbitrary FBO with provided view/proj
- Modify `UiRenderer` / cutscene/briefing to render into `cinemaFbo`
- Mode switching (cinema vs stereo)

### Agent: `mirror-output`
**Owns:** desktop mirror window output (optional but recommended)  
**Deliverables:**
- Blit left eye or combined view to desktop swapchain/window

---

## Minimal Implementation Checklist (Milestone Order)

### Milestone 1 — Headset Frame Submit
- [ ] Initialize OpenVR
- [ ] Create eye FBOs at recommended size
- [ ] Render solid color into each eye FBO
- [ ] Submit to compositor (left/right)
- [ ] Mirror one eye to desktop window

### Milestone 2 — Stereo In-Game
- [ ] Integrate poses + per-eye view/proj
- [ ] Render 3D world into each eye FBO
- [ ] Submit to compositor
- [ ] Add recenter

### Milestone 3 — Virtual Cinema
- [ ] Create cinema FBO/texture
- [ ] Render menus/briefings/cutscenes into cinema texture
- [ ] Generate curved mesh
- [ ] Render curved mesh into each eye FBO (VR environment)
- [ ] Submit to compositor

### Milestone 4 — Interaction & Polish
- [ ] Controller ray → UI cursor mapping on curved screen
- [ ] Configurable screen curve, distance, size
- [ ] Performance optimizations (dirty redraw for cinema)
- [ ] Optional: simple theater environment

---

## Implementation Notes (Practical)

### Mode Switch Contract
- UI code renders to `cinemaTex` whenever:
  - UI changes (buttons hover, selection changes)
  - video/cutscene frame updates
- VR render loop decides what to show:
  - If `MODE_CINEMA`, draw curved screen in each eye
  - If `MODE_VR_3D`, draw world scene per eye

### Texture Ownership
- The “source” renderers never draw directly to OpenVR.
- They only draw to textures.
- The VR pipeline composes and submits.

---

## Suggested File Layout

- `src/vr/VrSystem.h/.cpp`
- `src/vr/VrRenderer.h/.cpp`
- `src/vr/VrMath.h/.cpp` (matrix conversions)
- `src/vr/CinemaScreen.h/.cpp` (mesh + shader)
- `src/render/RenderTargets.h/.cpp` (FBO helpers)
- `src/app/VrModeController.h/.cpp` (mode switching & config)

---

## Configuration Keys (Example)

- `vr.enabled = true/false`
- `vr.mode = cinema|stereo`
- `vr.cinema.width = 1920`
- `vr.cinema.height = 1080`
- `vr.cinema.radius_m = 2.0`
- `vr.cinema.arc_deg = 90`
- `vr.cinema.height_m = 1.2`
- `vr.mirror.enabled = true/false`
- `vr.msaa = 0|2|4|8`

---

## Definition of Done

- Menus/briefings/cutscenes render to an OpenGL texture and display on a curved polygon in VR.
- In-game 3D renders stereoscopically to per-eye textures and submits to OpenVR.
- Desktop mirror shows output for debugging.
- Smooth head tracking with correct projection per eye.
- No direct “draw UI to headset” shortcuts: everything routes through textures.
