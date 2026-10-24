# 3D Twin Pipeline: design and optimization brainstorm (v1)

> Status: v1 draft, Sat 12:45 PM. Will be merged with an independent expert review and a verified tool/API list (v2).
> Rules reminder: everything is built inside Emergent. **Hosted APIs called from the app are fine. Do not write or host our own GPU code.**

## Core principle
**The scene graph is the product; the pixels are a skin.**

The twin is a small JSON file holding three things:
- **Room shell:** floor polygon, walls, openings.
- **Object types:** one 3D asset each.
- **Instances:** a type plus a position and rotation.

That makes it:
- **Editable:** move, duplicate, swap.
- **Tiny:** kilobytes, so it loads instantly.
- **LLM-readable:** enables natural-language edits and auto-layout.
- **Fast to render:** one instanced mesh per type.

Photoreal pixels (a Gaussian splat or textured mesh) are an optional "Reality" view layered on top.

## Data model
- **RoomShell:**
  - floor polygon (meters) and wall heights
  - openings: doors (exit flag), windows, with dimensions
  - fixed features: columns, stage, bar, built-ins
  - floor and wall materials or textures
  - scale source and confidence
- **ObjectType:**
  - name, category, dimensions W×D×H
  - asset: parametric params, GLB URL, or primitive
  - canonical crop or thumbnail, materials
  - **catalog fields:** source (`owned` or `partner`), product link, price, seats, clearance, stackable, quantity owned
- **Instance:** type ID, x, y, z, yaw, (scale), confidence, source frames, locked flag.
- **Layout:** a named set of instances: "as captured," "Banquet 150," "Theater 200," "Cocktail 250."
- **Capture:** frames, (poses), measurements, hero photos, environment map.

## Pipeline stages
Each stage has a **Fast option (build first)** and an **Ambitious option (upgrade)**.

### 1. Ingest
- **Fast:** upload photos and short videos. **Extract frames in the browser** (video → canvas at 2–3 fps, downscaled to ~1600–2048 px), score each frame for blur (variance of the Laplacian), drop near-duplicates, and upload only keyframes. This avoids multi-GB uploads and server-side video decoding.
- **Also:** a measurements form (tape numbers) and a hero-photo tag.

### 2. Room shell
- **Fast:**
  1. Read the floor outline from the **evacuation map photo** or a sketch with the VLM.
  2. Correct it with the tape measurements.
  3. Clean it up in a 2D polygon editor that snaps to right angles (shapely-style geometry; DoBu skills).
  4. Add heights from the tape.
- **Ambitious:**
  1. Get camera poses and a point cloud from a hosted feed-forward 3D model (VGGT / MASt3R class) or SfM.
  2. Fit floor and wall planes with RANSAC.
  3. Project to the floor, build an occupancy grid, take the contour, and simplify the polygon.
  4. Snap walls to right angles (Manhattan world).
  5. Set scale from the sticky-note markers and the tape numbers.
- **Textures (big quality win):** for each wall plane:
  - warp the best frames onto the plane (homography) and **median-blend** them, which **erases people and transient objects**
  - inpaint the leftover holes
  - use a tileable floor texture made from close-ups

### 3. Object discovery and dedupe (index only the unique things)
1. **VLM inventory:** send about 20–40 diverse keyframes to the VLM. It returns JSON listing each distinct object type with attributes (color, material, shape), a count estimate and the frames it appears in.
2. **Grounding:** get 2D boxes per type per keyframe, either from the VLM or from a hosted open-vocabulary detector plus SAM masks.
3. **Dedupe:**
   - cluster the crops by image embedding plus size
   - VLM pairwise check: "same model of chair? yes/no"
   - result: a list of ObjectTypes
4. **Canonical crop:** the sharpest, largest, most frontal view per type. That becomes the asset input and the catalog thumbnail.
5. **Human-in-the-loop:** low-confidence merges get one-click confirm or split.

### 4. One asset per type (cost scales with the number of types, not instances)
- **A. Parametric (default for most venue furniture):** tables (round or rect), counters, risers or stages, shelves, rugs, dance floor, boxes. The LLM fills in template parameters (top shape and size, thickness, leg style and count, colors), and we generate the geometry in three.js. The result has exact dimensions, perfectly clean geometry, is fully editable and costs nothing.
- **B. Retrieval:** match common items to existing free 3D models (licensed), then scale them to the measured dimensions.
- **C. Generation:** hosted image-to-3D (TRELLIS / Hunyuan3D / SAM 3D / Tripo / Meshy class), working from the canonical crop and multiple views. Then normalize: center it, ground it, **scale to the tape dimensions**, decimate, compress.
- **Fallback:** a textured box or billboard proxy.
- **Progressive UX:** proxies appear instantly and are swapped for real assets as each job finishes (shows a live "assets streaming in" timeline).

### 5. Placement
- **Fast (semi-auto):** the VLM proposes rough positions using the hero photos plus the room shell. The user drags and corrects instances on a top-down plan, with snapping. Expect VLM metric placement to be rough; the correction UI is the plan.
- **Ambitious:**
  1. **Lift each 2D detection into 3D** using camera poses plus metric depth (hosted depth model or a point map).
  2. Take each instance's 3D points, project them to the floor, and fit an oriented box (yaw only).
  3. Merge the same instance across views (same class plus 3D proximity).
- **Regularization solver (the optimization showcase):** turns noisy placements into a clean as-built layout. It minimizes a combined cost that:
  - keeps each item near where it was observed
  - keeps items on the floor, and wall-hugging items (sofas, shelves) against the wall
  - snaps rotation to the room's axes
  - **detects rows and grids and evens out their spacing**
  - pushes overlapping boxes apart
  - optionally rewards symmetry

  Solve with coordinate descent or annealing. A few hundred objects take milliseconds.

### 6. Rendering (high quality)
- three.js / react-three-fiber:
  - PBR materials
  - **environment map from the 360° photo**
  - ACES/AgX tone mapping
  - ambient occlusion (SSAO/N8AO)
  - soft and contact shadows
  - baked AO on the shell
  - **InstancedMesh per type**
  - Draco/Meshopt + KTX2 compression
  - LOD
- **Reality view (optional):** a Gaussian splat of the as-captured room, rendered in the browser (Spark / gsplat-class library) and aligned to the twin. Toggle or crossfade **Reality ↔ Editable twin**.
- **"Match view" slider:** render from a hero photo's viewpoint and show the real photo and the twin side by side. Strong demo moment.

### 7. Edit and simulate
- **Editing:** gizmos; snapping to grid, walls and other objects; collisions; multi-select; duplicate or array ("fill rows"); lock fixed features; undo/redo; save layouts.
- **Catalog drawer:** owned items with **used / available counts**, and partner items with **links and prices**. Drag into the scene. The quote updates live.
- **Natural-language editing:** an LLM with tool calls works on the JSON scene graph. Examples: "banquet for 120 with a dance floor by the stage," "move the bar to the east wall," "swap all chairs for chiavari."
- **Auto-layout optimizer (DoBu-style):** packing with constraints (room polygon, fixed features, exit clearances, aisle widths, occupancy cap, inventory counts), then local search or a genetic algorithm to maximize seats, sightlines and flow.
- **Simulations:**
  - **Capacity** per setup from code occupant-load factors (verify IBC Table 1004.5; the venue's posted occupancy overrides).
  - **Egress:** exit clear zones, minimum aisle widths, and a heatmap of travel distance to the nearest exit.
  - **Sightlines:** raycast from each seat's eye point to the stage, giving a heatmap of % obstructed.
  - **Crowd flow (stretch):** simple agent simulation for entry, buffet lines and evacuation, showing bottlenecks and evacuation time.
- **Outputs:**
  - **auto venue spec sheet:** area, ceiling height, capacity per setup, inventory, exits
  - share link
  - quote and deposit
  - (stretch) **USDZ/GLB export for AR**: judges scan a QR code and see the venue on their table

## Optimization ideas (brainstorm)
1. **Dedupe types:** cost and time scale with unique types, not instances. 60 chairs cost 1 asset.
2. **Keyframe selection** by blur, parallax and coverage: 10× fewer API calls.
3. **Browser-side frame extraction:** no huge uploads, no server ffmpeg.
4. **Coarse-to-fine:** low-res frames for discovery, high-res crops only for assets.
5. **Cache everything** by frame hash: never pay twice for the same API call.
6. **Parallel asset jobs** per type, run from a queue.
7. **Parametric-first:** most venue furniture is boxes, cylinders and legs, so it's free and exact.
8. **Known-product scale anchors:** a recognized product with published dimensions works as a free ruler.
9. **Sticky-note markers:** cheap metric scale plus texture on blank walls.
10. **Manhattan snapping plus row/grid regularization:** messy becomes clean.
11. **Median plane texturing** removes people from walls and floors.
12. **Instancing, LOD and compression:** 60 fps on laptops and phones.
13. **LLM-native scene JSON:** natural-language edits, diffs, versioning, cheap to send to models.
14. **Render, compare, refine:** render from a hero photo's viewpoint, ask the VLM "what's different from the real photo?", apply fixes (missing object, wrong color, wrong count) and repeat. This is the Astra-style agent loop.
15. **Confidence UI:** low-confidence objects are highlighted for one-click confirmation (also covers "handles a mistake").
16. **Accuracy report:** measured vs. model distances, giving a "±X%" score you can show judges.
17. **Bake once, edit many:** heavy processing happens once per venue; daily edits are cheap JSON operations.
18. **Setup templates** (banquet, theater, classroom, cocktail): instant layouts.
19. **Inventory-aware optimization:** owned counts are hard limits; partner items fill gaps at cost, which drives the upsell.
20. **Model routing:** a cheap model for per-frame tagging, a strong model for merging and planning.
21. **Structured JSON outputs** (schemas) for every LLM call: robust parsing.
22. **Deterministic demo:** cache one golden run of the real venue so the demo can't break.

## Where the GPU work happens
- **Phone:** captures only.
- **Emergent backend:** CPU only. It orchestrates jobs, stores data and calls APIs.
- **Hosted APIs (their GPUs):** VLMs (OpenAI, Claude), detection and segmentation, depth and poses, image-to-3D, splat training.
- **Viewer's browser:** renders with WebGL/WebGPU on the device GPU. Normal laptops and phones are fine.
- **Do not** deploy our own GPU servers (Modal, RunPod, Lambda). That means writing code outside Emergent, which breaks the rules.

## Build tiers (submission-safe first)
- **Tier 0, guaranteed demo (Sat afternoon to evening):**
  - upload and keyframes
  - room shell from the evac map plus tape numbers
  - VLM inventory and dedupe into types
  - parametric assets plus image-to-3D for 2–3 hero types
  - semi-auto placement
  - editable 3D with instancing
  - catalog drawer with links and prices
  - egress and capacity checks
  - quote and deposit
- **Tier 1, auto placement (overnight):** poses and depth to 3D boxes, plus the regularization solver.
- **Tier 2, photoreal (overnight, if an API works):** the Reality ↔ Twin toggle and the match-view slider.
- **Tier 3, simulations and AI:** sightline and egress heatmaps, natural-language editing, render-compare-refine, AR export.

## Open questions (being verified)
- Is the **GPT-6 Astra API** available, at what price and latency, and what does it output?
- Which hosted **image-to-3D**, **segmentation** and **depth/pose** endpoints are live today, and what do they cost?
- Is there a **room reconstruction / splat API** we can call from the app (World Labs, KIRI, Luma, Polycam...)?
- **Emergent limits:** upload size, file storage for GLB and splat files, long-running jobs, which Python packages can be installed.
