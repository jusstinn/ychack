# VenueTwin assets (data + design assets for the Emergent app)

Upload these into the Emergent project when prompt **L1** (docs/EMERGENT_LEAN_PLAN.md) asks for them.

- `venue_twin.json`: the reconstructed VibeCon hall. Room outline, 3.68 m ceiling, and 385 items (188 chairs, 32 tables, 12 columns, 34 pendant lamps, 50 orange panels, windows, doors, stage, screens…). Coordinates are meters; see `coordinate_system` inside the file. `inferred: true` marks items filled in by pattern where the scan had no coverage.
- `catalog.json`: one entry per object type, with its model file, size, seats, per-event price (partner prices are demo assumptions) and quantity owned.
- `kits.json`: Wedding and Theater kits, as placement rules for the edit agent.
- `models/*.glb`: 32 3D models (furniture, fixtures, decorations). Every model's front faces +z; the pendant's origin is its ceiling mount.
- `media/`: hero video and how-it-works images (being added).

Provenance: reconstructed from a 2.7-minute iPhone LiDAR scan (Stray Scanner) plus vision-agent detections on 95 keyframes; this was done outside the app. The app imports it.
