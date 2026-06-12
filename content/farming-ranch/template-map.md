# Farming / Ranch template ID map (`farm_*`, `ranch_*`, `beast_*`)

Authoritative mapping between numerical template IDs and data rows. Planet/biome rules live in datatables, not in template names.

| ID | Kind | Datatable row | Notes |
|----|------|-----------------|-------|
| `ranch_s01` | Ranch garden bed (marker / collision hook) | `ranch_rules` (`rule_id=default`, bed substring `ranch_s`) | Player-facing deed-style placeable; objvar `ranch.bed=1`. Hidden cell volume is authored on the shared/building side separately. |
| `farm_s10` | Crop seed (inventory) | `crop_definitions.crop_id=mvp_wheat` | Consumed by `farmPlant`; objvar `farm.cropId=mvp_wheat`. |
| `farm_s11` | Growing plant (world) | same crop row | Spawned at bed cell; objvars `farm.cropId`, `farm.plantedAt`; scale 0.10–1.20 from growth curve. |
| `beast_s50` | BM / ranch livestock placeholder creature | `ranch_breeding` | Reference creature template name substring for breeding filters (replace with real creature when content exists). |

Datatable sources (compiled to `datatables/farming/*.iff`):

- `crop_definitions` — growth, harvest output, Phase D `moisture_bonus` / `soil_category` columns.
- `ranch_rules` — bed name substring, max breed distance, plant cap.
- `economy_constants` — sink vs income tuning (ints).
- `ranch_breeding` — offspring template, cooldown, parent name filters.
- `soil_weather` — Phase D per-planet modifiers.

Commands: `farmPlant` (target=bed, param=seed OID), `farmHarvest` (target=plant), `ranchBreed` (target=parentA, param=parentB OID).
