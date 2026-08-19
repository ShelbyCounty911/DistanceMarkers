# Distance Markers ETL (FME Form)

This workspace collects multiple public distance-marker datasets, standardizes their schema and labels, and publishes the results to an ArcGIS Online feature layer named **DistanceMarkers**.

## Overview

The workspace consolidates distance marker data from several public REST sources into a single output layer:

- Tennessee DOT mile markers
- FRA rail mileposts
- USACE river mile markers
- Shelby County 911 ESZ polygons used to spatially filter river markers

The output is written to an **Esri ArcGIS Online Feature Service**.

## What the workspace does

### 1. Ingests source data from public REST endpoints
The workspace uses custom transformers to request features from public ArcGIS REST services with source-specific SQL where clauses.

Current source filters include:

- **TDOT Mile Markers**: `COUNTY=79`
- **FRA Rail Posts**: `STCYFIPS=47157`
- **USACE River Mile Markers**: `RIVER_NUMB=21`
- **SC911 ESZ**: `1=1`

### 2. Cleans and normalizes source attributes
For each source, the workspace exposes needed attributes and removes REST/query artifacts such as:

- `_http_status_code`
- `_response_body`
- pagination fields
- JSON helper attributes

### 3. Parses roadway sign text into structured fields
For the TDOT source, the workspace performs fairly detailed text parsing on the `COMMENTS` field:

- removes unnecessary tokens like `MILE`
- splits compound text on semicolons
- counts list elements
- routes features into different parsing streams depending on element count
- further splits text on colons where needed
- removes regex-unsafe/special characters
- normalizes abbreviations such as:
  - `E` → `East`
  - `W` → `West`
  - `N` → `North`
  - `S` → `South`
  - `SR-` / `SR` → `State Route`
  - `I-` → `Interstate`

### 4. Builds search-friendly alternate labels
For road-based markers, the workspace creates alternate naming variations to improve fuzzy matching / search behavior, such as different expansions of route abbreviations.

Examples of derived fields include:

- `Label`
- `AltLabel1`
- `AltLabel2`
- `AltLabel3`
- `AltLabel4`
- `AltLabel5`

### 5. Spatially filters river mile markers
River mile markers are clipped to a buffered geography derived from the **SC911 ESZ** layer:

- ESZ polygons are dissolved
- dissolved geometry is buffered by **5 miles**
- USACE river markers are clipped to that buffered area
- only features on the `INSIDE` output are retained

### 6. Standardizes output schema
Each source is mapped into a common destination schema centered on:

- `Type`
- `Marker`
- `MarkerExt`
- `Name`
- `Label`
- `AltLabel1`–`AltLabel5`

Examples of standardized `Type` values include:

- `Road`
- `Railroad`
- `River`

### 7. Merges all streams and writes to ArcGIS Online
The processed source streams are joined and written to a single ArcGIS Online output feature type:

- **Writer format:** Esri ArcGIS Online Feature Service
- **Feature type / layer name:** `DistanceMarkers`

---

## Workspace structure

The workspace is organized into several logical branches:

### TDOT Mile Markers branch
Main purpose:
- parse sign text from `COMMENTS`
- normalize route/direction naming
- create standardized labels and alternate labels

Key transformers:
- `AttributeExposer`
- `AttributeRemover`
- `Tester`
- `StringReplacer`
- `AttributeSplitter`
- `ListElementCounter`
- `TestFilter`
- `AttributeCreator`
- `AttributeManager`

### FRA Rail Posts branch
Main purpose:
- expose `MILEPOST`
- map rail mileposts into the common schema

Key output values:
- `Type = Railroad`
- `Name = Railpost`

### USACE River Mile Markers branch
Main purpose:
- expose `name`
- spatially constrain records using dissolved/buffered ESZ geometry
- map into the common schema

Key output values:
- `Type = River`
- `Name = Mississippi River`

### SC911 ESZ support branch
Main purpose:
- create a dissolved study area
- buffer it by 5 miles for river marker filtering

Key transformers:
- `Dissolver`
- `Bufferer`
- `Clipper`

---

## Output schema

The final output layer is intended to support user-facing label/search behavior across multiple distance marker types.

Typical output attributes:

| Attribute | Description |
|---|---|
| `Type` | Marker category such as Road, Railroad, or River |
| `Marker` | Primary marker value |
| `MarkerExt` | Marker extension / decimal / suffix where available |
| `Name` | Standardized route/feature name |
| `Label` | Preferred display label |
| `AltLabel1`–`AltLabel5` | Alternate labels for search/fuzzy matching |

> Note: Some sources intentionally leave certain alternate labels null because the source values are too inconsistent to map reliably.

---

## Destination

The workspace writes to an ArcGIS Online feature service using a configured web connection.

---

## Dependencies

- **FME Form**
- Access to the configured **ArcGIS Online web connection**
- Availability of the public ArcGIS REST endpoints referenced in the custom source transformers

---

## Notes and assumptions

- The TDOT parsing logic is **highly customized** to the current source text patterns.
- Some branches explicitly skip overly inconsistent patterns rather than forcing exception-heavy logic.
- The river branch depends on the SC911 ESZ support layer to spatially constrain results.
- Because this writes to ArcGIS Online, target schema changes in the hosted layer may require writer schema refresh/import.

---

## Source endpoints represented in the workspace

This repository currently uses public ArcGIS REST-based sources for:

- TDOT mile markers
- FRA rail mileposts
- USACE river mile markers
- SC911 ESZ polygons

---

## Future Work

1. **Document expected output schema in the workspace Description/Help**
4. **Add validation/testing for unmatched parsing cases**
5. **Log counts by source and by parsing branch**
6. **Externalize route normalization rules** if naming patterns continue to grow
7. Add additional repo infrastructure:
   - `docs/` — optional design notes, schema notes, sample outputs
   - `samples/` — optional sample source payloads or expected output extracts

---

## Maintenance

When updating this workspace, review:

- source endpoint availability
- schema changes in upstream feature services
- ArcGIS Online destination schema
- parsing assumptions in the TDOT branch
- spatial filter assumptions in the river branch

---

## License / data usage

This repository contains FME workspace logic only.  
Usage rights and attribution for source datasets depend on the original data publishers.

Please review the licensing and attribution requirements of each upstream public service before redistribution or republication.
