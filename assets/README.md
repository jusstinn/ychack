# VenueTwin assets (data + design assets for the Emergent app)

Upload these into the Emergent project when prompt **L1** (docs/EMERGENT_LEAN_PLAN.md) asks for them.

## Scenes (two venues)
- `venue_twin.json`: **VibeCon Hall**, reconstructed from our scan. Room outline, 3.68 m ceiling, 361 items (188 chairs, 32 tables, 12 columns, 34 pendant lamps, 37 orange acoustic panels in pairs on the piers between 8 windows, doors, stage, screens…). `inferred: true` marks items filled in by pattern where the scan had no coverage.
- `theater_twin.json`: **Studio Theater**, a 152-seat black-box theater (15 × 16 m, 6.4 m lighting grid). It has:
  - a stage deck with red house curtains, valance and a lit cyclorama
  - 6 seating tiers with 120 theater seats (rows C–H), plus 2 floor rows of 32 loose chairs (rows A–B)
  - aisle steps, 6 exits with signs, a control booth and 33 stage lights
  - This venue was **modeled from typical black-box dimensions, not scanned**; it shows that the app works for any venue.

Both scenes use the same format. Coordinates are in meters; see `coordinate_system` inside each file.
- `y`: elevation of the model origin. It is set for items standing on the stage or risers, and for wall screens, panels and exit signs.
- `w` / `d` / `h`: size overrides. The model is scaled to them.
- `label`: a seat label such as "E10".
- `style`: wall, floor and ceiling colors.
- `cameras`: named viewpoints. In the theater, `eye` is the view from seat E10.
- `inventory`: quantities this venue owns, overriding the catalog.

## Catalog, kits, models
- `catalog.json`: 56 object types. Each has:
  - model file, size, seats
  - per-event price (partner prices are demo assumptions) and quantity owned
  - `mount`:
    - `floor`: stands on the floor, or on a platform when over one
    - `wall`: placed at its `y`
    - `ceiling`: hangs from the ceiling; the model origin is the mount point
  - `platform`: true for the stage, stage deck, risers and aisle steps
  - `materials`: color presets the agent can switch to (chairs, tables, theater seats, sofas, armchairs)
- `kits.json`: Wedding, Theater, Conference, Cabaret and Living room set, as plain-language recipes for the edit agent. Positions are never precomputed.
- `models/*.glb`: 56 3D models. Every model's front faces +z.
  - **Hall:** chairs, trestle tables, columns, pendants, panels, stage…
  - **Theater:** theater seat, riser, aisle step, stage deck, stage steps, cyc, drape, house curtain, valance, batten, stage light, house light, PA speaker, double door, exit sign, booth window
  - **Decorations:** tablecloth, centerpiece, candle, dance floor, flower arch, string lights, curtain, spotlight, aisle runner, banquet round table
  - **Set pieces / props:** sofa, armchair, coffee table, area rug, floor lamp, bookshelf, scenic flat (wallpapered set wall). Sofas and armchairs come in teal, mustard, grey or burgundy.

## Media
- `media/hero.mp4` (10.5 s, 1600×1000, H.264) + `hero_poster.jpg`: the landing hero sequence. The hall's blueprint draws itself, an eraser wipes it, the theater layout redraws, then it turns into 3D (clay, then real) as the camera swoops to the overview. It's the fallback for the in-app SceneReveal.
- `media/pipeline_1_scan.jpg` … `pipeline_5_twin.jpg`: the how-it-works story for the scanned hall.
- `media/scene_theater_overview.jpg`, `scene_theater_seat.jpg` (view from seat E10), `scene_theater_stage.jpg`, `scene_theater_blueprint.jpg`, `scene_theater_clay.jpg`: the theater's look targets. `scene_theater_livingroom_seat.jpg` shows the stage after "dress the stage as a living room".
- `media/wedding_*.jpg`, `twin_overview.jpg`, `hall_theater_setup_eye.jpg`: look targets for edits in the hall.
- `design/*.png`: UI mockups (see docs/DESIGN.md).

Provenance:
- **Hall:** reconstructed from a 2.7-minute iPhone LiDAR scan (Stray Scanner) plus vision-agent detections on 95 keyframes. This was done outside the app.
- **Theater:** a modeled sample venue.
- The app imports both.
