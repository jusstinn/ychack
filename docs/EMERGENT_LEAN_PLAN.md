# Lean Emergent plan v2: 6 prompts, demo-first, agentive editing

> **Send these in order, one at a time.** After each one, test in the preview, then send the next.
> **Design:** follow `docs/DESIGN.md` and the mockups in `assets/design/` (upload the PNGs when a prompt mentions them).
> **Assets:** upload from `assets/` when L1 asks: the two scenes (`venue_twin.json`, `theater_twin.json`), catalog, kit recipes, 49 3D models and media.
> **Rules:** Emergent writes all code. Never paste code. API keys only in the backend.

**Decision (recap):**
- We reconstruct venues ourselves and ship them as **scene files (data) plus 3D models (design assets)**.
- The **app imports scenes**. Everything interactive is built here and is **agentive**: Claude decides placements live from the scene and the catalog. Nothing is precomputed.
- **Transparency line** for the video and submission: "We reconstructed this venue with our scan pipeline; the app imports the twin and everything you see is live."
- **Ask the Emergent desk:** "Can we import 3D venue scenes we generated with our own scan pipeline, plus our own 3D model files and a hero video, as data and design assets?"

**Credits:** about 2,000 + the 1,000 refill. Rough guess: L1 160 · L2 330 · L3 350 · L4 400 · L5 300 · fixes 400 = ~1,940.
- **Check usage after L1.**
- **Roll back instead of looping.**
- **Keep ~100 for deployment.**

---

## L1 — App skeleton, scenes, import
> Build a web app called **VenueTwin**: an AI venue planner. Venues get a 3D "twin" of their space, rearrange and decorate it by talking to an AI, and see seats and price update live.
>
> **Data:**
> - **Scene:** name, room outline (polygon in meters), ceiling height, room `style` (wall, floor and ceiling colors), optional `cameras` (named viewpoints), optional `inventory` (quantities this venue owns, overriding the catalog), and an item list.
> - **Item:** id, type, x, z, rotation in radians, optional `y` (elevation in meters), optional w/d/h (size), optional `label` (e.g. seat "E10"), an `inferred` flag, an optional `group` tag (e.g. "wedding"), and an optional `material` override.
> - **CatalogItem:** type, name, 3D model file, size in meters [w, d, h], seats, unit price, quantity owned, source owned/partner, category, `mount` (floor / wall / ceiling), and `platform` (true for stages, risers and steps that other items stand on).
> - **KitRecipe:** name, label, plain-language recipe, list of steps.
> - **SceneVersion:** a snapshot after every change (undo/redo and history).
>
> **Multiple scenes:**
> - An **Import scene** button accepts a scene JSON file (format = `assets/venue_twin.json`).
> - A **scene switcher** in the top bar.
> - **"Reset scene"** restores the imported original.
> - Import both scenes now: **"VibeCon Hall"** (`venue_twin.json`, our scanned hall) and **"Studio Theater"** (`theater_twin.json`, a 152-seat black-box theater).
>
> **Catalog and models:** import `catalog.json`, `kits.json` and every file in `models/` (GLB) into built-in storage. A scene can only use types that exist in the catalog; import shows a warning for unknown types.
> - **Assets page ( /admin/assets ):** lets me re-upload `catalog.json`, `kits.json`, scene files and model files at any time. Replace by type or scene name, keep everything else. We will keep updating these files, so updates must not need a code change.
>
> **Pages:**
> - Landing ( / )
> - Editor ( /venue/:sceneId )
> - How it works ( /how )
>
> Use the design system in `DESIGN.md` (I'll upload it with the mockups `design_landing.png`, `design_editor.png`, `design_how.png`).
>
> **Architecture rules:**
> - API keys only in backend environment variables.
> - Files in built-in storage, never local disk.
> - AI calls as background jobs with status polling (requests time out at ~2 min).
> - 3D with **@react-three/fiber v9 + @react-three/drei v10** (React 19), pinned.
>
> **Done when:** both scenes are in the switcher with counts by type (hall: chair 188, table 32, column 12; theater: theater_seat 120, chair 32, riser 7), I can switch between them, and reset a scene.

## L2 — The 3D twin (premium look)
> In the editor, render the current scene in 3D (react-three-fiber).
> - **Instancing:** one instanced mesh per catalog model (hundreds of chairs stay smooth). Floor y = 0. Every model's front faces +z at rotation 0.
> - **Placement (from the catalog's `mount`):**
>   - **floor** items stand on the floor, or on top of a platform when their center is over one (the podium on the stage, seats on risers); platforms themselves stand on the floor
>   - **wall** items sit at their `y` against the wall
>   - **ceiling** items hang from the ceiling (the model's origin is its ceiling mount: pendants, projectors, stage lights, battens)
>   - an item's own `y` always wins; when an item has w/d/h, scale its model from the catalog size to that size
> - **Visual states:**
>   - items with `inferred: true` render at 70% opacity until confirmed
>   - items with a `material` override show it (tables: white / black / walnut; chairs: white / black / gold)
> - **Room:**
>   - walls from the outline, ceiling at the scene's height
>   - colors from the scene's `style`: the hall is warm white with a dark varnished wood floor, the theater is a black box
>   - a **dollhouse toggle** (hide the ceiling, lower the two walls nearest the camera)
> - **Premium lighting (see DESIGN.md "3D look"):**
>   - drei `Environment` from `Lightformer` softboxes: a big overhead softbox, a strip light on each side, a dim warm floor bounce
>   - N8AO ambient occlusion, `NeutralToneMapping`, soft contact shadows
>   - pendant lamps glow softly
> - **Performance:** `PerformanceMonitor` (AO and shadows off on weak devices), pixel ratio ≤ 1.5 on phones.
> - **Camera presets:** Overview (dollhouse corner), Eye level (back of the room looking at the stage), Top-down, with smooth transitions. Use the scene's `cameras` when present.
> - **View from seat:** click any seat (theater seat or chair) to put the camera at that seat, 1.15 m above where the seat stands, looking at the stage center. Show "Seat E10" with a "Back to overview" button.
>
> **Done when:** the hall looks like `assets/media/pipeline_5_twin.jpg`, the theater looks like `assets/media/scene_theater_overview.jpg`, both orbit smoothly on a laptop, the presets work, and clicking seat E10 in the theater shows `scene_theater_seat.jpg`.

## L3 — Blueprint editor + live metrics (like DoBu's site-plan editor)
> Add the **Blueprint panel** (left side, see `design_editor.png`): a top-down plan in blueprint style (navy, white lines) with a grid (0.25 m squares; toggle 0.5 m / 1 ft).
> - **Select:** click, shift-click, drag-box, "select all of this type." Selected items are highlighted yellow in **both** 2D and 3D.
> - **Move:** drag (snaps to the grid). **Arrow keys nudge one square** (Shift = 4).
> - **Rotate / delete / duplicate / undo:** R = 90°, Q/E = 15°, Delete, Cmd/Ctrl+D duplicate, Cmd/Ctrl+Z / Shift+Cmd/Ctrl+Z undo/redo.
> - **"+ Add" palette from the catalog:** "Table + 6 chairs", Chair, Stage riser, Bar stool, and so on. Click to place at the plan center, then drag.
> - **Metrics bar** (top of the editor) with **before → after deltas** after every change:
>   - seats
>   - tables and chairs used vs. owned (warn if over)
>   - **narrowest aisle vs. 44 in (1.12 m)** (red warning and the offending items highlighted)
>   - exits clear
>   - **total price** (owned per-event fees + partner items)
> - 2D and 3D always show the same data; every change is saved as a SceneVersion.
> - **Platforms in the plan:** draw stages, risers and steps as filled tiers (lighter navy) with their front edge. Items standing on a platform don't count as overlapping it. Wall and ceiling items are ignored by the floor overlap and aisle checks.
>
> **Done when:** I can add a table, nudge a table run one square right with the arrow keys, see the 3D update instantly, and the metrics show deltas, including a red aisle warning when tables get too close.

## L4 — The AI edit agent (agentive, not precomputed)
> Add the **Edit chat** (right side, see `design_editor.png`). Use the **Anthropic API** with tool calling (model `claude-opus-5-5`, or `claude-sonnet-5` for speed; I'll paste `ANTHROPIC_API_KEY` when you ask).
>
> **Agent loop (up to 4 rounds per message):** send the system prompt, the conversation, and a compact scene summary (room size, aisle, counts, rows of tables or seats with ids and positions, the stage and other platforms with their heights, fixed seating vs. loose chairs, doors, open areas, this venue's inventory, catalog types with sizes and mounts, kit recipes). Claude calls tools; the backend executes them and **returns results and any rule violations**, so Claude can correct itself. All changes from one message form **one undoable step**; changed items flash in 2D and 3D.
>
> **Query tools:**
> - `get_scene_summary()`
> - `list_items(type?, area?)`
> - `find_open_areas(min_w, min_d, near?)`: returns free rectangles that respect the aisle and exits
> - `measure_aisles()`
>
> **Edit tools:**
> - `add_items(type, placements[{x, z, rot}], group?)`
> - `move(ids, dx_squares?, dz_squares?, to?)`
> - `rotate(ids, degrees)`
> - `remove(ids or group)`
> - `set_material(type or ids, preset)`
> - `swap_type(ids or from_type, to_type)`
> - `undo()`
> - `view_from_seat(label)`: moves the camera (the same view-from-seat as in 3D)
>
> **Validation (backend, after every edit tool):**
> - items stay inside the room
> - no overlaps with columns, walls or the stage beyond 5 cm
> - aisles ≥ 44 in
> - exits clear
>
> Return violations to Claude rather than silently fixing them.
>
> **Kits are recipes, not coordinates.** Put `kits.json`'s plain-language recipes in the system prompt. **Claude computes every position itself** from the scene, e.g. Wedding:
> - tablecloth + centerpiece + 2 candles on every table
> - swap chairs to gold
> - a dance floor in the best open area near the stage
> - a flower arch on the stage
> - string lights across the hall
>
> Tag everything a kit adds with `group`, so "remove the wedding decorations" works.
>
> **Behavior:**
> - Reply in one or two sentences after acting.
> - Never invent item types outside the catalog.
> - Ask when a request is ambiguous.
> - Show a small "thinking / placing 96 items…" status while working.
>
> **Demo commands that must work:**
> - "make all the tables black"
> - "add wedding decorations"
> - "remove the wedding decorations"
> - "theater setup for 150 with a center aisle" (Claude removes the tables, then places chair rows facing the stage itself, using `find_open_areas` and `measure_aisles`)
> - "move the left table run one square right"
> - "add 10 more chairs along the back wall"
> - in the theater: "set up a cabaret night for 64 on the floor" (cabaret kit), "make all the theater seats blue" (seat presets red / blue / black), "add a 4 × 2.4 m thrust in front of the stage" (a stage_deck 0.6 m high), "show me the view from seat E10"
>
> **Done when:** each command works in under ~20 s, passes validation, and is undoable.

## L5 — The signature animation (built here, from scene data) + Landing + How it works
> **Build a reusable `SceneReveal` component**, used as the landing hero **and** whenever a scene opens in the editor. It must be generated **from the scene JSON**, not from a video. Choreography (~10 s, see `design_landing.png` and DESIGN.md "Motion"):
> 1. **Draw (0–3 s):** on cream paper, an SVG blueprint of the scene draws itself like a draughtsman, using stroke-dash animation in ink navy.
>    - Walls first (thick strokes), then fixed elements (columns as circles, the stage, risers), then tables, then seats (thin), staggered from the stage outward. It works for any scene; opening the theater draws its stage and tiers.
>    - A faint drafting grid and dimension ticks along two walls (e.g. "19.75 m", "13.75 m").
> 2. **Erase (3–4.5 s):** a soft rounded eraser sweeps across in a zig-zag and wipes the drawing away. Use an animated SVG mask; leave faint graphite smudges.
> 3. **Redraw (4.5–6.5 s):** a different layout draws in the same way: **Theater** rows facing the stage with a center aisle, computed by the same layout helper the agent uses.
> 4. **Become 3D (6.5–10 s):** crossfade into the react-three-fiber scene, top-down camera matching the plan.
>    - Everything **grows up from the floor** (staggered scale-Y from 0 to 1) as matte warm-white clay.
>    - Real materials then spread across the surfaces.
>    - The camera eases into the overview angle.
>
>    Respect `prefers-reduced-motion` (show the final frame). The hero shows the VibeCon Hall; a "Replay" link restarts it.
>
> **Landing ( / ):**
> - `SceneReveal` as a full-bleed hero (fallback: `media/hero.mp4` if WebGL is unavailable).
> - Headline: **"Your venue, as a living blueprint."**
> - Subline: "Scan once. Rearrange, redecorate and quote any event in seconds, just by asking."
> - Buttons "Open the demo venue" / "How it works."
> - Three benefit cards: Scan → Twin · Edit by talking · Quote instantly.
>
> **How it works ( /how ):** a vertical story with `media/pipeline_1_scan.jpg` … `pipeline_5_twin.jpg`, one line each (scan → vision agents → true scale → blueprint/clay/real → edit by talking), plus the honest note "This venue was reconstructed with our scan pipeline and imported into the app."
>
> **Done when:** the hero plays the full draw → erase → redraw → grow-into-3D sequence smoothly on a laptop, opening a scene in the editor plays the reveal (skippable), and both pages match the mockups.
> **Budget tip:** if credits run low, keep steps 1 and 4 and drop the eraser and redraw.

## L6 (only if credits remain) — pick one
- **Product link → 3D:** paste a URL; the backend extracts name, price, size and image via Claude, then runs image-to-3D via fal.ai TRELLIS.2 (I'll paste `FAL_KEY`), scales the model to size, and adds it to the catalog as a partner item the agent can place.
- **Demo mode:** a button that plays the demo commands in sequence for recording.

---

## Demo video beat sheet (≤ 2:00, screen recording of the live app)
| Time | Beat |
|---|---|
| 0:00–0:12 | Landing hero. A venue manager answers "will 150 fit?" by email and site tours. Strongest stat: Prismm shuts down Dec 31, so venues must switch. |
| 0:12–0:25 | /how: "3 minutes of LiDAR → AI found every object → true-scale twin." |
| 0:25–1:20 | Editor: our scanned hall. Blueprint: add a table, nudge a run one square, live deltas and an aisle warning. Chat: "make all the tables black" → "add wedding decorations" → "theater setup for 150." |
| 1:20–1:35 | Switch to the Studio Theater: its reveal plays, click seat E10 for the view from that seat, then "set up a cabaret night for 64." Same agent, different venue. |
| 1:35–2:00 | Why we win: venue twins in minutes, editable by talking, priced instantly. Close on the wedding view. |

---

## Only if L1 already ran before these files changed (Sat ~10 PM)
> **L1b:** Update the data model: items get optional `y` (elevation), `h` and `label`; scenes get `style` (colors), `cameras` and `inventory`; catalog items get `mount` (floor / wall / ceiling) and `platform`. Re-import `catalog.json`, `kits.json` and all of `models/` (17 new models, and the stage, screen, projector, flower arch and string lights were re-exported), re-import `venue_twin.json`, and import `theater_twin.json` as a second scene. Then follow the updated L2.

