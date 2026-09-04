# Distance Markers ETL (FME Form)

This workspace ingests public REST-based distance marker datasets, standardizes them into a shared locator-friendly schema, generates alternate name tables for search, and writes the outputs to ArcGIS Feature Services.

---

## Overview

The workspace processes four source themes:

- **TDOT road mile markers**
- **FRA rail mileposts**
- **USACE river mile markers**
- **SC911 trail markers**

All streams are normalized into a common distance-marker model where possible, enriched with `CITY` from SC911 ESZ polygons, deduplicated by calculated `NGUID`, and written to ArcGIS-hosted outputs.

The workspace produces:

- **DistanceMarkers**: core distance-marker features
- **DistanceMarker_AltNames**: alternate names for road, rail, and river markers
- **TrailMarker_AltNames**: alternate names for trail markers

Trail markers do **not** write to the core `DistanceMarkers` layer in this workspace.

---

## Data Flow

    [Public REST Sources]
       -> [Attribute cleanup / exposure]
       -> [Spatial enrichment and clipping]
       -> [Source-specific normalization]
       -> [NGUID-based deduplication]
       -> [Core output and alternate-name expansion]
       -> [ArcGIS Feature Services]

---

## Source Inputs

| Source | Filter | Key Exposed Fields | Notes |
|---|---|---|---|
| TDOT_Mile_Markers | `COUNTY=79` | `COMMENTS`, `MUTCD`, `LOG_MILE` | Uses `COMMENTS` text parsing to derive route, direction, and marker value. Only `MUTCD=TN-45A` with non-empty `COMMENTS` is processed. |
| FRA_Rail_Posts | `STCYFIPS=47157` | `MILEPOST` | Rail post points are later associated with buffered FRA rail lines to derive/validate operator naming. |
| USACE_River_Mile_Markers | `RIVER_NUMB=21` | `name` | River markers are constrained using a buffered hull derived from ESZ polygons. |
| SC911_Trail_Markers | `1=1` | `SIGN`, `TRAIL`, `NAME`, `LABEL` | Used to build trail marker records and trail alt-name table only. |
| SC911_ESZ | `1=1` | `CITY` | Used for clipping and city/area attribution. |
| FRA_Rail_Lines | `STCNTYFIPS=47157` | `RROWNER1` | Buffered 5 ft and used to spatially support rail naming/association. |

The REST fetch logic is encapsulated in embedded custom transformers named **GET from REST**.

---

## Spatial Processing

### ESZ enrichment

TDOT, FRA, USACE, and trail points are clipped against `SC911_ESZ` polygons. Both `INSIDE` and `OUTSIDE` outputs are preserved for TDOT, FRA, and trail streams, then passed forward after exposing `CITY`. This means ESZ is used mainly for attribute enrichment rather than strict exclusion in those streams.

### River boundary constraint

For USACE river markers, ESZ polygons are accumulated into a **convex hull**, then buffered **5 miles**, and used as a second clipping boundary. Only the `INSIDE` output of this second clip is processed further. This is a true spatial filter.

### Rail corridor constraint

FRA rail lines are buffered **5 feet** and used to clip FRA rail post points. The `INSIDE` output continues to processing. This acts as a corridor-based validation/association step.

---

## Core Schema

The workspace standardizes records into fields including:

- `NGUID`
- `DiscrpAgID`
- `DateUpdate`
- `DM_Unit`
- `DM_Value`
- `DM_Rte`
- `DM_Type`
- `DM_Ind`
- `DM_Label`
- `Area`

`Area` is derived from ESZ `CITY` using title case formatting.

Core source mappings:

- **Road**: `DM_Type=Road`, `DM_Unit=Mile`, `DM_Ind=P`
- **Railroad**: `DM_Type=Railroad`, `DM_Unit=Mile`, `DM_Ind=P`
- **River**: `DM_Type=River`, `DM_Unit=Mile`, `DM_Ind=L`
- **Trail**: `DM_Type=Trail`, `DM_Unit=Unknown`, `DM_Ind=P`

---

## TDOT Parsing Logic

TDOT is the most complex stream. It parses `COMMENTS` using:

- string cleanup
- split on semicolon
- element counting
- branching by element count
- split on colon for 3- and 4-part cases
- regex-based normalization of route and direction tokens

Important details:

- **2-element cases are intentionally skipped** via a dead-end junction because the source strings are too inconsistent.
- Route normalization supports patterns like `SR`, `State Route`, `I`, and `Interstate`.
- Direction normalization supports both full and abbreviated values (`North`/`N`, etc.).
- Safe-to-handle punctuation is stripped before regex logic runs.
- Failed parsing cases are routed to a holding junction labeled as pending error handling.

---

## Alphanumeric Marker Handling

For TDOT, `DM_Value` must remain numeric. If the parsed marker value is null after the main parsing logic, the feature is sent into the embedded custom transformer **Alphanumeric_Markers**.

That transformer receives:

- `DM_Value`
- `_marker_pt1`
- `_marker_pt2`

Its purpose is to convert alphanumeric mile marker patterns into numeric-compatible values so the output schema stays valid. The workspace comments indicate this supports schema compliance for locator use and prevents alphanumeric values from breaking the numeric `DM_Value` field.

---

## NGUID and Deduplication

Deduplication is done with `DuplicateFilter` transformers keyed on **`NGUID`**, not source object IDs.

Examples of NGUID construction:

- **TDOT road**: built from parsed direction, route, and marker parts
- **FRA rail**: uppercased combination of rail owner code/name and `DM_Value`
- **USACE river**: uppercased combination of primary river name and `DM_Value`
- **Trail**: derived from `LABEL` with spaces replaced by hyphens

Deduplication occurs before alternate-name explosion and before final core writes.

---

## Alternate Name Generation

The workspace creates multiple alternate-name tables by source/type, then explodes them into one row per alternate name using `ListExploder`, strips geometry, cleans helper attributes, and writes the tabular outputs.

Two major alt-name categories are generated:

### Alternate street names

Used for road, rail, river, and trail “street-style” locator joins:

- Highway variants with full/abbreviated route and direction forms
- Rail operator full and short forms
- River synonyms such as `Mississippi`, `MS`, `Miss`
- Trail synonyms such as base trail name and abbreviated forms like `GG`

### Alternate POI names

Used for search-oriented distance marker phrases:

- `MM`, `Mile`, `MP`, `Milepost`, `Marker`, `Post`, `Trail Marker`
- Different ordering patterns, including marker-first and name-first variants
- Tight and spaced variants in some trail and road cases

Outputs:

- **DistanceMarker_AltNames** receives road, rail, and river alt names
- **TrailMarker_AltNames** receives trail alt names

---

## Writers and Connections

The workspace writes to ArcGIS Feature Services through the `safe.esri-agol` package using configured web connections:

- **ArcGIS Portal**
- **ArcGIS Online**

Current writer feature types are:

- `DistanceMarkers`
- `DistanceMarker_AltNames`
- `TrailMarker_AltNames`

---

## Embedded Custom Transformers

The workspace uses embedded custom transformers, including:

- **GET from REST**: handles dynamic query ingestion from public ArcGIS REST endpoints
- **Alphanumeric_Markers**: converts alphanumeric marker cases into numeric-compatible `DM_Value` outputs

Because these are custom transformers in FME Form, edits to their definitions apply to all instances in the workspace.

---

## Maintenance Notes

- Error handling for failed TDOT parse paths appears incomplete; failed features are currently routed to a pending junction.
- Bookmark names separate major stages such as source cleanup, clipping, and alternate-name generation.
- The workspace relies heavily on helper attributes such as `_primname`, `_altname*`, `_direction_*`, `_marker_pt*`, and `_sign_text*`.
- Road and trail deduplication is based strictly on calculated `NGUID` values rather than source IDs.
- USACE is the only stream in this workspace that is hard-filtered by the secondary spatial boundary after enrichment.