# 3D Twin Pipeline: design and optimization brainstorm (v1)

> Status: v1 draft, Sat 12:45 PM. Will be merged with an independent expert review and a verified tool/API list (v2).
> Rules reminder: everything is built inside Emergent. **Hosted APIs called from the app are fine. Do not write or host our own GPU code.**

## Decision (Sat ~1 PM): proof of concept, not 1:1
The demo renders must **look high quality**, but the twin doesn't have to match the room exactly.

**Build path:**
- the **compositional twin**: room shell from about 6 measurements plus the evacuation map, AI object inventory, one asset per unique type, approximate placement, a premium-looking renderer
- the **AI showcase render** for photoreal marketing images

**Skipped unless time allows:** photo-based geometry reconstruction (structure from motion, Gaussian splats) and exact 3D lifting.

Step-by-step build prompts: `docs/EMERGENT_PROMPTS.md`. Lite capture: top of `docs/CAPTURE_PROTOCOL.md`.

## v2: merged from an independent 3D-vision expert review (Sat ~1:30 PM)

### Architecture
- **Main build: the compositional twin.** A JSON scene graph is the source of truth: shell, object types, instances (with SKU/price). Edits, simulations and auto-layout only need the graph.
- **Optional "splat skin" (Reality view):** a photoreal Gaussian splat of the room behind the graph. Movable furniture is erased from the splat and replaced by meshes.
  - Fast source: a Scaniverse phone scan exported as SPZ/PLY (ask Emergent staff if OK).
  - In-app source: KIRI Engine API.
  - Browser renderer: **Spark** (three.js).
- **Skip generative "world" models** (e.g. World Labs Marble) for the twin. Invented geometry breaks capacity and egress math. Fine for pure marketing images only.

### New tricks worth adopting
1. **Dedupe cascade:** category → dimension gates → embedding pre-groups → **one VLM call on a numbered "contact sheet"** of crops ("group these by product model") → a human merge/split gallery with counts. Cheap and accurate.
2. **Hero canonicalization:** pick the sharpest, least-occluded still (blur score × mask area). An image-edit model cleans it onto a white background before image-to-3D. Heavily occluded in-room crops go to **SAM 3D Objects** (fal), which is built for objects in real scenes.
3. **Best-of-N assets:** run 2–3 image-to-3D providers in parallel (e.g. TRELLIS.2 / Hunyuan3D / Tripo). Render each from the photo's viewpoint and let the VLM pick the best. The reviewer estimates ~$1 per type; verify pricing.
4. **Normalize every asset:** stable resting pose, yaw matched to the footprint, scaled to measured dimensions, **≤20k triangles**, compressed textures.
5. **Orientation priors:** chairs face tables, rows face the stage. This fixes 180° flips without extra AI calls.
6. **One simulation grid** (10 cm occupancy grid, run in a web worker, <50 ms per drag):
   - distance transform → **aisle widths**
   - fast marching → **egress distance plus a flow field**
   - ray-marching → **sightlines**

   This is what makes live red/green feedback while dragging possible.
7. **Browser as GPU:** render plan views, crops and lighting cubemaps client-side. The backend does CPU geometry and API calls only.
8. **Async job graph:** content-hash cache, provider fallbacks, model routing (cheap model for bulk tagging, strong model for judging and merging). Proxies hot-swap as assets arrive.
9. **If using the splat:**
   - Use the splat's points to find floor and walls.
   - Do **plan-view segmentation**: render a top-down view and run SAM 3 text prompts ("chair," "table") on tiles to get footprints in meters.
   - **Erase** splats inside instance boxes and lay an inpainted **floor decal** underneath, so moved chairs reveal carpet, not ghosts.
   - Use a splat-rendered cubemap as the PBR lighting environment.

### Simulations, ranked by wow-per-hour (for the 2-minute video)

| # | Feature | What it does | Est. time |
|---|---|---|---|
| 1 | Seat POV + sightlines | Click a chair to drop to seated eye height. Seats are colored by how much of the stage they can see. | ~1.5 h |
| 2 | Live egress while dragging | Shove a table into an aisle and it turns red as the heatmap and capacity update. | ~2 h |
| 3 | Inquiry → auto-layout | The LLM writes only constraint JSON; a GA or lattice packer solves it, with the convergence animated. Gives 3 options with capacity, egress pass/fail and quote. **Justin's strength.** | ~4 h |
| 4 | Evacuation agents | ~150 agents follow the flow field; shows bottlenecks and time-to-clear before and after an edit. | ~2 h on top of #2 |
| 5 | Capacity by setup | Uses occupant-load factors of **7 / 15 / 5 net ft² per person** (chairs only / tables and chairs / standing), capped by exit width at **~0.2 in per person**. Verify against current IBC; the venue's posted occupancy overrides; label as an estimate. | ~1 h |
| 6 | Quote / bill of materials | Owned vs. rented counts, totals, links. | ~1 h |

### Failure modes and mitigations
- **Scale and axis errors:** one convention everywhere (**meters, Y-up, floor at y = 0**). Two-point calibration ("this distance is 1.83 m") that shows the residual error.
- **Wrong dedupe:** dimension gates, low-confidence flags, one-click merge/split.
- **Floating or intersecting objects:** floor snap, stacking on support surfaces, overlap resolution.
- **Slow APIs:** job queue, parallelism, timeouts, fallbacks. Never block the UI.
- **Huge files:** app backends cap request size, so upload directly to storage, extract frames in the browser, and compress splats (SPZ) and meshes.
- **Mobile performance:** level-of-detail budgets, a mesh-only plan mode, and testing on the oldest phone tonight.
- **Demo breakage** (regenerations, credits, Wi-Fi):
  - Save checkpoints often and serve the pre-baked scene statically.
  - **Record the backup demo video by 9 AM Sunday. Code freeze at 10 AM.**

### References found by the reviewer (pricing and limits pending verification)
- [Spark 2.0](https://sparkjs.dev/docs/new-features-2.0/) · [Spark splat editing](https://sparkjs.dev/docs/splat-editing/)
- [fal SAM 3](https://fal.ai/models/fal-ai/sam-3/image/api) · [fal SAM 3D Objects](https://fal.ai/models/fal-ai/sam-3/3d-objects/api)
- [KIRI 3DGS video upload API](http://docs.kiriengine.app/3dgs-scan/video-upload/) · [Scaniverse](https://radiancefields.com/platforms/scaniverse)
- [VGGT on Replicate](https://dev.to/aimodels-fyi/a-beginners-guide-to-the-vggt-1b-depth-model-by-vufinder-on-replicate-2fai)

## Render look: techniques from Justin's Loop projects

> **Rule:** no Loop code or files go into Emergent. They were written before kickoff and outside Emergent.
> We reuse the **techniques**, described in plain language, and Emergent's agent rebuilds them.
> Loop is a separate company, so keep its private material out of this public project.

What makes the Loop site (three.js) look premium, and the react-three-fiber / drei equivalent Emergent can use:

| Loop technique | Why it looks good | Ask Emergent for |
|---|---|---|
| **"Photographer's studio" environment for reflections:** warm-grey sweep, big overhead softbox, a long strip light on each side, dim floor bounce | Long soft highlights on metal and gloss instead of generic reflections | drei `<Environment>` built from `<Lightformer>` softboxes and strips (or the venue's 360° panorama) |
| **N8AO ambient occlusion** | Contact darkening that grounds objects | `N8AO` effect from @react-three/postprocessing |
| **Khronos PBR Neutral tone mapping** | True colors without the washed-out look | `NeutralToneMapping` |
| **Soft PCF shadows** | Grounded, not floating | Soft shadows; for a static scene, drei `<AccumulativeShadows>` + `<RandomizedLight>` (baked, very soft), or `<ContactShadows>` |
| **Decimated, meshopt-compressed GLBs with crease-split normals** | Crisp edges, small files, fast load | Meshopt/Draco compression; ≤20k triangles per asset |
| **Quality tiers:** shadows and AO toggled by device, pause rendering when the tab is hidden | Smooth on phones | drei `<PerformanceMonitor>` / `<AdaptiveDpr>`, DPR ≤ 1.5 on mobile |
| **"Drafting board comes to life":** crease and silhouette lines with hidden lines removed, drawn like a draughtsman (long lines first), then the model **turns to clay**, then **materials bleed in from seed points like ink in wet paper** | A signature reveal people remember | The **Blueprint → Clay → Real** reveal (prompt P6b). Simplified build: drei `<Edges>` with a crease angle, depth pre-pass so hidden lines are hidden by the GPU, animated stroke drawing, then a noise-dissolve shader from clay to textured materials. |
| **Splat background + meshes composited with correct depth** (`loop_render`): static room as a splat, movable things as meshes; fiducial alignment with a residual report | Photoreal room, editable objects | The optional splat "Reality" view (Spark renders splats inside three.js). Align with 2+ known points and show the residual. |
| **Seeded, reproducible placement** (`loop_scene`): same scene state + seed gives the same placement | Repeatable demo, comparable runs | Auto-layout takes a seed: same inquiry + seed gives the same layout. "Shuffle" picks a new seed. |

**Why the reveal fits:** the venue first appears as an architect's line drawing (a nod to the DoBu "blueprint"), becomes a clay massing model, then real materials bleed in. It's the perfect 5-second opener for the demo video and makes the AI processing feel intentional, not like a loading spinner.

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
