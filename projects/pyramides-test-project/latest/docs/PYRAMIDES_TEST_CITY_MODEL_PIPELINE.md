# Pyramides Test Project - City Model Pipeline

This document records the detailed build recipe for the Lille "Pyramides Test Project" city-area model.

Preview target:

- Local URL: `http://127.0.0.1:8770/reviews/lille_building_blocks/index.html`
- Review folder: `apps/preview_server/web/reviews/lille_building_blocks`
- Generator: `tools/build_lille_building_block_test.py`
- Landmark database: `data/landmark_model_database`
- Project name in UI: `Pyramides Test Project`

## Current Reconciled State - 2026-05-25

This section is the source of truth for the current standalone Pyramides preview.

The active standalone review has now been rebuilt from `GPX_Terrain_Studio_v3_LIVE/tools/build_lille_building_block_test.py` with the missing late changes restored:

- selected address red marker enabled for `6 rue des Pyramides, Lille`
- selected address source footprint: `way/42671152`
- selected address manifest result: `selected address red hits = 1`
- normal and selected-address building heights normalized to `3.0` default levels
- default normal-building height: `3.0 x 3.1 m = 9.3 m`
- three automatic landmark model replacements inserted from `data/landmark_model_database`
- custom landmarks use solid opaque white material
- roof treatment remains disabled
- printability status: `READY`
- auto-rotate remains off in the published viewer

Important correction:

- The generic Building Authority run `buildings_009` had the selected-address and height fixes first.
- The standalone Pyramides preview had the landmark replacements first.
- The current rebuilt standalone preview combines those two lines of work for selected address, normalized heights, and landmark replacements.
- The standalone preview still does not yet include the later road/green/water visible mask implementation described as a future/required layer below. Road, rail, and water are currently used as no-merge barriers in the standalone generator, not as visible printable overlays.

The goal of this test model is to prove a scalable city-to-print workflow on a small rectangle of Lille before applying it to larger or global terrain products.

## 1. Terrain Rectangle

The current test area is a small Lille rectangle around Rue des Pyramides, Theatre Sebastopol, Palais des Beaux-Arts, and Eglise Saint-Michel.

Geographic bounds:

```json
{
  "min_lat": 50.62565,
  "max_lat": 50.63215,
  "min_lon": 3.05525,
  "max_lon": 3.06745
}
```

Approximate metric size:

- Width: about `862 m`
- Depth: about `719 m`
- Model width: `4.0` model units
- Model depth: about `3.3371` model units
- Reference print width: `150 mm`
- Reference print depth: about `125.14 mm`
- Model-units-per-mm: `4.0 / 150 = 0.0266667`

Coordinate conversion:

- Longitude maps linearly to model `X`.
- Latitude maps linearly to model `Z`, with north at the top of the rectangle.
- Model origin is the center of the rectangle.
- Model bounds are approximately:
  - `X = -2.0` to `+2.0`
  - `Z = -1.6686` to `+1.6686`

## 2. Primary Source Data

Building, road, rail, water, landmark, and amenity context data are sourced from OpenStreetMap through Overpass in the active standalone generator.

Provider:

- Name: `OpenStreetMap via Overpass API`
- Endpoint: `https://overpass-api.de/api/interpreter`
- License: Open Data Commons Open Database License, ODbL
- Attribution text: `© OpenStreetMap contributors`
- Commercial posture: commercial use is possible with ODbL attribution and share-alike obligations for derivative databases.
- Fallback policy: `no_silent_fallback`

The generator caches the raw Overpass response in:

```text
apps/preview_server/web/reviews/lille_building_blocks/lille_osm_overpass_response.json
```

The cache exists so that local rebuilds do not silently change when OSM changes or when Overpass is unavailable.

Currently queried OSM feature groups in the standalone generator:

- `building=*`
- `highway=*`
- `railway=*`
- `bridge=*`
- `waterway=*`
- `tourism=*`
- `historic=*`
- `amenity=*`

The query also gathers nodes for `tourism`, `historic`, and `amenity` so that landmark classification can use nearby semantic evidence.

Planned/required green-space query additions:

- `leisure=park|garden|recreation_ground|playground|pitch`
- `landuse=grass|forest|meadow|recreation_ground|village_green|allotments|cemetery`
- `natural=wood|grassland|scrub|heath|water`

## 3. Source Footprint Parsing

Each OSM element is converted into local model geometry.

Buildings:

1. OSM ways with `building=*` become polygons from their geometry.
2. OSM relations with `building=*` are rebuilt from outer rings.
3. Polygons are clipped to the terrain rectangle.
4. Invalid or empty geometry is discarded.
5. Each retained source building record stores:
   - polygon shape
   - original OSM tags
   - `osm_ref`, for example `way/152835085`

Roads, rail, and waterways:

1. OSM line geometry is converted to `LineString`.
2. The line is buffered into a flat mask polygon.
3. That polygon is clipped to the terrain rectangle.
4. The same mask is used for:
   - visual road/water/rail overlay
   - no-merge barriers for building grouping

Green spaces:

1. This is required but not active in the standalone rebuilt preview yet.
2. Parks, gardens, grass, forests, meadows, cemeteries, playgrounds, pitches, scrub, heath, and other configured green features should be converted to polygons.
3. They should be drawn as flat green overlays on the terrain base in the next integration pass.

## 4. Building Classification

Every source building starts as `standard_building_or_marker` unless later rules promote it.

Categories:

- `standard_building_or_marker`: normal white buildings
- `enhanced_landmark_massing`: green preserved landmarks
- `bespoke_3d_model`: source footprint is replaced by a custom landmark model
- `selected_address_marker`: user-selected red building

Landmark classification uses:

- OSM tags such as `amenity`, `tourism`, `historic`, `religion`, and names.
- Target landmark text tokens for this test:
  - Palais des Beaux-Arts
  - Theatre Sebastopol
  - Eglise Saint-Michel
- Target landmark points inside the rectangle:
  - Palais des Beaux-Arts de Lille
  - Theatre Sebastopol
  - Eglise Saint-Michel

Large/high building preservation:

- A building is treated as large/high if:
  - its area is at least `frame.width * frame.depth * 0.0022`, or
  - `building:levels >= 6`, or
  - `height >= 22 m`

These large/high buildings remain individual instead of being merged into small building groups.

## 5. Height Policy

The latest standalone visual test intentionally normalizes normal and selected-address buildings to reduce unrealistic block-to-block height differences.

Normal building policy:

- Normal and selected-address buildings are normalized to the project default level count.
- Current default: `3.0` levels.
- Level height: `3.1 m`.
- Normalized height: `9.3 m`.
- This prevents entire blocks from looking much taller only because one OSM tag or grouping artifact implied extra height.

Source height parsing still exists for audit and future modes:

- Explicit `height=*` is high confidence.
- `building:levels=*` or `levels=*` is medium confidence.
- One level equals `3.1 m`.
- Missing height is inferred with low confidence.

Clamp rules:

- Minimum normal building height: `4.0 m`
- Maximum normal building height: `34.0 m`
- Maximum landmark/protected building height: `55.0 m`

Print minimums:

- Minimum building print height: `1.1 mm`
- Minimum landmark print height: `2.3 mm`
- Landmark/custom models can keep sourced or protected height behavior.

## 6. Building Grouping And Printable Modes

The project contains multiple model modes for review.

### Original All Buildings

Shows every OSM source footprint without grouping.

Purpose:

- Verify source coverage.
- Verify landmark classifications.
- Verify selected-address red building logic.

### Touch Consolidated

Only touching or overlapping normal buildings are dissolved together.

Important rule:

- No street-crossing buffer is used for normal building merge.
- Street gaps and courtyards should remain readable.

Tiny isolated fragments:

- Fragments below the printability threshold can be removed or flagged.
- This keeps the model printable without creating unrealistic filled blocks.

### Printable v1

Print-polished output from the touch-consolidated source.

Operations:

- Tiny physical corner polish.
- Tiny/sliver courtyard repair.
- Small isolated fragment handling.
- Coverage audit against original OSM footprints.

### Printable v2

Required next standalone review model.

Rules:

- Use individual source footprints or print-safe touch groups according to the active review mode.
- Keep normal-building height normalized to the project default unless a future mode explicitly opts into per-building heights.
- Keep landmark 3D model replacements inserted.
- Add visible roads, water, and green-space masks as flat overlays.
- Keep roof treatment disabled.

Current status:

- Not yet generated in the active standalone review folder.
- The current active standalone model is `Printable v1` with selected-address red fix, normalized heights, and three landmark replacements.

## 7. Base And Material Colors

The model is intentionally restrained and mostly white so landmarks, roads, water, green spaces, and selected address markers remain legible.

Core colors:

| Element | Color | Hex |
| --- | --- | --- |
| Normal buildings | white | `#FFFFFF` |
| Enhanced landmark footprints | green | `#16A34A` |
| Selected address building | red | `#DC2626` |
| Green spaces / parks | light green | `#7DDF96` |
| Major roads | light beige | `#E8D8B8` |
| Secondary roads | light beige | `#E8D8B8` |
| Local streets | light beige | `#E8D8B8` |
| Service roads | gray | `#9CA3AF` |
| Pedestrian / cycle | purple | `#7C3AED` |
| Rail | near black | `#111827` |
| Waterways | blue | `#2563EB` |
| Other roads | gray | `#6B7280` |
| Preview page background | pale gray | `#EDF2F7` |

3D material properties:

- Materials use PBR color factors.
- Landmark custom models are forced to solid opaque white.
- Landmark material name: `landmark-micro-model-solid-white`
- Landmark material is double-sided.
- Roads/green/water overlays sit just above the base to prevent z-fighting.

Vertical placement:

- Base top Y: `0.048`
- Flat visual overlays use approximately `BASE_TOP_Y + 0.0012`
- Landmark inserted models start just above the base, around `0.0485`

## 8. Road, Rail, And Water Mask Widths

Roads are generated from OSM lines using Shapely `line.buffer(width)`.

Standalone current behavior:

- Road/rail/water line masks are used as no-merge barriers for building grouping.
- They are not yet exported as visible beige/blue/black printable overlays in the standalone review folder.
- The generic Building Authority run `buildings_009` did export printable road/street strips, but that output does not yet include the landmark-model replacement pass.

Important detail:

- The configured number is the buffer radius on each side of the source centerline.
- The visible full strip width is approximately `2 x buffer`.
- Example: a `1.6 m` buffer produces an approximately `3.2 m` full visual strip.

The current narrowed road mask buffers are:

| OSM class | Buffer per side | Approx full strip |
| --- | ---: | ---: |
| `primary` | `3.2 m` | `6.4 m` |
| `secondary` | `2.6 m` | `5.2 m` |
| `tertiary` | `2.3 m` | `4.6 m` |
| `residential` | `1.6 m` | `3.2 m` |
| `unclassified` | `1.6 m` | `3.2 m` |
| `living_street` | `1.5 m` | `3.0 m` |
| `service` | `1.2 m` | `2.4 m` |
| `pedestrian` | `1.2 m` | `2.4 m` |
| `footway` | `0.8 m` | `1.6 m` |
| `path` | `0.8 m` | `1.6 m` |
| `cycleway` | `0.8 m` | `1.6 m` |
| `steps` | `0.7 m` | `1.4 m` |
| unknown/default highway | `1.4 m` | `2.8 m` |
| `waterway=*` line | `1.2 m` | `2.4 m` |
| `railway=*` minimum | `1.7 m` | `3.4 m` |

Required future default visible mask classes:

- `major`
- `secondary`
- `local`
- `water`

Counts observed in the generic Building Authority road pass for the current Lille rectangle:

- Major roads: `0`
- Secondary roads: about `68`
- Local streets: about `69`
- Service roads: about `45`
- Pedestrian/cycle: about `193`
- Rail: about `3`
- Waterways: about `1`
- Other roads: about `1`

Required future standalone UI behavior:

- Road layer checkboxes control both the 2D preview overlays and the active 3D GLB combo.
- Default 3D combo is `secondary + local + water` because there are no `major` masks in this rectangle.
- Water must be included by default and shown in blue when data exists.

## 9. Green-Space Mask Sources

Green spaces should use globally scalable OSM tags rather than a Lille-specific dataset.

Required classes for the next standalone green-space pass:

- `leisure=park`
- `leisure=garden`
- `leisure=recreation_ground`
- `leisure=playground`
- `leisure=pitch`
- `landuse=grass`
- `landuse=forest`
- `landuse=meadow`
- `landuse=recreation_ground`
- `landuse=village_green`
- `landuse=allotments`
- `landuse=cemetery`
- `natural=wood`
- `natural=grassland`
- `natural=scrub`
- `natural=heath`

Commercial/scalability posture:

- OSM is globally available and commercially usable with attribution and ODbL compliance.
- For production, the attribution manifest must keep OSM attribution.
- If a derived database is distributed, ODbL share-alike obligations must be reviewed.

## 10. Selected Address Red Building

The selected-address feature lets a user promote one source building to red.

Selection input fields:

- Address text
- Optional explicit OSM reference, for example `way/42671152`
- Optional latitude
- Optional longitude

Resolution order:

1. Explicit `osm_ref` if supplied.
2. Known local override when an address exists in the UI but is not tagged in OSM.
3. Coordinate containment if latitude/longitude are supplied.
4. Nearest building within a strict distance when coordinates are supplied.
5. Exact-ish address tag match from OSM tags.

Current known override:

- `6 rue des pyramides lille` maps to `way/42671152`

This override exists because the cached OSM building data does not contain `addr:housenumber=6` for Rue des Pyramides.

Current active selection file:

```text
apps/preview_server/web/reviews/lille_building_blocks/building_address_selection.json
```

Current active selection:

- label: `6 rue des Pyramides, Lille`
- `osm_ref`: `way/42671152`
- latitude/longitude hint: `50.6278, 3.06`
- match method in the rebuilt manifest: `explicit_osm_ref`

Expected manifest result when the selected address is active:

- `selected address red hits: 1`
- `selected address red footprints: 1`
- `source red classified: 1`

## 11. Landmark Database Replacement Contract

Landmark model insertion is a core requirement.

The generator must automatically include suitable landmarks from:

```text
data/landmark_model_database
```

It must not rely on a city-specific hard-coded replacement list for production.

### Landmark Discovery

For every build:

1. Read `data/landmark_model_database/landmark_model_database_index.json`.
2. Iterate every record in `records`.
3. Read each record's `anchor_wgs84.latitude` and `anchor_wgs84.longitude`.
4. Keep records whose anchor point falls inside the terrain bounds.
5. Prefer records marked as:
   - `micro_model_preferred: true`
   - `replacement_status: candidate_for_footprint_replacement`
6. Skip records outside the current terrain bounds.

For this test rectangle, records inside the bounds are:

- `theatre_sebastopol_lille`
- `palais_beaux_arts_lille`
- `saint_michel_lille`

Records outside the bounds, such as `eglise_sacre_coeur_lille`, must remain in the database but must not be inserted into this terrain.

### Footprint Matching

For each inside-bounds landmark record, the generator should find the source OSM building footprint to replace.

Preferred matching order:

1. Explicit OSM reference in the landmark metadata, for example:
   - Theatre Sebastopol: `way/37594524`
   - Palais des Beaux-Arts: `way/152835085`
   - Saint-Michel: `relation/11371647`
2. If an explicit OSM reference is unavailable, use the landmark anchor point and find the building polygon that contains or touches it.
3. If no containing building exists, use nearest building within a strict distance threshold.
4. If no reliable footprint exists, skip insertion and write a warning to the manifest.

Replacement rule:

- When a custom landmark model is inserted, the corresponding default OSM building footprint must be removed from normal building rendering.
- The custom model replaces the building, not merely sits on top of it.

### Model File Choice

Use the micro print model by default:

1. Prefer `models/micro/*_micro_print_slicer_ready.stl`.
2. If unavailable, use the micro GLB or micro STL listed in `micro_files`.
3. Use full models only when no accepted micro model exists.

Reason:

- The terrain area is physically small at `150 mm` width.
- Full landmark details can become fragile or noisy.
- Micro models are already simplified for print readability.

### Axis Conversion

The landmark packages use model-local conventions that may differ by file format.

Current STL insertion convention:

- Source STL uses horizontal `X/Y` and vertical `Z`.
- Terrain uses horizontal `X/Z` and vertical `Y`.
- Therefore the loader converts old source axes to:
  - source `X` -> terrain `X`
  - source `Y` -> terrain `Z`
  - source `Z` -> terrain `Y`

After conversion:

- The model is placed on the base top.
- The model is centered on the matched OSM footprint.
- The model is uniformly scaled to the footprint oriented bounding box.

### Scale And Placement

Fit method:

- `oriented_footprint_bbox_uniform_scale`

Steps:

1. Compute the oriented minimum rectangle of the matched source footprint.
2. Extract the footprint major and minor axis lengths.
3. Load the micro STL/GLB.
4. Convert axes to terrain convention.
5. Uniformly scale the model so the model's horizontal footprint fits the real OSM footprint.
6. Rotate using metadata bearing when possible.
7. Translate to the OSM footprint center.
8. Raise onto the terrain base.

Scale reporting:

- Store `scale_model_units_per_source_unit`.
- Store `scale_meters_to_model_units`.
- Store fitted bounds.
- Store target center and target footprint size.

### Orientation

The preferred orientation source is the model metadata field:

```json
orientation.model_axis_bearings_degrees_clockwise_from_true_north["-Y"]
```

If that numeric bearing exists:

- Treat model `-Y` as the documented front facade.
- Rotate the model so model `-Y` faces the real-world bearing.
- Set manifest `rotation_source` to `metadata_model_minus_y_bearing`.

If the numeric bearing does not exist:

- Use the matched footprint's major axis.
- This is weaker because a rectangle has two possible directions.
- If visual review proves it is reversed, add a numeric bearing override to the landmark metadata or generator config.

Current orientation values:

| Landmark | Front/facing convention | Bearing / rotation result |
| --- | --- | --- |
| Theatre Sebastopol | model `-Y` faces Place Sebastopol | override bearing `39.840307°`; rotation about `39.840307°` |
| Palais des Beaux-Arts | model `-Y` faces Place de la Republique | metadata bearing `311.9°`; rotation about `-48.1°` |
| Saint-Michel | model `-Y` is west/front porch, exact GIS yaw unresolved | footprint major-axis rotation about `0.412665°` |

Theatre and Palais were previously reversed because the first-pass footprint fit chose the opposite long-axis direction. The fix is to use model-side bearing metadata or explicit bearing overrides.

### Landmark Material

Inserted custom landmark meshes must be visually solid and opaque.

Rules:

- Apply one solid white material to every inserted landmark model.
- Material name: `landmark-micro-model-solid-white`.
- Color: `#FFFFFF`.
- Alpha: `1.0`.
- `doubleSided: true`.

Reason:

- Imported STL meshes otherwise appeared transparent or visually empty in the preview.
- White custom models match the premium print-output style and avoid introducing source-material noise.

### Landmark Attribution

Every inserted record must carry its attribution path into the manifest.

Required per-landmark metadata:

- `slug`
- `name`
- `package_dir`
- chosen micro model path
- anchor coordinates
- orientation metadata
- attribution report path
- commercial traceability note
- fit report
- replacement result or warning

The final commercial product must keep:

- OSM attribution for footprint placement.
- Landmark package attribution reports.
- Any Commons/Wikidata/source traceability listed in the package.

## 12. Current Inserted Landmark Models

### Theatre Sebastopol

- Slug: `theatre_sebastopol_lille`
- OSM replacement footprint: `way/37594524`
- Preferred model: `models/micro/theatre_sebastopol_lille_micro_print_slicer_ready.stl`
- Real-world reference height: about `29 m`
- Placement: matched to OSM footprint.
- Orientation: model `-Y` is the Place Sebastopol facade.
- Corrected bearing: `39.840307°`
- Material: solid white.

### Palais des Beaux-Arts

- Slug: `palais_beaux_arts_lille`
- OSM replacement footprint: `way/152835085`
- Preferred model: `models/micro/palais_beaux_arts_lille_micro_print_slicer_ready.stl`
- Front facade: Place de la Republique, model `-Y`.
- Metadata `-Y` bearing: `311.9°`
- Rotation result: about `-48.1°`
- Material: solid white.

### Eglise Saint-Michel

- Slug: `saint_michel_lille`
- OSM replacement footprint: `relation/11371647`
- Preferred model: `models/micro/saint_michel_lille_micro_print_slicer_ready.stl`
- Current yaw: footprint-major-axis estimate, about `0.412665°`.
- Model `-Y` is the west/front bell-tower porch, but exact GIS bearing still needs a cleaned footprint/cadastre refinement.
- Material: solid white.

## 13. Roads, Green Spaces, Water, And Landmark Layering

Layer order:

1. Flat rectangular base.
2. Green-space overlays.
3. Road/water/rail overlays.
4. Standard buildings.
5. Selected-address red building.
6. Enhanced landmark massing or custom 3D landmark models.

Road layer UI:

- The left panel lists all road classes.
- Each class has:
  - checkbox
  - color swatch
  - label
  - count
- Checking/unchecking a class updates:
  - the 2D authority overlay image
  - the selected 3D GLB combo when in Printable v2 mode

The default visible classes are intentionally narrow:

- Secondary/local streets are visible enough to understand the city fabric.
- Service/pedestrian/cycle/rail/other remain unchecked by default to reduce visual clutter.
- Water is checked by default because it is a separate blue context layer.

## 14. Preview Page Behavior

The standalone Pyramides review page is intentionally separate from the larger UI.

Primary purpose:

- Fast visual review of one small city rectangle.
- Compare source, topology, printability, and current printable output.

Modes:

- Original All Buildings
- Touch Consolidated
- Printable v2
- Printable v1
- Coverage Audit
- Topology QA
- Printability QA

Model viewer:

- Uses `model-viewer`.
- Camera controls are enabled.
- The viewer fills the central panel.
- Side panels scroll independently.
- Camera target is near the terrain center, around `0m 0.08m 0m`.
- Camera orbit is constrained so the user can inspect the model without flipping into confusing angles.

## 15. Printability Gates

The model emits a printability report:

```text
building_printability_report_v1.json
```

Current status target:

- `READY`
- Combined model watertight: `true`
- Non-manifold edges: `0`

Current warning classes:

- none in the latest standalone rebuild

Other print constants:

- Print width: `150 mm`
- Minimum recommended component area: about `0.18 mm2`
- Minimum recommended visible width: about `0.35 mm`
- Corner polish radius: `0.08 mm`
- Minimum courtyard width: `0.35 mm`
- Minimum courtyard area: `0.20 mm2`
- Extrusion precision grid: `1e-6` model units

The model is allowed to have warnings while we are in visual/fit testing, but failures must be investigated before production export.

## 16. Generated Files

Key review outputs:

- `index.html`
- `lille_building_block_manifest.json`
- `lille_source_authority_all_buildings.glb`
- `lille_touch_consolidated_buildings.glb`
- `lille_printable_buildings_v1.glb`
- `lille_printable_buildings_v1.stl`
- `lille_printable_coverage_audit.png`
- `lille_topology_qa.png`
- `lille_printability_qa.png`
- `building_authority_manifest_v1.json`
- `building_attribution_manifest_v1.json`
- `building_printability_report_v1.json`
- `building_address_selection.json`

Files not currently present in the standalone review folder, but required for the next v2 pass:

- `lille_printable_buildings_v2_green_base.glb`
- `lille_printable_buildings_v2_roads_green.glb`
- `lille_printable_buildings_v2_roads_green.stl`
- `lille_printable_buildings_v2_roads_combo_*.glb`
- `lille_road_layer_*.glb`
- `lille_road_layer_*.png`
- `lille_printable_buildings_v2_roads_green.png`

## 17. Rebuild Steps

Use the v2 worker Python environment because it has the required geometry stack.

Python:

```text
C:\Users\simon\Documents\New project\GPX_Terrain_Studio_v2\workers\.venv\Scripts\python.exe
```

Rebuild command:

```powershell
& 'C:\Users\simon\Documents\New project\GPX_Terrain_Studio_v2\workers\.venv\Scripts\python.exe' `
  'C:\Users\simon\Documents\New project\GPX_Terrain_Studio_v3_LIVE\tools\build_lille_building_block_test.py'
```

Expected runtime:

- About 30-60 seconds for the current standalone v1 rebuild on the local machine.
- The future road-combo GLB pass will be slower because every checkbox combination can be exported.

Preview server:

```powershell
& 'C:\Users\simon\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' `
  'C:\Users\simon\Documents\New project\GPX_Terrain_Studio_v3_LIVE\apps\preview_server\server.py' `
  --host 0.0.0.0 --port 8770
```

The address-selection endpoint should use a long timeout, about `900 s`, because a full rebuild can exceed `180 s`.

## 18. Required Future Generalization

The Pyramides test must become a reusable city-area generator.

Required scalable behavior:

1. Any terrain rectangle can be supplied.
2. OSM data is fetched or loaded from cache for that rectangle.
3. Roads, water, rail, and green spaces are extracted with the same global OSM tag policies.
4. Landmark database records are discovered by coordinates, not by per-city hard-coding.
5. Any database landmark inside the rectangle is automatically considered for insertion.
6. The custom landmark model replaces the matching OSM building footprint.
7. The manifest records every accepted, skipped, and failed landmark insertion.
8. Attribution from OSM and landmark model packages is preserved.
9. If a landmark has no reliable footprint match, the generator must warn instead of placing it blindly.

Recommended manifest additions:

- `landmark_database_records_considered`
- `landmark_database_records_inside_bounds`
- `landmark_replacements_inserted`
- `landmark_replacements_skipped`
- `landmark_replacements_failed`
- `landmark_replacement_matching_method`
- `landmark_attribution_reports`

This gives us a clean audit trail when the same workflow is applied outside Lille.
