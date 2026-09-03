# Distance Markers ETL (FME Form)

This FME Form workspace collects public distance-marker datasets, standardizes them into a shared schema, generates alternate search names, and publishes the results to hosted ArcGIS feature services.

## Sources

The workspace currently uses public ArcGIS REST services for:

- Tennessee DOT mile markers
- FRA rail mileposts
- FRA rail lines
- USACE river mile markers
- Shelby County 911 ESZ polygons
- Shelby County 911 trail markers

Current source filters:

- **TDOT Mile Markers**: `COUNTY=79`
- **FRA Rail Posts**: `STCYFIPS=47157`
- **FRA Rail Lines**: `STCNTYFIPS=47157`
- **USACE River Mile Markers**: `RIVER_NUMB=21`
- **SC911 ESZ**: `1=1`
- **SC911 Trail Markers**: `1=1`

## Outputs

The workspace writes to these ArcGIS feature service layers/tables:

- **DistanceMarkers** — core marker features
- **DistanceMarker_AltNames** — alternate names for road, rail, and river markers
- **TrailMarker_AltNames** — alternate names for trail markers

> Note: trail markers currently generate alternate-name output only. The workspace does not currently write a core trail marker feature layer.

## What the workspace does

### Standardizes multiple marker types
The workspace maps several source datasets into a shared marker model with values such as:

- `DiscrpAgID`: `Discrepancy Agency ID`
- `DateUpdate`: `Date Updated`
- `NGUID`: `NENA Globally Unique ID`
- `DM_Unit`: `Distance Marker Unit of Measurement`
- `DM_Value`: `Distance Marker Measurement Value`
- `DM_Rte`: `Distance Marker Route Name`
- `DM_Type`: `Distance Marker Route Type`
- `DM_Ind`: `Distance Marker Indicator`
- `DM_Label`: `Distance Marker Label`
- `Area`: city name derived from ESZ polygons

This schema aligns to the distance-marker structure used by the workspace for downstream publishing and locator support.

Current standardized marker types in the workspace include:

- `Road`
- `Railroad`
- `River`
- `Trail`

### Parses TDOT road marker text
The TDOT branch performs the most custom transformation work. It:

- filters to relevant sign records where:
  - `MUTCD = TN-45A`
  - `COMMENTS` has a value
- parses `COMMENTS`
- splits compound text into elements
- routes records based on element count
- normalizes route and direction abbreviations
- builds a standardized road marker label and NGUID
- handles special alphanumeric mile-marker cases through a dedicated custom transformer

Examples of normalization include:

- `N` → `North`
- `S` → `South`
- `E` → `East`
- `W` → `West`
- `SR...` / `SR-...` → `State Route ...`
- `I...` / `I-...` → `Interstate ...`

> Note: the 2-element TDOT parsing path is intentionally skipped because the source values are too inconsistent to map cleanly.

### Special handling for alphanumeric TDOT markers
Some TDOT markers include alphanumeric values that do not fit cleanly into the numeric `DM_Value` schema. The workspace handles these records in a dedicated **Alphanumeric Markers** custom transformer after initial TDOT parsing.

#### Overview
The Distance Marker Locator engine requires a strict `Double` data type for distance measure values. While the majority of the dataset consists of uniformly spaced numeric markers, certain corridors contain intermediate or legacy alphanumeric markers such as `1A`, `1B`, and `1C`.

To prevent schema validation errors without introducing collisions from dummy numeric values, these alphanumeric markers are encoded into sub-decimal offsets (`x.00y`) before being written to the locator-supporting output.

#### FME transformation logic
The conversion is handled dynamically in FME using parsed marker components:

1. **Test Condition:**  
   `@Value(_marker_pt1) CONTAINS_REGEX [a-zA-Z]`

2. **Value Expression:**  
   `@ReplaceRegularExpression(@Value(_marker_pt1),[a-zA-Z],"",caseSensitive=FALSE).00@Value(_marker_pt2)`

3. **Error Routing:**  
   Features that do not satisfy the expected parsing conditions are routed through a dedicated QA path for manual review.

In the current workspace implementation:

- records with a valid numeric `DM_Value` continue directly
- records with null `DM_Value` are passed to the **Alphanumeric Markers** custom transformer
- the custom transformer dynamically assigns a zero-padded sub-decimal suffix so the result still conforms to the numeric schema

#### Encoding strategy and examples

- **Source label fields** retain the original alphanumeric text for display, labeling, and search behavior.
- **`DM_Value`** stores the converted numeric placeholder used to satisfy schema requirements.

| Raw Input (`_marker_pt1`, `_marker_pt2`) | Source Label | Converted `Double` Value | Description |
| :--- | :--- | :--- | :--- |
| Base numeric, no alpha | `1` | `1.000` | Standard base mile marker |
| `1A`, `1` | `1A` | `1.001` | Alpha suffix `A` mapped to `.001` offset |
| `1B`, `2` | `1B` | `1.002` | Alpha suffix `B` mapped to `.002` offset |
| `1C`, `3` | `1C` | `1.003` | Alpha suffix `C` mapped to `.003` offset |
| `1D`, `4` | `1D` | `1.004` | Alpha suffix `D` mapped to `.004` offset |
| `1E`, `5` | `1E` | `1.005` | Alpha suffix `E` mapped to `.005` offset |

> Note: these sub-decimal values are schema-enforcement placeholders for distance locator indexing and do **not** represent exact ground measurements. Human-facing applications and locator labeling should continue to use the original text label.

### Enriches markers with city values from ESZ polygons
The ESZ layer is used for point-in-polygon clipping and city attribution. For TDOT, FRA, USACE, and trail marker branches:

- source markers are clipped against Shelby County ESZ polygons
- `CITY` is exposed from the polygon layer
- `CITY` is transformed into title case and stored as `Area`

For TDOT, FRA, and trail markers, both `INSIDE` and `OUTSIDE` outputs continue downstream, so clipping is primarily being used to accumulate `CITY` where available rather than to discard unmatched records.

### Spatially filters river markers
River mile markers are spatially constrained using the SC911 ESZ layer:

- ESZ polygons are merged into a hull
- the hull is buffered by **5 miles**
- USACE river markers are clipped against that buffered area
- only the `INSIDE` stream is used for core processing

> Note: this river filtering branch clips only. It does not accumulate polygon attributes in the buffered clip stage.

### Spatially filters rail markers using rail-line buffers
Rail mileposts are also spatially refined using an FRA rail line support layer:

- FRA rail lines are filtered to Shelby County
- source rail lines expose `RROWNER1`
- lines are buffered by **5 feet**
- FRA rail posts are clipped against those buffered rail lines
- the resulting features are then standardized and used to create output features

This provides a tighter spatial constraint than ESZ-only clipping for rail mileposts.

### Generates alternate search names
The workspace creates alternate-name records for locator/search support by:

- building lists of spelling and wording variations
- exploding each list into one record per alternate name
- removing geometry
- writing the result to dedicated alternate-name tables

Examples include variations using:

- abbreviated and expanded route names
- full and abbreviated directions
- `Mile`
- `Mile Marker`
- `Marker`
- `Milepost`
- `Mile Post`
- `Post`
- `MM`
- trail naming variants such as `Trail` and `Trail Marker`

The alternate-name generation is source-specific:

- **road markers**: route/direction spelling variants
- **rail markers**: railroad owner and shorthand naming variants
- **river markers**: river/abbreviation naming variants
- **trail markers**: marker/trail wording variants

### Deduplicates selected branches
Duplicate handling is source-specific:

- **road markers**: deduplicated by `NGUID`
- **trail markers**: deduplicated by `NGUID`
- **rail markers**: no explicit duplicate filter in the main branch
- **river markers**: no explicit duplicate filter in the main branch

For road markers, only unique features proceed to the core output and alternate-name generation branches.

## Workspace structure

### TDOT Mile Markers
Parses and standardizes road mile marker text from `COMMENTS`, derives route/direction/mile values, handles alphanumeric markers, creates core output features, and generates alternate-name records.

### FRA Rail Posts
Clips rail mileposts to ESZ polygons for city attribution, further constrains them using buffered FRA rail lines, maps `MILEPOST` values into the common schema, and creates alternate-name records.

### FRA Rail Lines
Provides buffered rail geometry used to spatially validate and constrain FRA rail milepost processing.

### USACE River Mile Markers
Filters river markers spatially using buffered ESZ support geometry, standardizes the schema, and creates alternate-name records.

### SC911 ESZ
Provides support geometry for:
- city attribution via point-in-polygon clipping
- river marker clipping via hull creation and buffering

### SC911 Trail Markers
Standardizes trail marker attributes, removes duplicate records by `NGUID`, and creates alternate-name records.

## Custom transformers

The workspace uses embedded custom transformers for source access and special-case processing:

- **GET from REST** custom transformer instances are used to retrieve source data from ArcGIS REST endpoints.
- **Alphanumeric Markers** handles TDOT marker records whose parsed marker values would otherwise violate the numeric schema requirements of `DM_Value`.

Because edits to a custom transformer apply to every instance of that transformer definition, source retrieval and special handling logic should be reviewed carefully before modification.

## Dependencies

- **FME Form**
- configured ArcGIS web connections
- public ArcGIS REST endpoints used by the custom source transformers
- ArcGIS feature service writer support used by the workspace
- the `safe.esri-agol` package dependency

## Maintenance notes

When updating this workspace, review:

- upstream source availability
- source schema changes
- destination feature service schema
- TDOT parsing assumptions and regex behavior
- alternate-name generation rules
- river buffer assumptions
- rail-line buffer assumptions
- `NGUID` uniqueness logic
- `CITY` / `Area` attribution behavior
- embedded custom transformer definitions

## Notes

- The TDOT parsing logic is highly customized and regex-heavy.
- The TDOT branch uses multiple parsing paths based on split element count.
- Road and trail duplicate handling is based on `NGUID`, not source object identity.
- Alternate-name outputs are non-spatial records created by exploding generated name lists and removing geometry.
- Hosted feature service schema changes may require writer schema refresh or feature type re-import.

## License / data usage

This repository contains FME workspace logic only.

Usage rights and attribution for source datasets depend on the original data publishers. Review the licensing and attribution requirements of each upstream service before redistribution or republication.