# Lean Emergent plan (Sat ~8:15 PM): 6 prompts, demo-first

**Decision:** we reconstruct the venue ourselves (scan → AI detections → twin) and ship the result as **data and design assets**: a twin JSON, 3D model files and media. The **app, built 100% in Emergent**, imports the twin and does everything interactive:
- the 3D view
- the blueprint editor
- the AI edit agent
- the decoration kits
- the landing and "How it works" pages

**Be transparent** in the video and submission: "We reconstructed this venue with our scan pipeline; the app imports the twin and everything you see is live."

**Confirm with the Emergent desk:** "Can we import a 3D venue model we generated with our own scan pipeline, plus our own 3D model files and a hero video, as data and design assets?"

**Video rule:** the demo video is a screen recording of the live app (no slides beyond a title card). So the pipeline story must live **inside the app**, on the "How it works" page.

## Assets we provide (upload into the project when a prompt asks)
- `venue_twin.json`: room outline and height, plus every item: type, x/z in meters, rotation, size, and `inferred: true|false`.
- `catalog.json`: one entry per object type: name, model file, size, seats, price per unit (assumption), quantity owned.
- `kits.json`: Wedding and Theater, each a list of catalog items plus placement rules.
- `models/*.glb`: the 3D models (chair, table, column, pendant lamp, orange panel, stage, screen, banner, window, door, partition, bin, AC unit, plus decorations: tablecloth, centerpiece, candle, flower arch, dance floor, string lights, curtain, spotlight, aisle runner, gold chair).
- `media/hero.mp4`: landing hero (blueprint draws → eraser → redraw → becomes 3D).
- `media/pipeline_*.jpg`: How-it-works images (LiDAR scan → AI detections → blueprint → clay → twin).

## Credit rules (about 2,000 + the 1,000 refill)
- **Get the 1,000-credit refill now.**
- **One prompt at a time.** Test in the preview, then send the next.
- **If something breaks:** describe the exact bug once (what you clicked, saw, expected). If the next try is still wrong, **roll back to the last checkpoint** instead of looping.
- Use the default (strong) model for L2–L4. A cheaper model, if offered, is fine for L5 only.
- **Rough budget (a guess; check after L1):** L1 150 · L2 300 · L3 350 · L4 350 · L5 150 · fixes 400 = **~1,700 credits**.

---

## L1 — App skeleton + import our venue
> Build a single-venue web app called **VenueTwin**: an AI venue planner for small event venues.
>
> **Data:**
> - **Twin:** room outline in meters, ceiling height, and a list of items. Each item has id, type, x, z, rotation in radians, optional width/depth, an `inferred` flag and an optional `kit` name.
> - **CatalogItem:** type, name, 3D model file, width/depth/height, seats, unit price, quantity owned.
> - **Kit:** name, item list, placement rules.
> - **EditHistory:** list of changes, for undo.
>
> **Import:** load the files I upload (`venue_twin.json`, `catalog.json`, `kits.json`, the `models/*.glb` files) into built-in storage and the database. Add a "Reset demo venue" button that restores the imported twin.
>
> **Pages:** Landing ( / ), Twin editor ( /venue ), How it works ( /how ).
>
> **Architecture rules:**
> - API keys only in backend environment variables.
> - Store files in built-in storage, never on local disk.
> - Any AI call runs as a background job with status polling (requests time out at ~2 min).
> - For 3D use **@react-three/fiber v9 + @react-three/drei v10** (matching React 19), pinned.
>
> **Done when:** after uploading the files, /venue lists the imported items by type with counts (e.g., chair 188, table 32, column 12), and "Reset demo venue" works.

## L2 — The 3D twin (premium look)
> On /venue, render the twin in 3D with react-three-fiber.
> - **Instancing:** one instanced mesh per catalog model (hundreds of chairs must stay smooth). Use each item's x, z and rotation; the floor is y = 0. Items with `inferred: true` render slightly faded.
> - **Room:**
>   - walls from the outline (white), ceiling at the given height
>   - dark varnished wood floor
>   - a "dollhouse" toggle that hides the ceiling and lowers the two near walls
> - **Premium lighting:** drei `Environment` built from `Lightformer` softboxes (big overhead softbox, a strip light each side, a dim warm floor bounce), N8AO ambient occlusion, `NeutralToneMapping`, soft contact shadows.
> - **Quality and performance:** `PerformanceMonitor` turns AO and shadows off on weak devices; pixel ratio ≤ 1.5 on phones.
> - **Camera presets:** Overview (dollhouse corner), Eye level (back of the room looking at the stage), Top-down plan.
>
> **Done when:** the venue looks like a premium architectural render, orbits smoothly on a laptop, and the presets work.

## L3 — Blueprint editor + live metrics (DoBu-style)
> Add a **Blueprint** panel next to the 3D view: a top-down plan in blueprint style (navy background, white lines) with a visible grid (0.25 m squares; toggle 0.5 m or 1 ft).
> - **Select:** click, shift-click, drag a box, or "select all of this type." Selected items are highlighted yellow in both 2D and 3D.
> - **Move:** drag to move (snaps to the grid). **Arrow keys nudge one grid square** (Shift = 4).
> - **Rotate / delete / duplicate / undo:** R = 90°, Q/E = 15°, Delete, Cmd/Ctrl+D duplicate, Cmd/Ctrl+Z undo, Shift+Cmd/Ctrl+Z redo.
> - **"+ Add" palette from the catalog:** Table (comes with 6 chairs), Chair, Round table, Stage riser, Bar, and so on. Click to place at the view center, then drag.
> - **Live metrics panel with before → after deltas** after every change:
>   - seats
>   - tables and chairs used vs. owned (warn if over)
>   - **narrowest aisle vs. 44 in (1.12 m)**: red warning plus the offending items highlighted
>   - exits clear yes/no
>   - total price (owned items' fees + partner items)
> - 2D and 3D always show the same data.
>
> **Done when:** I can add a table, nudge a table run one square right with the arrow keys, see the 3D update instantly, and watch the metrics show deltas, including a red aisle warning when I push tables too close.

## L4 — The AI edit agent + decoration kits
> Add an **Edit chat** under the blueprint panel. Use the **Anthropic API** (Claude Opus 5.5, or Sonnet for speed) with tool calling.
>
> **Request:** send the scene as compact JSON (room, items with ids/types/positions, materials, catalog, kits, open areas) plus the message.
>
> **Tools** (execute all calls from one message as **one undoable step**; changed items flash):
> - `select(type?, area?, near?)`
> - `move(ids, dx_squares, dz_squares)`
> - `rotate(ids, degrees)`
> - `set_material(type, preset)`. Presets:
>   - tables: white / black / walnut / linen
>   - chairs: white / black / gold
> - `add_items(catalog_type, count, rule: rows | around_each_table | along_wall | on_stage | in_open_area)`
> - `remove(ids)`
> - `apply_kit(name)` / `remove_kit(name)`
> - `generate_layout(setup: theater | banquet | classroom, guests)`: rows facing the stage with a 44-in center aisle, skipping columns, never blocking exits
> - `undo()`
>
> **Kits** (from `kits.json`):
> - **Wedding:**
>   - a tablecloth, centerpiece and 3 candles on every table
>   - chairs set to gold
>   - a 4 × 2.5 m dance floor in the largest open area (loose chairs nearby are cleared)
>   - a flower arch on the stage
>   - string lights across the hall
> - **Theater:**
>   - `generate_layout(theater)`
>   - a black curtain behind the stage
>   - two spotlights
>   - a red aisle runner
>
> **Behavior:**
> - Reply in one short sentence.
> - Never invent items outside the catalog.
> - Ask if a request is ambiguous.
>
> **Demo commands that must work:** "make all the tables black," "add wedding decorations," "theater setup for 150 with a center aisle," "move the left table run one square right," "remove the wedding kit."
>
> **Done when:** each of those works in under ~10 s and can be undone.

## L5 — Landing page + How it works
> **Landing ( / ):**
> - Full-screen hero playing `media/hero.mp4` (muted, autoplay, loops), cream-paper aesthetic, thin drafting lines.
> - Headline: "Your venue, as a living blueprint."
> - Subline: "Scan once. Rearrange, redecorate and quote any event in seconds."
> - CTA "Open the demo venue" → /venue.
> - Three short benefit cards: Scan → Twin, Edit by talking, Quote instantly.
>
> **How it works ( /how ):** a vertical story with the pipeline images:
> 1. LiDAR walk-through (3 minutes with an iPhone)
> 2. AI finds every object and groups duplicates into types
> 3. Everything placed in true scale
> 4. Blueprint → clay → real materials
> 5. Edit by talking
>
> Each step gets one line of text. Include a short honest note: "This venue was reconstructed with our scan pipeline and imported into the app."
>
> **Done when:** the landing looks premium on laptop and phone, and /how tells the pipeline story in under 30 seconds of scrolling.

## L6 (only if credits remain) — pick one
- **Product link → 3D** (EMERGENT_PROMPTS.md P7c).
- **Demo mode:** a preset button sequence that plays the 5 demo commands for the video.

## Demo video beat sheet (≤ 2:00, screen recording of the live app)
| Time | Beat |
|---|---|
| 0:00–0:12 | Landing hero. Customer: a small venue manager answering "will 150 fit?" by email and site tours. Strongest stat: the Prismm shutdown on Dec 31 forces venues to switch. |
| 0:12–0:25 | /how: "3 minutes of LiDAR → AI found 12 object types → true-scale twin." |
| 0:25–1:35 | /venue: our scanned hall in 3D. Blueprint: add a table, nudge a run one square with live deltas and an aisle warning. Chat: "make all the tables black" → "add wedding decorations" → "theater setup for 150." |
| 1:35–2:00 | Why we win: real venue twins in minutes, editable by talking, priced instantly. Close on the wedding render. |
