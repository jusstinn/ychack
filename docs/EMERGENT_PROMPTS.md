# Emergent build guide: how the APIs plug in, and the prompt sequence

> Rule: **Emergent's agent writes 100% of the code.** We (and Claude) only write plain-language instructions.
> Calling external APIs (Claude, OpenAI, image-to-3D, Stripe) **from inside the app** is explicitly allowed.

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

**Fill in before starting:**
- `[LLM]`: Claude or OpenAI, plus the model. Suggested: a strong vision model for inventory and placement, a cheap one for small tasks.
- `[IMG3D]`: the image-to-3D provider. Pending verification; pick one with a simple HTTP API that returns a GLB.
- `[IMGGEN]`: the image-generation model for showcase renders.

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
> - **Merge** results across batches (same type → one entry, using the descriptions and photos), and save them as ObjectTypes.
> - **Review screen:** cards with the crop image, name, dimensions, count and a confidence flag. Let me edit, merge two types, split one, or delete.
> - Show job progress live. Cache results so re-running doesn't re-pay for the same photos.
>
> **Done when:** my test photos produce a sensible list (e.g., "office chair ×48, round table ×6, sofa ×2") that I can correct.

### P4 — One 3D asset per object type
> For each ObjectType, create a 3D asset.
> - **Parametric (default for simple furniture):** tables (round/rectangular), counters, bars, stages/risers, dance floors, rugs, shelves, boxes. Ask the `[LLM]` to pick a template and fill its parameters from the photo and dimensions (top shape and size, thickness, leg style and count, colors, materials). The frontend builds the geometry from those parameters.
> - **Generated (complex items like chairs, sofas, plants):**
>   1. Remove the background from the crop.
>   2. Send it to the `[IMG3D]` image-to-3D API and get a GLB.
>   3. Download it into our storage.
>   4. Center it on the floor and scale it to the type's dimensions.
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

### P6 — High-quality 3D editor
> Make the Twin editor the centerpiece.
> - **Rendering:** react-three-fiber, one instanced mesh per object type (fast with hundreds of chairs), soft shadows, ambient occlusion, good tone mapping, and an environment map (use my uploaded 360° panorama if present, otherwise a tasteful default). Make it look like a premium architectural visualization.
> - **Editing:** select, move and rotate with a gizmo; snap to floor, walls and a grid; highlight collisions in red; multi-select; duplicate; "arrange in rows" tool; undo/redo.
> - **Views:** camera presets (top-down plan, entrance view, corner views, first-person walk).
> - **Performance:** must stay smooth on a laptop and a phone.
>
> **Done when:** I can rearrange the room smoothly, and it looks good enough for a marketing screenshot.

### P7 — Catalog (owned inventory + partner products with links)
> Add a Catalog drawer in the editor.
> - **Owned:** the venue's own items (from ObjectTypes plus manual entries) with quantity owned and quantity used in this layout. Warn when used is more than owned.
> - **Partner:** rental/purchase products (name, image, dimensions, price, link, supplier). Seed 8 examples: chiavari chairs, lounge set, 8 ft banquet table, cocktail table, stage riser, dance floor panel, uplighting, draping.
> - I can drag catalog items into the scene. Dragging a partner item adds it to the quote.
>
> **Done when:** I can swap all office chairs for partner chiavari chairs and see the cost change.

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
> **Done when:** "150 guests, banquet, dance floor, stage" produces a clean layout in seconds, and "250 guests" in a 180-person room is flagged with alternatives.

### P9 — Quote and deposit
> Generate a quote from the layout: venue fee (venue sets day rates), owned items (venue sets prices), partner items. Give it a shareable quote page with a 3D preview of the layout. Add a **Stripe test-mode** deposit button. The deposit status shows on the venue dashboard.
>
> **Done when:** a quote link opens on a phone, the test deposit succeeds, and the dashboard shows it as paid.

### P10 — AI showcase render ("make it photoreal")
> Add a **"Showcase render"** button in the editor.
> 1. Capture the current 3D view as an image.
> 2. Send it to `[IMGGEN]` with an instruction like: "photorealistic interior photograph of this exact room and furniture layout, styled for [event type], natural lighting; keep the layout and proportions."
> 3. Save the results to a gallery labeled "AI visualization."
>
> **Done when:** one click turns a layout into a marketing-quality image.

### P11 (stretch) — Natural-language editing
> Add a chat panel. The `[LLM]` receives the scene as JSON and may only respond with edit operations: add, move, remove, arrange in rows, swap type, generate layout. We apply them with undo. Example: "Set up a banquet for 120 with a dance floor by the stage and move the bar to the east wall."

### P12 (stretch) — Simulations
> Add overlay toggles:
> - **sightlines:** color each seat by how much of the stage it can see
> - **exit distance:** heatmap of walking distance to the nearest exit
> - **capacity per setup:** summary card

### P13 — Demo hardening
> Add **demo mode**: load the saved real-venue run and replay cached AI results instantly, so the live demo never waits on an API. Add friendly loading states and error messages everywhere. Make sure every page works on a phone.
