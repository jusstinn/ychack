# Emergent build guide: how the APIs plug in, and the prompt sequence

> Rule: **Emergent's agent writes 100% of the code.** We (and Claude) only write plain-language instructions.
> Calling external APIs (Claude, OpenAI, image-to-3D, Stripe) **from inside the app** is explicitly allowed.

## ▶ START HERE (Sat ~4:30 PM; ~19 h left)
**Keys to have ready:**
- **Anthropic** (or OpenAI) API key
- **fal.ai** API key
- **Stripe test keys**
- Replicate and World Labs are only needed for the optional prompts.

**Build order (test in the preview after each one):**
1. **Must (tonight):** P0 → P1 → P2 → P3 → P4 → P6 (incl. blueprint mode) → P7 → **P11 edit agent** → P7b kits → P8 → P9, then **P7c** (product link → 3D)
2. **Strongly recommended:** **P5L** (we already have a good LiDAR scan of the venue; it gives true-scale walls and exact positions). If it fights you, fall back to **P5** (AI-guessed placement + drag to fix).
3. **High wow, after the core loop works:** **P6b** (Blueprint → Clay → Real reveal), **P10** (photoreal showcase render)
4. **Stretch:** P12 (sightline/egress overlays)
5. **Skip unless spare time:** P5+, P5b
6. **Sunday morning:** **P13** (demo mode with cached results). Record the backup video by 9 AM; code freeze 10 AM.

If a prompt breaks something, describe the exact bug (what you clicked, saw, expected). Never paste code.

## How the integration works

```
 Phone photos/videos ──► [Frontend: React, built by Emergent]
                           • upload + in-browser frame extraction
                           • 3D editor (react-three-fiber)
                                   │  (our own backend endpoints)
                                   ▼
                         [Backend: FastAPI, built by Emergent]
                           • job pipeline (status saved in DB)
                           • API keys live HERE (env vars), never in the frontend
                                   │
              ┌────────────────────┼─────────────────────┐
              ▼                    ▼                     ▼
      Vision LLM API        Image-to-3D API        Image model API
   (Claude or OpenAI)     (provider TBD, being    (photoreal "showcase"
   object inventory,       verified) → GLB per     renders)
   dedupe, dimensions,     unique object type
   placement, NL edits
```

**What each piece does:**
- **Emergent writes all of it:** the frontend, the backend, the database, and the code that calls each API.
- **You tell Emergent:**
  - which API to use
  - the docs link
  - which model
  - what to send and what JSON to expect back
- **Emergent asks you for the API key** and stores it as a backend secret.
- **Every AI step is a backend job.** It saves its result to the database, and the UI shows progress, e.g. "Finding objects… 12 types found."
- **Results are cached**, so you never pay twice for the same images.
- **Download any generated files** (GLBs, images) into the app's own storage. Provider URLs can expire.

**How to prompt Emergent:**
- One feature per prompt. Every prompt ends with **"Done when:"** tests.
- Test in the preview after each prompt before sending the next.
- If it breaks, describe the bug precisely (what you clicked, what you saw, what you expected). Don't say "fix it."
- Never paste code into Emergent.
- Only one teammate prompts at a time.

## Verified stack (checked Sat Oct 24, ~1 PM; confirm exact model IDs in each provider's console)

| Slot | Primary | Fallback | Notes |
|---|---|---|---|
| `[LLM]` vision + reasoning | **Claude Opus 5.5** (`claude-opus-5-5`) with structured JSON outputs | **GPT-6 Astra** (`gpt-6-astra`); Claude Sonnet 5.5 for speed; Haiku 5.5 for cheap bulk tagging | Claude: up to 600 images per request (≤2000 px each when sending >20). Its boxes are approximate. No video input; send frames. **The Astra API outputs text only, not 3D files.** |
| `[SEG]` masks per object type | **fal SAM 3** (`fal-ai/sam-3/image-rle`), prompted with each type's name | Roboflow SAM 3; Grounding DINO (Replicate) | ~$0.005 per image. Raise `max_masks` (default 3) to count many chairs. |
| `[IMG3D]` asset per unique type | **fal SAM 3D Objects**: full frame + the object's box; handles partly hidden objects | **fal TRELLIS.2** at 1024 on a background-removed crop (best quality, PBR); Rodin v2.5 fast; Hunyuan3D 3.1 Rapid | SAM 3D ≈ $0.02, no PBR materials. TRELLIS.2 ≈ $0.30, ~17 s. Download every GLB into our storage. |
| `[GEO]` (Tier 1) camera poses / depth / points | **MapAnything** (Replicate): real-world scale | fal VGGT (scale is arbitrary; weights are non-commercial) | ~$0.022 and ~23 s per run, plus cold start. |
| `[SPLAT]` (optional "Reality" view) | **World Labs World API**: video or photos → splat (SPZ) + collider GLB | KIRI Engine API (faithful splat from ≤3-min video; 10 free scans; testing stage) | World Labs **invents unseen areas**: fine for visuals, never for capacity math. ~$1.28 per video, ~5 min. |
| `[IMGGEN]` showcase renders | **OpenAI `gpt-image-2.5-flare`** | Imagen 4 / Gemini through Emergent's universal key | ~$0.05 per high-quality image; up to ~2 min, so run it as a background job. |
| 3D in the browser | **@react-three/fiber v9 + drei v10**, plus **Spark 2.4** for splats | drei `<Splat>` (.splat only) | Emergent apps ship **React 19.2.x**, so r3f v9 / drei v10 are the matching versions. Spark needs three ≥ 0.180. **Pin versions.** |

**Estimated cost per venue:** about **$5–10** (10 types × ~$0.30 + segmentation + an optional splat + ~$1 of LLM calls). A good number for the pitch.

## Emergent platform constraints (verified). Put these in P0.
- **Requests are cut at about 125 s** (Cloudflare). Every AI or 3D step must be a **background job** with status saved in the database and the frontend polling for progress. Never wait for an API inside a single request.
- **Built-in storage is 5 GB and files can't be deleted; the container disk is wiped on restart.**
  - Store everything in the built-in storage or database, never on local disk.
  - Keep files small: frames ≤2000 px JPEG, compressed GLBs, **no raw videos** (the browser extracts frames).
- **No GPU** (up to 4 vCPU / 16 GB). Avoid heavy Python libraries (open3d, bpy); they may break once deployed.
- **API keys:**
  - Emergent has a **Universal LLM key**. Check whether it spends your Emergent credits and whether Opus 5.5 / Astra are listed. Using our **own Anthropic/OpenAI keys** for heavy vision calls is safer.
  - We also need **fal.ai** and **Replicate** keys, and optionally **World Labs** or **KIRI**.

---

## Prompt sequence (send one at a time)

### P0 — App skeleton and data model
> Build a web app called **VenueTwin** for small event venues. It turns photos and videos of a venue into an editable 3D model ("twin") that the venue uses to showcase its space and answer booking inquiries with layouts and quotes.
>
> **Data:**
> - **Venue** (name, address)
> - **Room** (name, floor outline as a polygon in meters, ceiling height, doors/exits with width and position, fixed features like columns/stage/bar, posted max occupancy)
> - **Photo** (file, kind: overview / object / measurement / hero / evacuation-map, optional "where I stood" point and facing direction on the floor plan)
> - **ObjectType** (name, category, width/depth/height in meters, count in room, source photo + crop box, 3D asset status and file, catalog fields: owned or partner, quantity owned, price, product link, seats, clearance)
> - **Instance** (object type, x, y, rotation, locked)
> - **Layout** (name, list of instances)
> - **Inquiry**
> - **Quote**
>
> **Pages:** Venues list, Venue setup (upload + room), Twin editor, Catalog, Inquiries, Quotes.
>
> **Style:** clean and modern, readable by a non-technical venue owner.
>
> **Architecture rules for the whole project:**
> - Every call to an AI or 3D service runs as a **background job**: save the job and its status in the database, return immediately, and the frontend polls for progress. Requests time out after ~2 minutes.
> - Store all files in the built-in storage, never on local disk (it's wiped on restart). Keep files small.
> - Keep API keys only in backend environment variables.
> - For 3D, use @react-three/fiber v9 and @react-three/drei v10 (matching our React 19), pinned.
>
> **Done when:** I can create a venue and a room, the pages navigate, and data persists after refresh.

### P1 — Capture upload with in-browser frame extraction
> On Venue setup, let me upload photos and videos (multiple files).
> - **Videos:** extract frames **in the browser** at 2 frames per second, downscaled to max 1600 px on the long side. Score each frame's sharpness and drop blurry or near-duplicate frames. Keep up to 60 frames per video. Upload only the kept frames, not the video.
> - **Each photo:** let me set its kind (overview / object / measurement / hero / evacuation-map).
> - **Overview photos:** optionally let me tap where I stood on the room's floor plan and drag the direction I faced.
> - Show a gallery grouped by kind.
>
> **Done when:** a 2-minute phone video becomes ~40–60 sharp frames in the gallery, and nothing is lost on refresh.

### P2 — Room shell (measurements + optional AI from the evacuation map)
> On Room, add a measurements form: room length, room width, ceiling height, and a list of doors (width, which wall, position along the wall, exit yes/no).
> - Build a 2D top-down editor for the floor outline: drag corners, snap walls to right angles, show dimensions in feet and meters.
> - Add a button **"Extract outline from evacuation map"**: send the evacuation-map photo to the `[LLM]` vision API. Ask for the outline of the selected room as a list of corner points (normalized 0–1) plus door/exit positions, as JSON. Scale it using my measured length, show it for me to confirm and edit, and save.
> - Show a **3D preview of the empty room**: floor, walls, ceiling, door openings, with good default materials.
>
> **Done when:** I can enter 3 measurements and get a correct-looking 3D room I can orbit around.

### P3 — AI object inventory with deduplication
> Add an **"Analyze objects"** backend job.
> - It sends the overview and object photos (in batches) to the `[LLM]` vision API and asks for **distinct object types only**. Identical chairs count as ONE type with a count, not separate objects.
> - **For each type, return JSON with:** name, category, short visual description (color, material, shape), estimated width/depth/height in meters, estimated count in the room, the best photo for that type, and a crop box around one clean example in that photo.
> - Use the API's **structured JSON output** with a fixed schema, so parsing never fails.
> - **Merge** results across batches (same type → one entry, using the descriptions and photos), and save them as ObjectTypes.
> - **Contact-sheet check:** combine the candidate crops into one numbered image grid, and ask the `[LLM]` once to group the numbers that show the same product model. Use that to confirm merges.
> - **Optional precise masks and counts:** for each type, call `[SEG]` (fal SAM 3) on the keyframes with the type's name as the text prompt and max masks set high. Save the masks, the best crop, and a per-frame count.
> - **Review screen:** cards with the crop image, name, dimensions, count and a confidence flag. Let me edit, merge two types, split one, or delete.
> - Show job progress live. Cache results so re-running doesn't re-pay for the same photos.
>
> **Done when:** my test photos produce a sensible list (e.g., "office chair ×48, round table ×6, sofa ×2") that I can correct.

### P4 — One 3D asset per object type
> For each ObjectType, create a 3D asset.
> - **Parametric (default for simple furniture):** tables (round/rectangular), counters, bars, stages/risers, dance floors, rugs, shelves, boxes. Ask the `[LLM]` to pick a template and fill its parameters from the photo and dimensions (top shape and size, thickness, leg style and count, colors, materials). The frontend builds the geometry from those parameters.
> - **Generated (complex items like chairs, sofas, plants):**
>   1. **Primary:** send the full photo plus the object's box to **fal SAM 3D Objects**. It handles partly hidden objects and returns a GLB.
>   2. **For the 2–3 most visible types:** also run **fal TRELLIS.2** (1024 resolution) on a background-removed crop. Render both results from the photo's viewpoint and let the `[LLM]` pick the closer match.
>   3. **Fallback:** Rodin fast, then a colored box.
>   4. Download every GLB into our storage (provider links expire).
>   5. Normalize it: rest it upright on the floor, center it, scale to the type's dimensions, keep it under ~20k triangles. If the model has no materials, apply the type's color and material.
> - Show status per type (queued / generating / ready / failed). While an asset isn't ready, show a correctly sized colored box, and swap in the real model when it arrives.
> - Run asset jobs in parallel. Allow "retry" and "use box instead."
>
> **Done when:** every type shows a 3D model or a clean placeholder at the right size.

### P5 — Place the objects in the room
> Add a **"Place objects"** job. Send the `[LLM]` the room outline and dimensions, the doors, the object types with counts and sizes, and the overview photos with where each was taken from. Ask for instance positions (x, y in meters, rotation) as JSON.
>
> **Then clean up automatically:**
> - keep everything inside the walls and out of door openings
> - snap rotations to the room's axes
> - push overlapping items apart
> - line up items that form rows or grids with even spacing
>
> **UI:** a top-down plan plus 3D. I can drag to fix items. Save the result as the layout **"As captured."**
>
> **Done when:** the room looks plausibly like my photos and nothing overlaps or floats.

### P5+ (Tier 1, optional) — Automatic measuring from the video
> Add an **"Auto-measure"** job. Send 30–60 keyframes to **MapAnything** on Replicate, which returns camera poses and a point cloud at real-world scale.
> - Fit the floor plane and the main walls, and suggest a room outline for me to accept.
> - For each object mask from P3, project its center into 3D to get its floor position, then group nearby detections of the same type into one instance.
> - Check the scale against my tape measurements and show the error ("±4%").
> - Then run the same cleanup as P5.

### P5L (Tier 1, RECOMMENDED: we already have the scan) — Import a LiDAR scan with phone positions
> Add **"Import LiDAR scan"** for zips exported by the **Stray Scanner** iPhone app. **Verified on our real export:**
> - `odometry.csv` has a header row and one row per video frame: `timestamp, frame, x, y, z, qx, qy, qz, qw, fx, fy, cx, cy` (+2 empty columns). Position is in meters and the quaternion is the camera-to-world rotation. **fx, fy, cx, cy are per-frame intrinsics at RGB resolution 1920×1440.**
> - `depth/NNNNNN.png`: 16-bit, **256×192**, millimeters. `confidence/NNNNNN.png`: values 0/1/2 (use only 2). **A few depth frames are missing; skip them.**
> - `rgb.mp4` has exactly one video frame per odometry row. Also `camera_matrix.csv` and `imu.csv` (unused).
>
> **The zip is large (~450 MB for a 3-minute scan), so do NOT upload it.** In the browser:
> 1. Open the zip locally (JSZip).
> 2. Upload only `odometry.csv`, every 10th depth + confidence PNG, and ~60 JPEG keyframes grabbed from `rgb.mp4` at the matching frame times.
> 3. Keep the total under ~40 MB, uploaded in chunks.
>
> Backend background job (numpy + Pillow only), processed in chunks:
> 1. For each used frame, scale the intrinsics to depth resolution (multiply fx, fy, cx, cy by 256/1920). Back-project high-confidence pixels with depth 0.25–5 m: `X = (u − cx)·d / fx`, `Y = (v − cy)·d / fy`, `Z = d`.
> 2. **IMPORTANT: Stray Scanner poses use OpenCV camera axes (x right, y down, z forward). Do NOT flip Y or Z.** World point = R(quaternion) · [X, Y, Z] + [x, y, z]. World up is +y.
> 3. Merge into a 3–5 cm voxel grid (average position and color per voxel). Color comes from the matching keyframe pixel (depth pixel × 7.5).
> 4. Floor = the strongest histogram peak of y below the median phone height. Ceiling = the strongest peak above it.
> 5. Align to the room: try yaw angles 0–90° in 0.25° steps and pick the one where wall points (1.6–2.6 m above the floor) give the sharpest x/z histograms. Main walls = the strongest histogram peaks. Propose a right-angle outline for me to accept.
> 6. For each object detected in the keyframes (P3/SAM 3), take the median 3D point of its mask to get its floor position. Group same-type detections within ~40 cm into one instance. **No depth (far away or glass)?** Cast the camera ray through the box and intersect it with the known floor (floor items), walls (wall items) or a hanging height (lamps ~2.4 m).
> 6b. **Free-space rule (important):**
>    - Place tables only where the scan shows tabletop points (~0.68–0.76 m above the floor). Snap runs to the room's axes.
>    - **Never place furniture where the scan clearly sees empty floor.** Those are aisles and open areas.
>    - Extend a run only into areas the scan didn't cover, and mark those tables "inferred" (shown faded until confirmed).
>    - Chairs go on the long sides of tables, never inside a table footprint.
> 7. Show the scan as an optional point-cloud layer in the 3D editor, aligned with the twin. Report the scale check against tape measurements.
>
> **Sanity check with our venue scan:** floor-to-ceiling ≈ **3.68 m**, phone height ≈ 1.45 m above the floor, main hall ≈ **19.8 × 13.8 m**, and table runs show up as 0.8 m-wide strips.
>
> **Done when:** our venue scan produces that outline and height, and placed tables land within ~20 cm of the scan.

### P5b (optional) — Photoreal "Reality" view
> Add a **"Create photoreal scan"** job. Send one continuous walkthrough video (≤3 min) to the **World Labs World API** (fallback: KIRI Engine API), and get a Gaussian splat (SPZ).
> - Render it with **Spark** (sparkjs) in the same react-three-fiber scene, aligned to the twin: I click 2 matching points in each and see the alignment error.
> - Toggle **Reality ↔ Twin**, with a crossfade.
> - Label the view "AI reconstruction: areas the camera didn't see may be invented." Never use it for capacity or egress math.

### P6 — High-quality 3D editor
> Make the Twin editor the centerpiece.
> - **Rendering:** react-three-fiber with drei. Make it look like a premium architectural visualization:
>   - one instanced mesh per object type (fast with hundreds of chairs)
>   - **studio-style environment lighting**: drei `Environment` built from `Lightformer` softboxes (a big overhead softbox, a long strip light on each side, a dim warm floor bounce). If I uploaded a 360° panorama of the venue, offer it as an alternative environment.
>   - **N8AO ambient occlusion** (@react-three/postprocessing)
>   - **Neutral tone mapping** (`NeutralToneMapping`)
>   - **soft shadows** (`AccumulativeShadows` with `RandomizedLight` while the scene is still, `ContactShadows` while dragging)
>   - compressed GLB loading (meshopt/Draco)
>   - quality tiers via `PerformanceMonitor`: shadows and AO off on weak devices, pixel ratio ≤ 1.5 on phones, pause rendering when the tab is hidden
> - **Editing:** select, move and rotate with a gizmo; snap to floor, walls and a grid; highlight collisions in red; multi-select; duplicate; "arrange in rows" tool; undo/redo.
> - **Views:** camera presets (top-down plan, entrance view, corner views, first-person walk).
> - **Performance:** must stay smooth on a laptop and a phone.
>
> - **Blueprint mode (2D plan editor, like DoBu's site-plan editor):**
>   - A top-down plan drawn like an architect's blueprint, with a visible **grid** (default 0.25 m squares; toggle 0.5 m or 1 ft).
>   - **Select:** click, shift-click, drag a box, or "select all of this type."
>   - **Move by dragging** (snaps to the grid), or **nudge with the arrow keys: one square per press** (Shift = 4 squares).
>   - **Rotate:** R = 90°, Q/E = 15°.
>   - Delete, duplicate (Cmd/Ctrl+D), undo/redo (Cmd/Ctrl+Z / Shift+Z).
>   - **Live metrics panel with before → after deltas** (DoBu-style): seats, tables and chairs used vs. owned, capacity vs. the occupancy limit, aisle/exit violations, quote total. Example: "Seats 148 → 154 (+6), Quote $2,310 → $2,370."
>   - The 2D plan and the 3D view edit the same scene data, so changes show in both instantly.
>
> **Done when:** I can rearrange the room smoothly (dragging in 3D, or nudging tables one square at a time in blueprint mode with the metrics updating live), and it looks good enough for a marketing screenshot.

### P6b — Signature reveal: Blueprint → Clay → Real (high wow, ~2–3 h)
> When a twin finishes processing, or when I press **"Reveal,"** play a 6–8 second animation:
> 1. **Blueprint:** the room and furniture appear as an architect's line drawing on a warm paper background, built from the models' crease and silhouette edges (drei `Edges`, crease angle about 35–40°). Hidden lines must not show; use a depth pre-pass so the GPU hides back edges. Lines draw in progressively like a draughtsman: long structural lines (walls, floor outline) first, furniture details after.
> 2. **Clay:** the drawing fills into a matte warm-white clay model with soft shadows and ambient occlusion.
> 3. **Real:** real materials spread across surfaces from a few seed points, like ink in wet paper (noise-driven dissolve with a thin darker rim at the edge of the spreading front), until everything has its final materials.
>
> Use one shared animation timeline so every material transitions in sync. Add a "replay" button.
>
> **Done when:** the reveal runs smoothly on a laptop, and hidden lines never appear.

### P7 — Catalog (owned inventory + partner products with links)
> Add a Catalog drawer in the editor.
> - **Owned:** the venue's own items (from ObjectTypes plus manual entries) with quantity owned and quantity used in this layout. Warn when used is more than owned.
> - **Partner:** rental/purchase products (name, image, dimensions, price, link, supplier). Seed 8 examples: chiavari chairs, lounge set, 8 ft banquet table, cocktail table, stage riser, dance floor panel, uplighting, draping.
> - I can drag catalog items into the scene. Dragging a partner item adds it to the quote.
>
> **Done when:** I can swap all office chairs for partner chiavari chairs and see the cost change.

### P7b — Decoration kits (seed for the demo)
> Add **kits**: named bundles of catalog items plus placement rules, applied in one undoable step and removable as a group.
> - **Wedding:**
>   - white floor-length linens on every table, floral centerpiece + 3 candles per table
>   - a **dance floor** (checkered panels) in the largest open area in front of the stage
>   - a **flower arch** on the stage
>   - **string lights** draped between the pendant lamps
>   - optional chair swap to gold chiavari chairs
> - **Theater:**
>   - chairs in rows facing the stage (uses P8's theater layout)
>   - black pleated **curtain backdrop** behind the stage
>   - two **spotlight stands**
>   - a red **aisle runner** down the center aisle
> - **Conference:** classroom rows, podium, two screens.
>
> Placement rules are relative to existing things (each table, the stage, the open floor area, walls). They must respect aisles and exits (P8 checks), and each kit's items are added to the quote.
>
> **Done when:** "Wedding" transforms our venue in one click, and "remove kit" restores it exactly.

### P7c — Product link → 3D model → into the room
> In the Catalog, add **"Add from link."** I paste a product URL (e.g., a rental chair or decor item).
> 1. Background job: fetch the page server-side. Ask the `[LLM]` to extract name, price, dimensions (W×D×H), and the main product image (use `og:image` when present). Return structured JSON.
> 2. Remove the background from the image and run image-to-3D (`[IMG3D]`: TRELLIS.2, fallback SAM 3D Objects). Normalize: upright, centered, **scaled to the extracted dimensions**, ≤20k triangles.
> 3. Save it as a **partner catalog item with the link and price**. I can drag it into the scene; the quote updates.
> 4. Fallback when a site blocks fetching: let me upload the product photo and type the dimensions.
> 5. Show progress: fetching → modeling → ready (~30–90 s).
>
> **Done when:** pasting a product link produces a correctly sized 3D item I can place, with its price in the quote.

### P8 — Inquiry → auto-layout with safety checks
> Add an **Inquiry** form: date, guest count, event type (banquet / theater / classroom / cocktail / wedding), must-haves (dance floor, stage, buffet, bar, head table, lounge).
>
> **"Generate layout" builds a layout in the real room that:**
> - stays under the room's max occupancy (entered by the venue; if missing, estimate from floor area and flag it as an estimate)
> - keeps exit doors clear
> - keeps aisles at least 44 inches wide
> - uses owned inventory first, then suggests partner items to fill gaps
> - places must-haves sensibly (stage at a short wall, dance floor in front of the stage, buffet near a door)
>
> **Show violations live.** When I drag something into an aisle or exit zone, it turns red and explains why.
>
> **Make generation reproducible:** each layout stores a random seed, so the same inquiry and seed always produce the same layout. A "Shuffle" button tries a new seed, and "Show 3 options" returns three seeds side by side.
>
> **Done when:** "150 guests, banquet, dance floor, stage" produces a clean layout in seconds, and "250 guests" in a 180-person room is flagged with alternatives.

### P9 — Quote and deposit
> Generate a quote from the layout: venue fee (venue sets day rates), owned items (venue sets prices), partner items. Give it a shareable quote page with a 3D preview of the layout. Add a **Stripe test-mode** deposit button. The deposit status shows on the venue dashboard.
>
> **Done when:** a quote link opens on a phone, the test deposit succeeds, and the dashboard shows it as paid.

### P10 — AI showcase render ("make it photoreal")
> Add a **"Showcase render"** button in the editor.
> 1. Capture the current 3D view as an image.
> 2. Send it to `[IMGGEN]` (OpenAI `gpt-image-2.5-flare`, as a background job, since it can take up to ~2 min) with an instruction like: "photorealistic interior photograph of this exact room and furniture layout, styled for [event type], natural lighting; keep the layout and proportions."
> 3. Save the results to a gallery labeled "AI visualization."
>
> **Done when:** one click turns a layout into a marketing-quality image.

### P11 (MUST: core feature) — The edit agent ("tell the room what to do")
> Add an **Edit chat** beside the 3D/blueprint view. Send the `[LLM]` the scene as compact JSON (types, instances with ids/positions/rotations, materials, room outline, open areas, catalog, kits) plus the user's request. It answers **only with tool calls**, which we execute as **one undoable transaction**:
> - `select(type?, area?, near?, ids?)` → ids
> - `move(ids, dx_squares, dz_squares)` / `move_to(ids, x, z)`
> - `rotate(ids, degrees)`
> - `set_material(type or ids, preset or color)`, e.g. tables black / walnut / linen-covered, chairs white / black / gold
> - `add_items(catalog_id, count, rule: rows | around_each_table | along_wall | on_stage | in_open_area)`
> - `apply_kit(name)` / `remove_kit(name)`
> - `generate_layout(setup, guests, must_haves)` (P8)
> - `remove(ids)`
> - `undo()`
>
> After executing, the agent replies in one sentence and the changed items flash. It never invents items that aren't in the catalog, and asks when a request is ambiguous.
> - **Demo commands that must work:** "make all the tables black," "add wedding decorations," "theater setup for 150 with a center aisle," "move the left table run one square right," "add 6 of the chairs from this link: <url>" (uses P7c).
>
> **Done when:** each demo command produces the right change in under ~10 s and can be undone.

### P12 (stretch) — Simulations
> Add overlay toggles:
> - **sightlines:** color each seat by how much of the stage it can see
> - **exit distance:** heatmap of walking distance to the nearest exit
> - **capacity per setup:** summary card

### P13 — Demo hardening
> Add **demo mode**: load the saved real-venue run and replay cached AI results instantly, so the live demo never waits on an API. Add friendly loading states and error messages everywhere. Make sure every page works on a phone.
