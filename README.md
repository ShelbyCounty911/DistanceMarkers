# Distance Markers ETL (FME Form)

This FME Form workspace collects public distance-marker datasets, standardizes them into a shared schema, generates alternate search names, and publishes the results to ArcGIS Online.

## Sources

The workspace currently uses public ArcGIS REST services for:

- Tennessee DOT mile markers
- FRA rail mileposts
- USACE river mile markers
- Shelby County 911 ESZ polygons
- Shelby County 911 trail markers

Current source filters:

- **TDOT Mile Markers**: `COUNTY=79`
- **FRA Rail Posts**: `STCYFIPS=47157`
- **USACE River Mile Markers**: `RIVER_NUMB=21`
- **SC911 ESZ**: `1=1`

## Outputs

The workspace writes to these ArcGIS Online layers/tables:

- **DistanceMarkers** — core marker features
- **DistanceMarker_AltNames** — alternate names for road, rail, and river markers
- **TrailMarker_AltNames** — alternate names for trail markers
> Note: trail markers are stored in [a separate layer](https://scecd.maps.arcgis.com/home/item.html?id=3b8ff43116c04ddab019af4c40e87d92) by design.

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

This schema follows version 3 of the [NENA Standard for NG9-1-1 GIS Data Model](https://cdn.ymaws.com/www.nena.org/resource/resmgr/standards/standards1/NENA-STA-006.3_NG9-1-1_GIS_D.pdf)

Current standardized marker types include:

- `Road`
- `Railroad`
- `River`

### Parses TDOT road marker text
The TDOT branch performs the most custom transformation work. It:

- filters to relevant sign records
- parses `COMMENTS`
- splits compound text into elements
- normalizes route and direction abbreviations
- builds a standardized road marker label

Examples of normalization include:

- `N` → `North`
- `S` → `South`
- `E` → `East`
- `W` → `West`
- `SR...` → `State Route ...`
- `I...` → `Interstate ...`

> Note: the 2-element TDOT parsing path is currently skipped because the source values are too inconsistent to map cleanly.

### Spatially filters river markers
River mile markers are spatially constrained using the SC911 ESZ layer:

- ESZ polygons are dissolved
- the result is buffered by **5 miles**
- USACE river markers are clipped to that buffered area
- only features inside the buffered area are kept

### Generates alternate search names
The workspace creates alternate-name records for locator/search support by:

- building a list of spelling and wording variations
- exploding the list into one record per alternate name
- removing geometry
- writing the result to dedicated alternate-name tables

Examples include variations using:

- `Mile`
- `Mile Marker`
- `MM`
- abbreviated vs expanded route names
- reversed marker/name ordering

### Deduplicates selected branches
Duplicate handling is source-specific:

- **road markers**: deduplicated by `ShelbyID`
- **rail markers**: no duplicates expected by design
- **river markers**: no duplicates expected by design

## Workspace structure

### TDOT Mile Markers
Parses and standardizes road mile marker text from `COMMENTS`, then creates alternate-name records.

### FRA Rail Posts
Maps `MILEPOST` values into the common schema and creates alternate-name records.

### USACE River Mile Markers
Filters river markers spatially using buffered ESZ geometry, standardizes the schema, and creates alternate-name records.

### SC911 ESZ
Creates dissolved and buffered support geometry for river filtering.

### SC911 Trail Markers
Standardizes trail marker attributes, removes duplicate core records, and creates alternate-name records.

## Dependencies

- **FME Form**
- configured ArcGIS Online web connections
- public ArcGIS REST endpoints used by the custom source transformers
- ArcGIS feature service writer support used by the workspace

## Maintenance notes

When updating this workspace, review:

- upstream source availability
- source schema changes
- ArcGIS Online destination schema
- TDOT parsing assumptions
- alternate-name generation rules
- river spatial filter assumptions
- `ShelbyID` logic used for joins and deduplication

## Notes

- The TDOT parsing logic is highly customized and regex-heavy.
- `ShelbyID` is used as the core join key between the main marker output and alternate-name outputs.
- The workspace uses separate outputs for core marker features and alternate-name records.
- Hosted ArcGIS Online schema changes may require writer schema refresh or feature type re-import.

## License / data usage

This repository contains FME workspace logic only.

Usage rights and attribution for source datasets depend on the original data publishers. Review the licensing and attribution requirements of each upstream service before redistribution or republication.