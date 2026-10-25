# VenueTwin design system (for Emergent to implement)

> Mockups: `assets/design/design_landing.png`, `design_editor.png`, `design_how.png`. Upload them with this file at L1. They are pictures to match, not code.

## 1. Idea
**An architect's drafting board that comes alive.**
- Marketing pages: cream paper and navy ink linework.
- Editor: a dark, focused "design tool" chrome around a premium 3D scene, with a blueprint panel beside it.
- One hot accent, **venue orange** (the hall's acoustic panels), and **selection yellow**.
- Calm, precise, no gradients except subtle paper grain.

## 2. Color tokens
| Token | Hex | Use |
|---|---|---|
| `--paper` | `#F4EFE4` | Landing / How background |
| `--paper-2` | `#ECE5D6` | Cards on paper |
| `--ink` | `#1F3A5F` | Lines, headings on paper |
| `--ink-soft` | `#5B7290` | Secondary text on paper, dimension labels |
| `--graphite` | `#2A2A2A` | Body text on paper |
| `--chrome` | `#0F141A` | Editor top bar, background |
| `--panel` | `#151C25` | Editor side panels |
| `--panel-2` | `#1C2531` | Inputs, chips, cards in editor |
| `--line` | `#2A3644` | Dividers in editor |
| `--text` | `#E7EDF5` | Primary text in editor |
| `--text-dim` | `#93A3B8` | Secondary text in editor |
| `--blueprint` | `#0D2440` | Blueprint panel background |
| `--blueprint-grid` | `#3C6EA3` | Grid lines (major 100%, minor 35%) |
| `--blueprint-ink` | `#EAF1FA` | Walls, furniture outlines in the plan |
| `--accent` | `#FF6A1A` | Primary buttons, agent highlights, "changed" flash |
| `--select` | `#FFD23F` | Selection in 2D and 3D |
| `--ok` | `#3FBF7F` | Checks passed |
| `--warn` | `#F2A93B` | Over inventory, inferred items |
| `--bad` | `#E5484D` | Rule violations (aisle < 44 in, blocked exit) |

## 3. Typography (Google Fonts)
- **Display:** *Instrument Serif* (headlines on Landing / How). H1 72/1.0, H2 44/1.05, letter-spacing −0.5px.
- **UI:** *Inter* 400/500/600. Body 15/1.5, labels 13/1.3 medium, buttons 14 semibold.
- **Data:** *JetBrains Mono* 500 for metrics, measurements, coordinates, tool-call chips. 13px.
- Numbers in metrics always mono with tabular figures.

## 4. Layout and spacing
- 8-pt spacing grid. Radius: 10 px on cards, 8 px on inputs, 999 px on chips and pills.
- Borders: 1 px `rgba(31,58,95,.16)` on paper; 1 px `--line` in the editor.
- Shadows: only on floating menus (`0 8px 24px rgba(0,0,0,.25)`).

### Landing ( / ), see `design_landing.png`
- Full-bleed **SceneReveal** hero on paper.
- Left column (max 520 px): eyebrow "AI VENUE PLANNER" (mono, ink-soft, letter-spacing 2px), H1, subline, primary button (orange) "Open the demo venue", ghost button "How it works".
- Thin vertical drafting lines at the page edges, plus dimension ticks.
- Below the fold: three benefit cards in a row (icon line drawing + title + one sentence), then a footer.

### Editor ( /venue/:sceneId ), see `design_editor.png` (1440×900 reference)
- **Top bar (56 px, `--chrome`):**
  - wordmark "VenueTwin" with a small blueprint-square icon
  - scene switcher pill ("VibeCon Hall ▾")
  - **metric chips** (mono): Seats · Tables used/owned · Narrowest aisle · Exits · Total $
  - each chip shows a delta after a change (e.g. "+6", green/red) for 3 s
  - right side: camera presets (Overview / Eye / Top segmented control), Dollhouse toggle, Replay reveal
- **Left: Blueprint panel (420 px, `--blueprint`):**
  - toolbar: Select · Move · Rotate · + Add · Grid 0.25 m · Undo · Redo
  - the plan with grid, selection in yellow, and the previous position as a faint ghost after a nudge
  - footer hint line: "←→↑↓ nudge 1 square · Shift ×4 · R rotate · ⌘Z undo"
- **Center: 3D canvas:** dark background `#1D1F23` in dollhouse mode. A floating "inferred items" legend appears when any exist.
- **Right: Edit agent (380 px, `--panel`):**
  - header "Edit agent" + status dot
  - messages: the user's in `--panel-2`; the agent's as text plus **tool-call chips** (mono, small, e.g. `swap_type · chair → chair_gold · 188`)
  - suggestion chips: "Add wedding decorations", "Theater for 150", "Make tables black", "Remove wedding"
  - input with send button (orange)
- **"+ Add" palette:** a popover of catalog cards (thumbnail, name, size, price, owned count).
- **View from seat:** clicking a seat moves the camera to that seat. A pill at the top-left of the 3D canvas shows "Seat E10" (in mono), with a ghost button "Back to overview". The look target is `assets/media/scene_theater_seat.jpg`.

### How it works ( /how ), see `design_how.png`
- Paper background, H2 "From a 3-minute scan to a living twin."
- Five steps in a vertical timeline: a number in a circle (ink), one line of text, and a wide image with a 10 px radius.
- The honest provenance note sits at the end in ink-soft.

## 5. Motion
- **Easing:** `cubic-bezier(.2,.7,.2,1)` everywhere.
- **Durations:** UI 160–240 ms. Tool-call chips appear in sequence, 80 ms apart.
- **SceneReveal** (landing hero and scene open):
  - **draw**, 3 s, stroke-dash: walls → columns → tables → chairs, staggered from the stage outward
  - **erase**, 1.5 s: zig-zag eraser mask with faint smudges
  - **redraw** the theater layout, 2 s
  - **grow into 3D**, 3.5 s: items scale up from the floor in clay with a 12 ms stagger, real materials spread in, and the camera eases to overview
- **Changed items** flash `--accent` for 600 ms. **Selection** shows a yellow outline in 2D and a yellow rim in 3D.
- Respect `prefers-reduced-motion` (show final states).

## 6. 3D look (react-three-fiber)
- **Lighting:** drei `Environment` from `Lightformer`s:
  - overhead softbox, intensity ~2, scale 10×4
  - two side strips, intensity ~1
  - a warm floor bounce, intensity ~0.3
  - no HDRI file needed
- **Post-processing:** N8AO (radius ~0.3 m) and `NeutralToneMapping`, exposure ~0.8. Soft contact shadows under furniture.
- **Materials:**
  - walls warm white `#F2EFE8`
  - floor dark varnished wood, clearcoat
  - pendant lamp shades emissive warm `#FFE2BD`, low intensity
  - inferred items at 70% opacity
- **Per-venue room colors** come from the scene's `style`. The hall has warm white walls and a dark wood floor. The theater is a black box: matte black walls and a black floor. Keep the same lighting there. The cyc gradient, stage-light lenses, exit signs and step lights are emissive in the models, so they glow by themselves. The look targets are `scene_theater_overview.jpg` and `scene_theater_stage.jpg`.
- In dollhouse mode, draw a thin light cap (`#9FA2AA`) on top of cut walls so dark rooms keep a readable outline.
- **Performance:**
  - one InstancedMesh per model type
  - device pixel ratio ≤ 1.5 on phones
  - AO and shadows off when `PerformanceMonitor` reports a weak device

## 7. Voice and copy
- Short, confident, concrete. Use numbers, not adjectives: "188 seats · aisle 1.60 m · $2,718."
- The agent replies in 1–2 sentences, saying what changed and the effect: "Dressed 32 tables, swapped 188 chairs to gold, dance floor placed by the stage. +$2,342."
