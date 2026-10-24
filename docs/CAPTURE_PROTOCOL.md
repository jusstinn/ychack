# Capture Protocol: filming the venue for a high-quality 3D twin

> A field checklist. Print it or keep it open on the second phone.
> Goal: one capture that supports both (a) photoreal reconstruction (splat or mesh) and (b) a clean, editable twin (room shell + one 3D asset per *unique* object type).

## 0. Before you film (10 min)
- [ ] **Ask the organizers for permission** to scan a specific area. Builders may only use the **ground and first floors**.
- [ ] **Never capture** whiteboards, screens, laptops, documents, badges or people's faces. If people are in a shot, keep them small or moving out of frame; faces get blurred later. Don't publish raw footage.
- [ ] **Pick ONE zone** (one room or about a 10×10 m area). Ideal zone:
  - lots of **identical repeated items** (rows of the same chair or table), which shows off deduplication
  - 4–8 **distinct** items: sofa, plant, counter or bar, podium or stage, stools, a TV stand
  - clear walls, doors and exits
  - good even light
  - not mostly glass
- [ ] **Timing:**
  - Do a **5-minute pilot now** to test the pipeline.
  - Do the **full capture at the quietest window**: during the fireside chat (2:00–2:30 PM) if people gather elsewhere, dinner (6–7 PM), overnight (2–5 AM), or Sunday 6–8 AM. Remember: leave after midnight and you can't come back until 6 AM.
- [ ] **Roles:**
  - **Camera operator.**
  - **Spotter / logger:** keeps people out of frame, holds doors, logs every take, takes the measurements.

## 1. Kit
- Best phone available. An **iPhone Pro (LiDAR)** is ideal; any recent flagship works. **Wipe the lens.** Smudges cause haze.
- **Tape measure.** Ask organizers or facilities. Fallback: the iPhone Measure app (LiDAR models).
- **Scale markers:** 6–10 **3×3 in sticky notes** (76 mm, a standard size). Put them on the floor and on blank walls; they also give blank walls trackable texture. Alternatives: letter paper (8.5×11 in) or printed AprilTags.
- **White paper** for a white-balance reference shot.
- Optional: 360 camera (Insta360 etc.), gimbal, power bank.

## 2. Camera settings (do this once)
- **Main 1× lens only.** No zoom, no 0.5×, no switching mid-take.
  - iPhone: Settings › Camera › **Macro Control ON**, then keep the macro toggle **off** in the camera app so the phone doesn't auto-switch lenses up close.
- **Video:**
  - 4K, 30 fps (60 fps if bright). Standard stabilization; no Action or Cinematic mode.
  - Turn HDR video off if your phone allows it.
- **Lock exposure and focus:** long-press a mid-gray area until it shows **AE/AF LOCK**.
- **Better: Blackmagic Camera** (free iOS app).
  - Shutter **1/125–1/250 s** to avoid motion blur.
  - Fixed white balance, fixed focus, ISO as needed.
- **Photos:** full resolution; Night mode off, Live Photos off, never Portrait mode.
- Turn the **grid** on and keep the horizon level.

## 3. Shot list

### A. Pilot (5 min, do it NOW)
- 60-second slow loop around a small area, plus about 12 photos around **one chair**.
- Upload and run it through the pipeline **before** the full capture to catch problems early.

### B. Room shell and layout (most important for the reconstruction)
Walk slowly, about **0.3–0.5 m/s** (one step per second). **Translate while turning; never spin in place.**

| Take | What | Camera | Length* |
|---|---|---|---|
| **B1** | Perimeter loop facing **inward**, end exactly where you started (loop closure) | chest height (~1.4 m), level | 2–3 min |
| **B2** | Same loop | head height (~1.8 m), tilted **down ~30°** | 2–3 min |
| **B3** | Same loop | low (~0.7 m), tilted slightly **up** (sees table legs and chair bases) | 2–3 min |
| **B4** | 2–3 straight lines through the middle, slowly sweeping the camera side to side | chest height | 1–2 min |
| **B5** | Slow sweep of the **ceiling and light fixtures** | pointing up | 30–60 s |
| **B6** | **Each door or exit**: 3 photos (straight on, 45° left, 45° right) with the tape measure stretched across the width | — | photos |

\*Rule of thumb: about 1 minute per 15 m of perimeter. A 10×12 m room is roughly 2.5 minutes per loop.

**Also photograph, if present:**
- the posted **evacuation / fire map** (a free floor plan plus exits)
- the **max occupancy placard** (real capacity data)

### C. One capture per UNIQUE object type ("index only the things that aren't similar")
Do this for each distinct type, e.g. the chair, each table style, sofa, stool, plant, counter, podium, rolling whiteboard (blank side only). **Don't capture every copy.**

1. If it's movable (with permission), move **one clean example** into an open spot, then put it back.
2. **Orbit video:** a full slow circle at about 1–1.5 m distance.
   - Once at **mid height**.
   - Once **from above, looking down ~45°**.
   - **20–40 s per ring.**
   - *Or* photos: 3 rings (low, mid, high) × **12–16 photos**, plus 1 top-down.
3. **Front, side and back photos** at the object's center height. These are the best inputs for image-to-3D.
4. **Measure W × D × H**, with the tape visible in at least 1 photo, and log the numbers.
5. **2–3 texture close-ups** (fabric, wood, metal) about 30 cm away, straight on, in even light.
6. Log a **rough count** of how many of this type exist in the zone.

### D. "Hero" reference photos (for the demo and quality checks)
- **8–12 photos** from memorable spots: each corner at eye height, and the entrance view.
- Uses:
  - "real photo vs. 3D twin" comparison slider in the demo
  - checking render accuracy
  - helping the AI reason about the layout

### E. Lighting and environment
- [ ] A **360° photo from the room center**: 360 camera, or phone panorama passes (horizontal, plus one tilted up and one tilted down). This becomes the lighting environment, which makes renders look real.
- [ ] One **white paper** photo under the room's light (white balance).
- [ ] Note the light types (warm or cool), windows and sun direction, and the time.

### F. Measurements (ground truth for scale and an accuracy score)
Log about 10–15 numbers, with a photo of the tape for each:
- room length and width (2 places each)
- ceiling height
- door widths and heights
- 3 distances between fixed features (pillar to wall, etc.)
- table sizes
- chair footprint
- main aisle width

### G. Optional extras
- **Clean wall and floor texture set:** straight-on, overlapping photos of each wall and the floor, for clean, people-free textures.
- **LiDAR scan** (iPhone Pro) with a free scanner app (e.g. Polycam, 3D Scanner App, Scaniverse) for **ground-truth dimensions and fallback only**, not as the product.

## 4. Do / Don't

| ✅ Do | ❌ Don't |
|---|---|
| Move slowly and smoothly | Spin in place (it gives no depth information) |
| Overlap ~70–80% between consecutive views | Zoom or switch lenses |
| Stay 1–3 m from surfaces | Point at blank walls up close |
| Arc around corners | Film straight into bright windows |
| See every surface from ≥3 angles | Change exposure mid-take |
| Close every loop | Capture people, screens or whiteboards |
| Turn TVs and projectors off | Let messaging apps compress your files |
| Angle away from glare | |

## 5. File hygiene
- **Naming:** `Z1_B1_perimeter_chest.mov`, `Z1_C3_sofa_orbit_mid.mov`, `Z1_F_measurements.jpg`
- **Log sheet columns:** take ID | zone | pass | duration | notes / problems.
- **Upload originals** to the shared Drive folder (use original quality; AirDrop or the Drive app, not iMessage or WhatsApp).
- **Quick QC (5 min) right after filming:** scrub each video for blur, exposure jumps or people. **Redo bad passes immediately.**

## 6. Budget
- **Capture time:** about **30–45 min** for one zone with about 8–12 unique object types, plus measurements.
- **Data:** about 10–20 min of 4K video plus about 150–250 photos.
- The app extracts frames at **2–3 fps** and drops blurry ones, so roughly 1,500–3,000 frames are filtered down to the best few hundred.
