# RFS V3

This is a draft of the RFS v3 specifications. It is not complete and is subject to change.

Documentation for the River Forecast System version 3, RFS v3.

- Model Version: 3.0
- Launch Date: <mark>1 March 2027</mark>
- End Date:

| Property                       | Value                                    |
|--------------------------------|------------------------------------------|
| Reference IFS Version          | 50r1                                     |
| IFS Resolution                 | O1280 grid, 0.1 degree, 9km equator      |
| Forecast Length (Lead Time)    | 15 days                                  |
| Forecast Time Step             | 3 hours                                  |
| Forecast Compute Start Time    | Typically before 0600 UTC                |
| Forecast Compute End Time      | Typically before 1200 UTC                |                         
| Retrospective Start Time       | 1940-01-01                               |
| Retrospective Update           | Daily, 00:00 UTC, available by 01:00 UTC |
| Retrospective Native Time Step | 1 hourly average                         |

## Journal Papers

- <mark>TBD</mark>

## Conceptual explanation of the model

RFS produces 4 main datasets:

1. Hydrography and Model Configs (static)
2. Retrospective simulation and derivatives
3. 15 Day Forecasts and derivatives
4. Flood extent and depth maps

RFS data is available in the following ways:

1. AWS S3 buckets
2. Code packages
    1. Python: `pyrfs`
    2. JavaScript: `jsrfs`
3. Web apps: https://hydroviewer.geoglows.org
4. Lambda functions deployable to user's cloud account (AWS, GCP, Netlify, Vercel, etc)
5. ArcGIS Living Atlas web map layer

## Migrating from v2 to v3

This is the migration guide for consumers moving from v2 to v3, and it is the only place in this document where v3 is described in terms of v2. Every other section describes v3 on its own terms, so a
reader with no v2 history can skip ahead from here.

### Renames

These account for most of the mechanical work in porting v2 code.

| v2                                                         | v3                    | Notes                                                                     |
|------------------------------------------------------------|-----------------------|---------------------------------------------------------------------------|
| `rivid` (forecasts from RAPID), `river_id` (retrospective) | `riverId`             | One name in every product, where v2 used a different spelling per product |
| `Qout`                                                     | `Q`                   | One name in every product, including the ensemble arrays                  |
| `return_period`                                            | `recurrence_interval` | Now `float32`, not int, because the axis carries a 1.5 year value         |

### Reading discharge values

- **Discharge is bitrounded `float32`. Precision is now relative rather than absolute, see [Encoding and compression](#encoding-and-compression)
- **`riverId` order is now a formal promise.** The axis is topologically sorted and identical across every product. It is not ascending id order, so an id cannot be located by binary search on the
  published array. The new `riverIndex` attribute on the streams gives a river's position directly, which avoids a lookup when downloading
- **Flow duration curves are `float32` and bitrounded** like every other discharge array. v2 stored them as `float64`
- Streams carry additional attributes, and some renamed ones, which make the data more self documenting and convenient to use. Hydrography geometry is now EPSG:3857, not EPSG:4326,
  see [Geometry storage](#geometry-storage)

### Where the data live

- All v3 data are under a single bucket, `s3://river-forecast-system/v3/`, alongside migrated v1 and v2 backups.
- Within each product, subdivisions use hive partition style names
    - Hydrography is organized in 3 digit computational group numbers `group=XXX`
    - Forecasts are organized by a sequence of year, month, day dividers: `year=YYYY/month=MM/day=DD`
    - Flood maps are organized by 1x1 degree tiles labeled by latitude and longitude: `lat=YYY/lon=XXX`
- Monthly and yearly averages are available in timeseries and timestep chunked forms in a single zarr
- The retrospective simulation is updated daily and is typically available by 01:00 UTC
- Forecast warnings are published as `alerts.csv`, formatted for CAP alerts, instead of a warnings parquet file
- Forecast flood extents are published as a single vector `fim.geo.parquet` rather than a raster set
- Reference flood maps are stored by tile as rasters
- Global mapping hydrography is published as PMTiles: streams, catchments (with a basin hierarchy for low zooms), and group outlines. <mark>A file geodatabase for Esri clients is TBD</mark>

### New in v3

- Confluences, catchments, and group outlines published per computational unit, a global attribute table (`metadata.parquet`, `metadata.zarr`) and a Pfafstetter-like basin hierarchy in `group=0`
- `riverIndex` and `upstreamCount` on every reach: the reaches upstream of any reach are the contiguous row range `[riverIndex - upstreamCount, riverIndex]` in every table and every Zarr
- Lakes and reservoirs are handled in the network itself: interior reaches are removed, inlets point at the outlet, and the outlet reach carries a traced line through the lake. There is no separate
  lakes layer
- The 1.5 year recurrence interval and a derived `annual_exceedance_probability` array on the return periods

### Discontinued in v3

- The forecast records product
- The "52nd ensemble member" IFS HRES, so the `member` axis runs 1..51 with nothing off axis. HRES was discontinued after becoming identical to the control forecast
    - [https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens](https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens)
- The https://geoglows.ecmwf.int web portal is retired

### Model changes behind the numbers

These change the values themselves rather than how they are read, so a difference between v2 and v3 at a given river is expected.

- Forecasts and retrospective simulations are tightly coupled using shared initializations
- Reduction in total stream count from about 6.8 million to about 5.45 million, for computational efficiency and accuracy, better treatment of lakes and reservoirs, and reduction in total storage
  burden. Streams in the middle of deserts, within lakes, in oceans, on small islands, in terminal watersheds under 250 km2, and reaches shorter than 2 km were removed or merged, see
  [Network simplification](#network-simplification)
- Upgrading to use the newest IFS version, 50r1
- Runoff volumes are aggregated to catchment scale on the native octahedral, or reduced gaussian, mesh native to the IFS version instead of being resampled to a uniform lat/lon grid
- Upgrade computations to use river-route (Python) instead of RAPID (user compiled Fortran) for matrix muskingum routing, with synthetic reach subdivision for greater computational stability
- Application of bitrounding to 15 bits, which compresses much better, but data are modified after computation

## Possible Changes

These are under consideration for v3 but are not committed:

- <mark>Implement non-linear routing???</mark>
- <mark>Extended range 45 day forecasts and the 3-6 month statistical outlook derived from them???</mark>
- <mark>Hydrology Early Warnings for All</mark>

## Available Products

### Organization on S3

```text
s3://river-forecast-system/
├── v1/
│   └── backups....
├── v2/
│   └── backups....
└── v3/
    ├── hydrography/
    │   ├── group=0/                                # the 0 group is the global catch-all so we don't make near-duplicate names
    │   │   ├── metadata.parquet                    # every reach's attributes, all groups concatenated, groupId retained
    │   │   ├── metadata.zarr/                      # the network walking columns as chunked arrays, for browser clients
    │   │   ├── groups.geo.parquet                  # one exact outline per group
    │   │   ├── basins_level{2..8}.geo.parquet      # Pfafstetter-like basin hierarchy, one file per level
    │   │   ├── streams.pmtiles                     # z0-11, for mapbox-gl-js, maplibre, leaflet, etc
    │   │   ├── catchments.pmtiles                  # z0-10, basins at low zooms, leaf catchments at z10
    │   │   ├── groups.pmtiles                      # z0-12, group outlines
    │   │   ├── riverNames.json                    # hand curated names of major rivers as riverIndex ranges, updated as edits accumulate
    │   │   └── streams_map_optimized.gdb.zip       # for esri map layers - not produced by the hydrography pipeline, TBD
    │   └── group=XXX/                              # all geometry epsg3857 snapped to a 1 m grid, geoarrow geoparquet 1.1
    │       ├── streams_XXX.geo.parquet             # TDX-Hydro derived - the only streams product, full attribute table
    │       ├── metadata_XXX.parquet                # the streams' attribute table without geometry, plus outlet lat/lon
    │       ├── catchments_XXX.geo.parquet          # TDX-Hydro derived - one polygon per reach
    │       ├── confluences_XXX.geo.parquet         # TDX-Hydro derived - junction points
    │       ├── boundary_XXX.geo.parquet            # the group's outline
    │       ├── params_XXX.parquet                  # for river-route - TBD, musk_k/musk_x/velocity_factor are already in the metadata
    │       └── synthetic_rating_curve.parquet      # from ARC, for routing, FIM? Q_baseflow depends on a modeled value!
    ├── retrospective/
    │   ├── hourly.zarr/
    │   ├── daily.zarr/
    │   ├── monthly.zarr/                           # holds both Q and Q_timesteps
    │   ├── yearly.zarr/                            # holds both Q and Q_timesteps
    │   ├── return-periods.zarr/
    │   ├── maximums.zarr/
    │   └── fdc.zarr/
    ├── flood-maps/
    │   └── lon=XXX/
    │       └── lat=YYY/
    │           ├── arc/
    │           │   ├── fim.tiff
    │           │   ├── velocity.tiff
    │           │   ├── depth.tiff
    │           │   ├── c2f_config.yaml             # arc/c2f config used to make this
    │           │   └── impact? DEM? burned DEM? TBD
    │           └── fldpln.zarr/
    │               ├── library/
    │               ├── streams/
    │               └── zarr.json
    ├── forecasts15/                                # 15 day @ 3 hour forecasts, lead_time coordinate alongside absolute time
    │   └── year=YYYY/
    │       └── month=MM/
    │           └── day=DD/
    │               ├── alerts.csv                  # for CAP alerts
    │               ├── maps/                       # summary files
    │               │   ├── esri_animation_tables/  # for animated forecast layer (arcgis living atlas)
    │               │   │   ├── YYYYMMDDHH.csv
    │               │   │   └── x120...
    │               │   ├── timeseries/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   ├── max-flow/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   ├── below-q95/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   └── time-to-peak/
    │               │       ├── styles.bin
    │               │       └── styles.json
    │               ├── next-init-files/
    │               │   └── group=XXX/
    │               │       └── warmstate_YYYYMMDDHHMM_groupXXX.parquet
    │               ├── discharge.zarr/
    │               └── fim.geo.parquet             # vector extent polygons, faster/smaller, single file
    └── forecasts45/                                # 1-3 month outlooks, assuming this works out
        └── year=YYYY/
            └── month=MM/
                └── day=DD/
                    ├── summary.csv
                    └── discharge.zarr/
```

### Hydrography

| Property             | Value                                                                              |
|----------------------|------------------------------------------------------------------------------------|
| Streams (reaches)    | ~5.45 million (5,452,029 in the current build)                                     |
| Computational groups | 127                                                                                |
| Source regions       | 46 TDX-Hydro regions (HydroBASINS level 2 ids, e.g. `1020000010`)                  |
| Reference product    | TDX-Hydro                                                                          |
| Preparation code     | `tdxhydro-postprocessing`, see [Hydrography preparation](#hydrography-preparation) |

Hydrography and the routing configs derived from it are divided by computational unit. A computational unit is called a **group** and is written into the bucket as a `group=XXX` hive partition. The
`group=0` partition is a global catch-all, used so that global products do not need near duplicate names of their own.

A group is a set of whole terminal watersheds: `groupId` is assigned by `outletRiverId` from a version controlled lookup table, so no reach ever drains across a group boundary and a group's files are
self contained. Every group lies inside exactly one TDX-Hydro region, and every group is one contiguous run of the global `riverIndex`. There is no group index file: `groupId` is a property of the
reach (and rides in the vector tiles), `riverIndex` and `upstreamCount` give position and extent, and `groups.geo.parquet` gives each group's outline and therefore its bounding box.

Global products, in `group=0`:

| File                             | Format     | Description                                                                                                   |
|----------------------------------|------------|---------------------------------------------------------------------------------------------------------------|
| `metadata.parquet`               | Parquet    | Every reach's attribute table, all groups concatenated in `riverIndex` order, `groupId` retained              |
| `metadata.zarr/`                 | Zarr v3    | The network walking columns of `metadata.parquet` as chunked arrays, for browser clients                      |
| `groups.geo.parquet`             | GeoParquet | One exact outline per group, dissolved from that group's catchments, one row group per row                    |
| `basins_level{2..8}.geo.parquet` | GeoParquet | Pfafstetter-like basin hierarchy, one file per level, each basin addressed by the reach it drains through     |
| `streams.pmtiles`                | PMTiles    | Global stream tiles, z0-11, for mapbox-gl-js, maplibre, leaflet, and similar clients                          |
| `catchments.pmtiles`             | PMTiles    | Global catchment tiles, z0-10, the basin hierarchy at low zooms and the leaf catchments at z10                |
| `groups.pmtiles`                 | PMTiles    | Group outline tiles, z0-12                                                                                    |
| `riverNames.json`                | JSON       | Hand curated names of major rivers, each a contiguous `riverIndex` range, with the colors a map draws them in |

Per group products, in `group=XXX`:

| File                             | Format     | Description                                                                                               |
|----------------------------------|------------|-----------------------------------------------------------------------------------------------------------|
| `streams_XXX.geo.parquet`        | GeoParquet | Stream center lines with the full attribute table, a modified copy of the TDX-Hydro region                |
| `metadata_XXX.parquet`           | Parquet    | The same rows and columns as `streams_XXX` without the geometry, for walking the network without geometry |
| `catchments_XXX.geo.parquet`     | GeoParquet | Drainage area of each reach, one polygon per reach, TDX-Hydro derived                                     |
| `confluences_XXX.geo.parquet`    | GeoParquet | Junctions of the stream network, TDX-Hydro derived                                                        |
| `boundary_XXX.geo.parquet`       | GeoParquet | The group's outline, the same geometry as its row in `groups.geo.parquet`                                 |
| `params_XXX.parquet`             | Parquet    | Routing parameters for river-route                                                                        |
| `synthetic_rating_curve.parquet` | Parquet    | Synthetic rating curves from ARC, used for routing and flood inundation mapping                           |

<mark>`params_XXX.parquet` and `synthetic_rating_curve.parquet` are not written by the hydrography pipeline. The Muskingum parameters river-route needs (`musk_k`, `musk_x`, `velocity_factor`) and the
connectivity (`riverId`, `nextRiverId`) are already columns of `metadata_XXX.parquet`, so confirm whether `params_XXX.parquet` is a separate file or a projection of the metadata.</mark>

<mark>`Q_baseflow` in the synthetic rating curves depends on a modeled value.</mark>

<mark>A file geodatabase for Esri clients (`streams_map_optimized.gdb.zip`) is not produced by the hydrography pipeline. Decide whether it is built from `streams_XXX` downstream or dropped.</mark>

There is deliberately **no global streams or catchments table** and **no lakes product**:

- Per reach geometry is published per group only. A world sized copy of it is a bulk download of what `group=XXX/` already serves, and a client that wants geometry wants one group of it. The global
  attribute table is `metadata.parquet`.
- Lakes and reservoirs are not a separate layer. Reaches inside a lake are removed from the network, the lake's inlets are pointed at its outlet, and the outlet reach's geometry becomes a traced line
  through the lake from each significant inlet, see [Lakes and reservoirs](#lakes-and-reservoirs). The lake table that drives this is an input to the pipeline, not a product.
- There is no separate connectivity table. `riverId`/`nextRiverId` are columns of `metadata_XXX.parquet`, and parquet is columnar, so a consumer that wants only those two pays for those two column
  chunks. A dedicated file was measured to be no smaller.

#### Row order, `riverIndex`, and `upstreamCount`

Every hydrography table is in one global row order, and that order is the `riverId` axis of every Zarr store. `riverIndex` is a reach's row position in that order; `upstreamCount` is the number of
reaches strictly upstream of it. The order is nested three levels deep:

1. **Groups, ascending `groupId`.** No reach drains across a group, so whole groups can be ordered freely, and doing so gives every group one contiguous `riverIndex` range. A group's own files are
   indexed by `riverIndex` minus the group's first `riverIndex`.
2. **Terminal watersheds within a group, along a Hilbert curve through their outlet points** (16 bit index on the outlet lon/lat). Watersheds are disjoint, so neighbouring basins land next to each
   other in the file, which is what keeps the geometry compressible and the tiles coherent.
3. **Reaches within a watershed, depth first post-order from the outlet, descending the largest subtree first.** Post-order emits a reach after everything upstream of it, so this is still a valid
   topological sort, and a subtree occupies a contiguous interval that ends at its root.

Three promises follow, and each is asserted by the build rather than assumed:

- **Upstream before downstream.** Every reach is ordered after every reach that drains into it.
- **Every upstream subset is one contiguous row range**, exactly `[riverIndex - upstreamCount, riverIndex]`. An upstream query is a range filter, on any table or any Zarr.
- **Every group is one contiguous row range**, so a group's rows are a slice of the global tables.

Because it is a topological order and not ascending id order, an id cannot be found by binary search on the published `riverId` array. Read `riverIndex` off the streams or metadata instead. Descending
the largest subtree first puts a reach a mean of 3.5 rows from the reach it drains into (96.5% within one 64 byte cache line), which both the compressor and the Muskingum routing kernel exploit.

`riverIndex` is positional and **not stable across rebuilds**. `riverId` is the only stable identifier; it is the TDX-Hydro `LINKNO` plus a per region offset that makes it globally unique.

#### Attribute schema

`metadata_XXX.parquet` and `streams_XXX.geo.parquet` carry the same rows in the same order; the streams add `geometry` and omit `lat`/`lon`. `group=0/metadata.parquet` is every group's metadata
concatenated with `groupId` retained. Column order puts the network walking columns first so a projected read of them is one contiguous run of column chunks.

| Column            | Type     | Description                                                                                                                      |
|-------------------|----------|----------------------------------------------------------------------------------------------------------------------------------|
| `riverId`         | int32    | Unique reach id, the TDX-Hydro `LINKNO` offset per region. The only stable identifier                                            |
| `nextRiverId`     | int32    | Downstream reach id, -1 at an outlet                                                                                             |
| `outletRiverId`   | int32    | The terminal outlet reach of this reach's watershed                                                                              |
| `riverIndex`      | int32    | Row position in the global order, see above. Positional, not stable across rebuilds                                              |
| `upstreamCount`   | int32    | Reaches strictly upstream; they are rows `[riverIndex - upstreamCount, riverIndex]`                                              |
| `strahlerOrder`   | int32    | Strahler stream order, from TDX-Hydro `strmOrder`, taking the max when reaches were merged                                       |
| `shreveOrder`     | int32    | Shreve magnitude, the upstream headwater count recomputed on the simplified network. **Not** a reach count                       |
| `USContArea`      | float64  | Contributing area at the reach's upstream end, m2, from TDX-Hydro                                                                |
| `DSContArea`      | float64  | Contributing area at the reach's downstream end, m2, from TDX-Hydro                                                              |
| `areaM2`          | float64  | The reach's own catchment area, m2, `DSContArea - USContArea` summed over every source reach merged into it                      |
| `Length`          | float64  | Reach length, m, the TDX-Hydro TauDEM length summed over merged reaches                                                          |
| `TDXHydroRegion`  | string   | Source TDX-Hydro region id                                                                                                       |
| `groupId`         | int32    | Computational group. **`group=0/metadata.parquet` only**, dropped from the per group files because the partition name carries it |
| `musk_k`          | int64    | Muskingum K, seconds, `geodesic length / velocity_factor` rounded to an integer                                                  |
| `musk_x`          | float64  | Muskingum X, constant 0.20                                                                                                       |
| `velocity_factor` | float64  | Reference velocity, m/s, `exp(0.10 ln(DSContArea) - 4.68) + 0.1`                                                                 |
| `lat`, `lon`      | float64  | Outlet point of the reach in degrees, EPSG:4326. **Metadata only**                                                               |
| `geometry`        | geoarrow | The reach line, MultiLineString, EPSG:3857. **Streams only**                                                                     |

Ids, indices, orders, and `groupId` are int32 on purpose. A parquet int64 column decodes to `BigInt` in a browser, which neither compares nor hashes equal to `Number`, so a client keying a `Map` on
ids silently matches nothing. int32 decodes to `Number`, and it is what every Zarr uses for `riverId`, so the two agree. The build range checks rather than assumes; an id scheme that outgrows int32
fails the build instead of wrapping.

<mark>`musk_k` is currently written as int64 (the one integer column not covered by the int32 enforcement). Values are seconds and comfortably fit int32; consider downcasting it for the same
`BigInt` reason.</mark>

<mark>The geodesic reach length is computed in the pipeline and used for `musk_k`, but the published `Length` column is the TDX-Hydro planar length. Decide whether to publish the geodesic length too,
or instead.</mark>

`group=0/metadata.zarr` holds the network walking subset of the same table as chunked arrays: `riverId`, `riverIndex`, `upstreamCount`, `nextRiverId`, `outletRiverId` as int32 and `lat`, `lon` as
float32, each `(riverId,)` in the global order, chunks of 10,000, blosc zstd level 5 with shuffle, Zarr v3, not consolidated. It exists so a browser client can pull one range of the network without a
parquet reader.

#### Confluences

`confluences_XXX.geo.parquet`, one row per reach that has at least one reach draining into it, in the same `riverIndex` order as the reaches, default row groups.

| Column         | Type     | Description                                                           |
|----------------|----------|-----------------------------------------------------------------------|
| `riverId`      | int32    | The reach that begins at this junction, i.e. the downstream reach     |
| `upstream_ids` | string   | Comma separated `riverId`s of the reaches meeting here                |
| `geometry`     | geoarrow | Point, EPSG:3857, the outlet point of the first listed upstream reach |

#### Catchments

`catchments_XXX.geo.parquet`, one polygon per reach, in the same order as the reaches, so one selector addresses both.

| Column       | Type     | Description                                                                                                  |
|--------------|----------|--------------------------------------------------------------------------------------------------------------|
| `riverId`    | int32    | The reach this catchment drains to                                                                           |
| `riverIndex` | int32    | The reach's row position, identical to the streams                                                           |
| `geometry`   | geoarrow | MultiPolygon, EPSG:3857, the TDX-Hydro source basins of every reach merged into this one, dissolved together |

The catchments are the TDX-Hydro `streamreach_basins` polygons dissolved along exactly the edits made to the stream network, so a catchment covers the drainage of everything that was merged into its
reach and the catchment coverage conserves area. They are then coverage simplified at 20 m with the outer boundary pinned. That is not a rendering decision: it is a few times the 3.4 m cell of the 1/9
arcsec DEM the basins were polygonised from, so it removes the raster staircase and little else, and it stays sub pixel until z13, two zooms past the deepest the tiles carry. Nothing zoom dependent is
decided in the published catchments; the tile bands are cut separately.

#### Group outlines

`boundary_XXX.geo.parquet` is a single MultiPolygon, the union of every catchment in the group, with interior holes closed except where another group stands in them. `group=0/groups.geo.parquet`
is the same 127 outlines stacked with a `groupId` column, written with **one row group per row** so a client wanting one outline fetches one row group rather than the world. The outlines are exact
rather than simplified: adjacent groups share edges instead of overlapping, and the build logs any overlapping pair.

#### Basin hierarchy

`group=0/basins_level{L}.geo.parquet` for `L` in 2..8 is a nested set of drainage basins coarser than the catchments, used to draw the catchment map at low zooms where 5.45 million leaf polygons
cannot be rendered. Level 2 is a region's whole footprint; each level below splits its parent with a Pfafstetter-like rule (within a watershed, the largest tributaries take the even digits and the
interbasins between them the odd ones; along a coast, whole terminal watersheds are grouped) with a budget of about 4x more basins per level, which is the growth a tile pyramid wants. The basin codes
are assigned once on the raw TDX-Hydro network and frozen alongside it, so they do not change between releases; each release stamps them with its own reach ids.

| Column           | Type     | Description                                                                                                                |
|------------------|----------|----------------------------------------------------------------------------------------------------------------------------|
| `riverId`        | int32    | The reach this basin drains through, so a basin is addressed exactly like a reach, and selectors written for streams apply |
| `riverIndex`     | int32    | That reach's row position                                                                                                  |
| `TDXHydroRegion` | string   | Source region                                                                                                              |
| `basinId`        | int32    | Sequential id of the basin **within its level and region**, not globally unique                                            |
| `level`          | int32    | Which level this file is                                                                                                   |
| `pfafCode`       | string   | The basin's code, `level - 2` digits, empty at level 2. A code is a prefix of the codes of the basins nested inside it     |
| `riverCount`     | int64    | Source reaches in the basin                                                                                                |
| `areaM2`         | float64  | Their total area                                                                                                           |
| `strahlerOrder`  | int32    | The largest order in the basin                                                                                             |
| `geometry`       | geoarrow | MultiPolygon, EPSG:3857, dissolved from the source basins and cut to the tolerance of the zoom band that draws this level  |

The outlet sets are strictly nested: the reach a level 4 basin drains through also names a level 5 basin, and so on down to the leaf catchment, so a selection survives zooming. `riverId` is unique
within a level, not across levels; code that merges across levels keys on `(level, riverId)`. The basins carry the raw network's codes, so a basin whose pour reach was merged away by the network
simplification is named after the surviving reach that absorbed it, and a basin with no surviving reach at all is not published.

<mark>`pfafCode` and `basinId` are unique within a region and level, not globally. Prefixing the region's own digits was designed but is not implemented.</mark>

#### River names

`group=0/riverNames.json` names the major rivers. It is the one **hand curated** product in the hydrography: TDX-Hydro carries no river names, so every entry is an editorial act — a person deciding
that a particular reach is the mouth of a river that people call something. The editable source is `network_data/river_names.csv` in the hydrography pipeline, one row per name; the published JSON is
what `scripts/extras_river_name_ranges.py` compiles that CSV into against one specific network build. The CSV is the source of truth and that script primarily restructures it: everything descriptive
is read from the CSV and republished verbatim, and only the `riverIndex` spans and the colors are worked out from the network. `scripts/extras_river_name_enrich.py` is what computes the derived
columns into the CSV in the first place — it fills blank cells and never overwrites a filled one, so a hand correction survives every rerun.

A name is stored as a **range**, not as a per reach column. The rule the table encodes is that a name covers everything upstream of the reach it is attached to, and a name attached further up
overwrites it there — so the name on a reach is the one from the smallest named span containing it, and it reads as "the name of the segment you are on, or you are on an unnamed tributary of it".
Everything upstream of a reach is one contiguous `riverIndex` interval, exactly `[riverIndex - upstreamCount, riverIndex]`, so the whole global assignment flattens to a sorted list of disjoint
intervals on a single axis. That is ~105 kB for the current 477 names against a multi-million row string column, and it is why the file can be fetched by a browser rather than joined server side.
Named spans are required to nest or miss entirely — a partial overlap would make the winner depend on row order — and the build refuses to emit a table that has one.

| Field                      | Type     | Description                                                                                                                               |
|----------------------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `rivers[]`                 | array    | One entry per name, the searchable table                                                                                                  |
| `.name`                    | string   | The river's name as published                                                                                                             |
| `.riverId`                 | int32    | The reach the name is attached to, i.e. the mouth of the named segment                                                                    |
| `.outletRiverId`           | int32    | The terminal outlet of the watershed the name sits in                                                                                     |
| `.watershed`               | string   | The name of that terminal watershed, so a tributary carries the river system it belongs to                                                |
| `.country`                 | string   | The country the mouth stands in. Editorial — see the note on disambiguation below                                                         |
| `.parent`                  | int32    | Index in `rivers[]` of the smallest named river containing this one, `null` at a river that nests in nothing                              |
| `.bbox`                    | float[4] | `[west, south, east, north]` of the named span, in degrees, over the outlet point of every reach in it                                    |
| `.lo`, `.hi`               | int32    | The named span as `riverIndex` bounds, inclusive. `hi` is the mouth reach's own `riverIndex`                                              |
| `.color`                   | int32    | Index into `palette`, resolved at build time so no two touching spans share a colour                                                      |
| `generatedAt`              | string   | ISO 8601 UTC, when this file was compiled. There is no schedule, so this is how a client tells one release from another                   |
| `palette`                  | string[] | The colors named rivers are drawn in                                                                                                      |
| `unnamed`                  | string   | The colour unnamed water keeps                                                                                                            |
| `namedReaches`             | int64    | How many reaches fall inside some named span                                                                                              |
| `first`, `bounds`, `stops` | arrays   | The same assignment precompiled as a MapLibre `step` expression over `riverIndex`: `stops[k]` holds until `bounds[k]`, -1 meaning unnamed |

Because the spans are `riverIndex` values, and `riverIndex` is positional and **not stable across rebuilds**, this file is published beside the tiles in `group=0` and is only valid against the network
release it was compiled from. A client must fetch it from the same release it draws, never bundle a copy: a stale copy keeps painting confidently onto reach numbers that have moved. `riverId` and
`outletRiverId` are the stable half of each row and are what a client should persist if it stores a name across releases.

**Coverage and currency.** The table is a curated list of rivers people look for, not a gazetteer: 477 names over 217 terminal watersheds at present — 260 of them named tributaries of another named
river — covering the large systems and their principal tributaries. There is no schedule. Names are added and corrected as edits accumulate, and the file is regenerated whenever the network is rebuilt
or a name is edited — so it may go months without changing and may change twice in a week. **Any client caching it must treat it as refreshable rather than fixed**, and no downstream product should
assume that a river absent today will stay absent or that a spelling is final. Clients that hold a local copy should revalidate on a fixed monthly boundary, the 5th at 00:00 UTC, matching the cadence
the monthly retrospective products already use, and refetch on any change to the network release. `generatedAt` is what makes that revalidation cheap: it identifies the release without the client
having to compare the body.

**Disambiguation.** The name plus the watershed does not uniquely identify a river and cannot be made to. Ten names are currently duplicated, and three of those collide on the watershed name as well:
the British and the Canadian `Severn` are distinct terminal watersheds that share a name, so are the two `Colorado`s, and the second `Yenesei` nests inside the first. `country`, `parent`, and
`bbox` exist for that reason and for no other — they are what lets a results list say which river a row is, and what lets a client frame the river it found rather than drop a pin on its mouth.

`country` is the mouth's country, which is a fact about the mouth rather than about the river, and the two part company often enough to matter: the Colorado's mouth is genuinely in Mexico and the
Nile's is genuinely in Egypt, and only one of those reads oddly as a label. It is therefore an **editorial** column. The enrichment tool fills it from a Natural Earth admin-0 join — by containment,
falling back to the nearest coast within one degree, because a river mouth sits on the coastline and the coastline in a 1:50m dataset is a generalisation of it — and then never touches a filled cell
again, so the value in the CSV is whatever a person last decided it should be. Rivers that cross many countries carry only their mouth's; there is no list of every country a span touches.

<mark>`country` is populated for all 477 rows from the automatic join, and has not yet been reviewed by hand. 139 rows were attributed to the nearest coast rather than to a polygon containing them,
and 84 sit in a different country from the middle of their own bounding box; both populations are mostly correct, but that is where an error would be. The enrichment tool prints both counts.</mark>

#### Geometry storage

Every geometry product in the bucket — streams, catchments, confluences, outlines, basins — is stored the same way.

| Property                   | Value                                                                                       |
|----------------------------|---------------------------------------------------------------------------------------------|
| CRS                        | EPSG:3857 (web mercator)                                                                    |
| Coordinate grid            | snapped to 1 m                                                                              |
| GeoParquet version         | 1.1                                                                                         |
| Geometry encoding          | geoarrow (native), **not** WKB                                                              |
| Geometry types             | streams MultiLineString, confluences Point, catchments, outlines, and basins MultiPolygon   |
| Coordinate column encoding | `BYTE_STREAM_SPLIT`, dictionary encoding off                                                |
| Compression                | zstd level 3                                                                                |
| Row group size             | 500 rows for streams, catchments, basins; 1 row for `groups.geo.parquet`; default otherwise |

These choices are one decision, not seven. A power-of-two metre grid is exactly representable in float64, so snapping to 1 m leaves ~30 trailing zero mantissa bits in every coordinate; the geoarrow
encoding then stores x and y as separate columns instead of interleaving them into WKB blobs, and `BYTE_STREAM_SPLIT` groups the resulting zero bytes so zstd can collapse them. Together they cut the
hydrography roughly 60-70%. None of the three is worth much alone — applied to unsnapped coordinates the same encoding makes files *larger*. There is no equivalent in degrees, because no useful
decimal grid (1e-7 and friends) is a power of two. zstd level 3 rather than the default 1 because the row order creates locality that is beyond snappy's window and inside zstd's: on one 352,529 reach
region, Hilbert order plus zstd is 380 MB against 1,247 MB unordered plus snappy, and read speed is flat across zstd levels so the level is purely a size against write time choice.

Snapping to a 1 m grid moves a vertex at most 0.71 m, and web mercator inflates distances by 1/cos (lat), so the true ground error is worst at the equator and finer toward the poles. The TDX-Hydro
source is a 1/9 arcsec DEM, about 3.4 m, so the snap sits well inside the resolution the data was derived from. `Length`, `areaM2`, and the contributing areas are attributes carried from the source
rather than measured on the projected geometry, so web mercator's distortion never reaches them, and the `lat`/`lon` outlet columns in the metadata stay in degrees.

Stream geometry is otherwise at source resolution: no line simplification is applied to the published streams, because a tolerance baked into the file applies at every zoom and can never be undone.
Generalizing for a zoom is tippecanoe's job when the tiles are built. The one exception is the traced line through a lake, see below.

<mark>Consumers who need geographic coordinates must reproject. The inverse mercator transform is analytic, so degrees come back to within the 1 m snap.</mark>

#### Lakes and reservoirs

A version controlled lake table lists each lake's inlet reaches and its single outlet reach. For each lake

- every reach in the interior between the inlets and the outlet is removed from the network and its catchment area (`areaM2`) folded into the outlet, so drained area is conserved;
- inlets whose contributing area (`DSContArea`) is below 100 km2 are too small to route into the lake on their own, so their whole upstream branch is absorbed into the lake as well;
- the surviving inlets point directly at the outlet, i.e. `nextRiverId` is the outlet;
- the outlet reach's geometry becomes the line traced from each significant inlet (Strahler order 4 or more, plus each lake's largest inlet) through the lake to the outlet, merged into a
  MultiLineString when there is more than one, and generalized at 100 m because it is a synthetic line across open water rather than a surveyed channel;
- the outlet reach is protected from every other simplification step, so it keeps its identity and its short length.

A lake therefore appears as one reach with a branched line through it, and the catchment of that reach is the lake plus the drainage of everything absorbed into it.

#### Network simplification

The published network is smaller than TDX-Hydro because, per region and in this order, the pipeline

1. drops whole terminal watersheds on the version controlled drop lists: no-runoff basins in the Sahara, Gobi, and Australian interior, small islands, small ocean draining watersheds, and manually
   excluded watersheds;
2. drops every remaining terminal watershed whose outlet contributing area is under 250 km2;
3. applies the lake edits above;
4. repairs zero length reaches, which the source DEM processing leaves at some confluences and coasts;
5. folds orphaned coastal order 1 reaches into the neighbour they shared a confluence with;
6. dissolves order 2 headwaters whose upstreams are all order 1 into a single reach;
7. folds order 1 reaches that join an order 2 or higher reach into a sibling at the same confluence, moving area but not geometry;
8. consolidates reaches shorter than 2 km into a neighbour that is not separated from them by a confluence, so routing is numerically stabler and fewer results are stored.

Every edit is recorded as a keeper to members mapping and replayed on the catchments, so the catchment coverage matches the published network exactly, and an original TDX-Hydro reach can be mapped to
the v3 reach that now represents it.

#### Vector tiles

Three tilesets are published in `group=0`, all built with tippecanoe from the parquet products above, so there is no second simplified copy of any geometry.

| Tileset              | Zooms | Layers                          | Notes                                                                                                                                                                                                                                                                                             |
|----------------------|-------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `streams.pmtiles`    | 0-11  | `streams`                       | Tiled per group then joined. Carries every attribute except `musk_k`, `musk_x`, `velocity_factor`, and `USContArea`. Reaches appear by Strahler order: 7+ at every zoom, 6+ from z5, 4+ from z7, 2+ from z9; order 1 reaches are not tiled at any zoom; the densest tiles drop features as needed |
| `catchments.pmtiles` | 0-10  | `catchments`, `catchment_lines` | Level 3 basins at z0-3, then one level per zoom, level 8 at z8-9, leaf catchments at z10; clients overzoom past z10. Every feature carries `riverId` and `riverIndex`, and the basin bands carry the basin columns. Region footprints (level 2) are included as lines at every zoom               |
| `groups.pmtiles`     | 0-12  | `groups`                        | The group outlines with `groupId`                                                                                                                                                                                                                                                                 |

Every catchment band is tiled twice: `catchments` holds the polygons for fills and hit testing, and `catchment_lines` holds their boundaries for strokes. Clipping a polygon to a tile has to close the
ring along the tile edge, so stroking the polygon layer draws the tile grid across the map; clipping a line does not. **Style fills from `catchments` and strokes from `catchment_lines`, never stroke
the polygon layer.** Each band's geometry is cut at a quarter pixel of the band's finest zoom, rounded to a power of two (32 m for the z10 leaf, 64 m at z9, and so on), and tippecanoe generalizes
again per zoom on top of that.

### Flood Forecast Products

The daily 15 day forecast is published under `forecasts15/`, partitioned `year=YYYY/month=MM/day=DD/`. Each day's partition holds

| File                                                                | Format     | Description                                                       |
|---------------------------------------------------------------------|------------|-------------------------------------------------------------------|
| `discharge.zarr/`                                                   | Zarr v3    | Forecasted discharge for 15 days, 50+1 ensemble                   |
| `alerts.csv`                                                        | CSV        | Alert records, formatted for CAP alerts                           |
| `fim.geo.parquet`                                                   | GeoParquet | Vector flood extent polygons for all groups                       |
| `maps/esri-animation-tables/YYYYMMDDHH.csv`                         | CSV        | Summary table per forecast timestep (120) for ArcGIS Living Atlas |
| `maps/timeseries/styles.{bin,json}`                                 | bin + JSON | Map styleset for the forecast timeseries                          |
| `maps/max-flow/styles.{bin,json}`                                   | bin + JSON | Map styleset for maximum forecasted flow                          |
| `maps/below-q95/styles.{bin,json}`                                  | bin + JSON | Map styleset for flows below the Q95 low flow threshold           |
| `maps/time-to-peak/styles.{bin,json}`                               | bin + JSON | Map styleset for time to peak                                     |
| `next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet` | Parquet    | Routing warm state per group, the initialization for the next run |

The forecast discharge store contains

1. `Q` (member, time, riverId): 3 hourly discharge for each of the 50+1 IFS forecast members
2. `Qpercentiles` (percentiles, time, riverId): 3 hourly discharge ensemble at deciles 0 (min), 10, 20 ... 50 (median) ... 90, 100 (max)
3. `Qmean` (time, riverId): 3 hourly discharge ensemble mean

Request parameters for ECMWF IFS data:

- Stream: enfo
- Types:
    - pf, perturbed forecast, 50 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid should be the native resolution of reduced gaussian grid/mesh. It should not be regridded or resampled.

Example MARS request

```text
retrieve,
class=od,
date=2025-07-15,
expver=1,
levtype=sfc,
number=1/2/3/4/5/6/7/8/9/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27/28/29/30/31/32/33/34/35/36/37/38/39/40/41/42/43/44/45/46/47/48/49/50,
param=205.128,
step=0/1/2/3/4/5/6/7/8/9/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27/28/29/30/31/32/33/34/35/36/37/38/39/40/41/42/43/44/45/46/47/48/49/50/51/52/53/54/55/56/57/58/59/60/61/62/63/64/65/66/67/68/69/70/71/72/73/74/75/76/77/78/79/80/81/82/83/84/85/86/87/88/89/90/93/96/99/102/105/108/111/114/117/120/123/126/129/132/135/138/141/144/150/156/162/168/174/180/186/192/198/204/210/216/222/228/234/240/246/252/258/264/270/276/282/288/294/300/306/312/318/324/330/336/342/348/354/360,
stream=enfo,
time=00:00:00,
type=pf,
target="output"
```

### Extended Range Forecast Products

Produced monthly on the 1st and the 15th. These may be generated a few days after the nominal date so that the retrospective initialization is available for the dates being forecasted for.

Extended range forecasts are published under `forecasts45/`, partitioned `year=YYYY/month=MM/day=DD/` on the same convention as the daily forecasts.

| File              | Format  | Description                                                         |
|-------------------|---------|---------------------------------------------------------------------|
| `discharge.zarr/` | Zarr v3 | Forecasted discharge for 45 days, 100 perturbed + 1 control members |
| `summary.csv`     | CSV     | Summary of the extended range forecast                              |

The extended range store contains

1. `Q` (member, time, riverId): daily average discharge for each of the 100+1 IFS forecast members
2. `Qpercentiles` (percentiles, time, riverId): daily discharge ensemble at deciles 0 (min), 10, 20 ... 50 (median) ... 90, 100 (max)
3. `Qmean` (time, riverId): daily discharge ensemble mean

<mark>What other derivatives or summaries should be made?</mark>

1. Expected volumes? Or could be computed on the fly, allowing for easy use for reservoir management
2. Comparisons to normal, or some sort of classification system

A statistically generated 3-6 month outlook uses the 45 day simulation together with groundwater baseflow. This is a method to calculate, not necessarily a dataset that is cached.

1. Based on historical daily discharge data from ERA5
2. So far: plots observed past 9 months, then forecast of next 3 months
3. Forecast includes median, 25-75 percentile, and wettest/driest 3-month period
4. Trying methods of scaling/stretching the historical median to see how well they forecast, in the process of deciding which metrics are most telling
5. Plans to incorporate other variables such as the 45 day forecast and groundwater

Request parameters for the 45 day forecasts:

- Stream: eefo
- Types:
    - pf, perturbed forecast, 100 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid should be the native resolution of reduced gaussian grid/mesh. It should not be regridded or resampled.

Example MARS request

```text
retrieve,
class=od,
date=2025-07-15,
expver=1,
levtype=sfc,
number=1/to/100,
param=205.128,
step=0/to/1104/by/24,
stream=eefo,
time=00:00:00,
type=pf,
target="output"
```

### Retrospective Simulation Products

| Product Type         | Time Step       | Format  | Frequency | Description                                                    |
|----------------------|-----------------|---------|-----------|----------------------------------------------------------------|
| Hourly Discharge     | hourly average  | Zarr v3 | Daily     | Hourly average simulation, native resolution                   |
| Daily Discharge      | daily average   | Zarr v3 | Daily     | Daily average simulation                                       |
| Monthly Discharge    | monthly average | Zarr v3 | Monthly   | Monthly average simulation, `Q` and `Q_timesteps` in one store |
| Yearly Discharge     | yearly average  | Zarr v3 | Yearly    | Yearly average simulation, `Q` and `Q_timesteps` in one store  |
| Maximums Discharge   | annual maximum  | Zarr v3 | Yearly    | Annual maximums from hourly and daily averages                 |
| Return Periods       |                 | Zarr v3 | Once      | Return periods from multiple methods                           |
| Flow Duration Curves |                 | Zarr v3 | Once      | Flow duration curves                                           |

Routing warm states are not stored under `retrospective/`. They are written per group with the forecast that consumes them, at
`forecasts15/year=YYYY/month=MM/day=DD/next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet`.

### Flood Maps

Flood maps are tiled, partitioned `flood-maps/lon=XXX/lat=YYY/`. Each tile holds the ARC/Curve2Flood raster products and the FLDPLN library used by the flood worker.

| File                  | Format  | Description                                                 |
|-----------------------|---------|-------------------------------------------------------------|
| `arc/fim.tiff`        | GeoTIFF | Flood inundation extent                                     |
| `arc/depth.tiff`      | GeoTIFF | Inundation depth                                            |
| `arc/velocity.tiff`   | GeoTIFF | Flow velocity                                               |
| `arc/c2f_config.yaml` | YAML    | The ARC/Curve2Flood configuration used to produce this tile |
| `fldpln.zarr/`        | Zarr v3 | Per tile FLDPLN library, read directly by the flood worker  |

<mark>Impact layers, DEM, and burned DEM per tile are still TBD.</mark>

The daily forecast also publishes vector flood extents as a single `fim.geo.parquet` per forecast, which is faster and smaller than a raster set for map clients. See
[Flood Forecast Products](#flood-forecast-products).

The internal layout of `fldpln.zarr` is documented in [Dataset Structure and Schematics](#flood-mapslonxxxlatyyyfldplnzarr).

### Web Maps

1. Daily forecasted flood maps [https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b](https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b)
2. Return period flood maps and/or forecasted flood maps (ESRI)
    1. Return period flood maps
    2. Daily forecast flood maps, from the 90th percentile of the ensemble maximum
3. Maps for the extended range forecast
    1. HydroSOS style maps showing what basins fall into different categories of wet, dry, normal, etc. Similar to what is currently produced for the retrospective data, but based on forecasts
    2. Potentially a map highlighting areas forecasted to be significantly lower than before. Similar to the highlighted streams for the daily forecast maps but focused on low flows and larger areas
       instead of individual streams
4. Charts, tables, etc. for the extended range forecast

### Summary Table

Note: All times are given in UTC.

| Product Type                | Category           | Format            | Update Frequency               | Updates Available | Size               |
|:----------------------------|:-------------------|:------------------|:-------------------------------|:------------------|:-------------------|
| Stream tiles (`group=0`)    | Model Sources      | PMTiles           | None                           | N/A               | ~3.2 GB            |
| Catchment tiles (`group=0`) | Model Sources      | PMTiles           | None                           | N/A               | ~5.5 GB            |
| Group tiles (`group=0`)     | Model Sources      | PMTiles           | None                           | N/A               | ~180 MB            |
| Global metadata (`group=0`) | Model Sources      | Parquet + Zarr v3 | None                           | N/A               | ~215 MB + ~70 MB   |
| Basin hierarchy (`group=0`) | Model Sources      | GeoParquet        | None                           | N/A               | ~1 GB              |
| Group outlines (`group=0`)  | Model Sources      | GeoParquet        | None                           | N/A               | ~120 MB            |
| River names (`group=0`)     | Model Sources      | JSON              | Irregular, as edits accumulate | N/A               | ~105 kB            |
| Hydrography (by group)      | Model Sources      | GeoParquet        | None                           | N/A               | ~15 GB all groups  |
| Routing Configs (by group)  | Model Sources      | Parquet           | None                           | N/A               |                    |
| Forecast 3-hourly Discharge | Forecasts          | Zarr v3           | Daily @ 00:00                  | 6am-12pm          | 150 GB             |
| Esri Animation Tables       | Forecasts          | CSV               | Daily @ 00:00                  | 6am-12pm          | 120 x 120 MB       |
| Map Stylesets               | Forecasts          | bin + JSON        | Daily @ 00:00                  | 6am-12pm          |                    |
| Alerts                      | Forecasts          | CSV               | Daily @ 00:00                  | 6am-12pm          | 500 MB             |
| Warm States                 | Forecasts          | Parquet           | Daily @ 00:00                  | 6am-12pm          |                    |
| Hourly Discharge            | Retrospective      | Zarr v3           | Daily @ 00:00                  | by 1am same day   | 10 TB              |
| Daily Discharge             | Retrospective      | Zarr v3           | Daily @ 00:00                  | by 1am same day   | 500 GB             |
| Monthly Average Discharge   | Retrospective      | Zarr v3           | Monthly on 5th at 00:00        | by 1am same day   | ~20 GB             |
| Yearly Average Discharge    | Retrospective      | Zarr v3           | Yearly on Jan 5 at 00:00       | by 1am same day   | ~2 GB              |
| Annual Maximums Discharge   | Retrospective      | Zarr v3           | Yearly on Jan 5 at 00:00       | by 1am same day   | ~1 GB              |
| Return Periods              | Retrospective      | Zarr v3           | None                           | N/A               |                    |
| Flow Duration Curves        | Retrospective      | Zarr v3           | None                           | N/A               |                    |
| Forecast Flood Extents      | Flood Maps         | GeoParquet        | Daily @ 00:00                  | 6am-12pm          | <5 GB              |
| Flood Map Tiles (ARC)       | Flood Maps         | GeoTIFF           | None                           | N/A               | <10 GB             |
| FLDPLN Libraries            | Flood Maps         | Zarr v3           | None                           | N/A               |                    |
| Extended Forecast Discharge | Extended Forecasts | ??                | Monthly 1 and 15               |                   | TBD ballpark 150GB |

## Dataset Structure and Schematics

Every store is **Zarr v3**. The river axis is named `riverId` and discharge is named `Q`. The chunking shown in the schematics below is a placeholder to tune.

### Datatypes

Every array gets an explicit dtype at write time, nothing is left for xarray to infer. The `encodings` dict in `generate_v3_examples_data.py` is the source of truth. `encodings_for_dataset` raises on
any variable missing from it, so a renamed or newly added variable fails the write instead of silently taking a default.

| Array                                                   | Dtype     | Bitrounded | Notes                                    |
|---------------------------------------------------------|-----------|------------|------------------------------------------|
| `riverId`                                               | `int32`   | no         | ids exceed int16, well inside int32      |
| `time`                                                  | `int32`   | no         | integer hours, see below                 |
| `member`                                                | `int32`   | no         | forecasts only, 1..51                    |
| `lead_time`                                             | `int32`   | no         | forecasts only, `units "hours"`          |
| `Q`, `Q_timesteps`                                      | `float32` | **yes**    | m3 s-1                                   |
| `daily`, `hourly`                                       | `float32` | **yes**    | maximums.zarr annual maxima              |
| `{gumbel,logpearson3,lognormal,weibull}_{hourly,daily}` | `float32` | **yes**    | return-periods.zarr, 8 arrays            |
| `max_simulated_hourly`, `max_simulated_daily`           | `float32` | **yes**    | return-periods.zarr                      |
| `Qpercentiles`, `Qmean`                                 | `float32` | **yes**    | forecasts, reduced across `member`       |
| `percentiles`                                           | `int32`   | no         | forecasts only, deciles 0..100 by 10     |
| `recurrence_interval`                                   | `float32` | no         | **float**, not int, the axis carries 1.5 |
| `annual_exceedance_probability`                         | `float32` | no         | derived `1 / recurrence_interval`        |
| `p_exceed`                                              | `int32`   | no         | fdc.zarr, percent 0..100                 |
| `hourly_annual`, `daily_annual`                         | `float32` | **yes**    | fdc.zarr flow-duration curves            |

Only discharge valued arrays are bitrounded. Do not bitround `annual_exceedance_probability`, derived from an exact axis value, or `Q_timesteps`, a rechunked copy of an already rounded `Q`. Every
value on the `recurrence_interval` axis, 1.5 included, is exact in float32, so an equality match against a literal is safe. Do not round-trip the axis through an integer type.

### Coordinate variables

- `riverId`
    - int32
    - The order of the ids is the same in every Zarr. To get this list, read the coordinate array off any store or refer to the hydrography datasets.
    - **The axis is in topological order, not ascending id order.** It is the hydrography row order described in [Row order, `riverIndex`, and
      `upstreamCount`](#row-order-riverindex-and-upstreamcount):
      groups ascending, watersheds along a Hilbert curve within a group, reaches in depth first post-order within a watershed. A river's *position* on the axis is therefore meaningful, every group and
      every upstream subset is one contiguous slice, and an id cannot be located by binary search on the array as published. That position is the `riverIndex`, and it is the join key shared by the
      vector tiles, the map style tables, the hydrography parquet, and every Zarr.
- `time`
    - int32, counted in hours, including on the daily, monthly and yearly axes, which are still expressed in hours rather than in their own step unit.
    - Left aligned time windows. That is, the corresponding value applies from the stated time step, t, until the start of the next time step, t+1.
    - Uses a unit string of style "\<interval\> since \<reference time\>", e.g. `hours since 1940-01-01T00:00:00+00:00`, with `calendar: proleptic_gregorian`.
    - **The reference time is per store, not shared.** Read `units` off the array and parse it, do not assume a common epoch. The string always carries an explicit UTC offset, which a reader must
      honor rather than letting a local time default apply.
    - `time` is written as an integer type, never as `datetime64[ns]`. Writing the latter re-encodes the integers into the datetime container, lands `units` in the attributes, and blocks any later
      rewrite of the store.
- `member`
    - int32. 1..51 for the 15 day forecast (50 perturbed + 1 control) and 1..101 for the 45 day forecast (100 perturbed + 1 control).
- `lead_time`
    - int32, units `hours`. A coordinate **on the time dimension, not a dimension of its own**. The time delta since initialization.
    - Left aligned on the same convention as `time`: **120 values**, running 0, 3, 6, ... 357, where the value 357 covers hour 357 through hour 360. The 15 day horizon is 120 intervals, not 121
      instants.
- `recurrence_interval`
    - Are exactly the float values: [1.5, 2, 5, 10, 25, 50, 100]. Float rather than integer because of 1.5. Every value is exact in float32, so equality matching against a literal is safe.
- `annual_exceedance_probability`
    - float32 on the `recurrence_interval` axis, `1 / recurrence_interval`.
- `p_exceed`
    - int32, exceedance probability in percent, 0..100. 101 values, so Q95 is row 95.
- `percentiles`
    - Are exactly the integer values: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]. Evenly spaced deciles, so 0 is the ensemble minimum, 50 the median, and 100 the maximum.

### Data variables

- Discharge
    - **Q** in every store. In the forecasts it carries a `member` dimension, there is no separate variable name for ensemble discharge.
    - **Q_timesteps** in the monthly and yearly stores: the same values as `Q`, rechunked for all-rivers-few-timesteps access (map styling) where `Q` is chunked for one-river-whole-series access
      (plots). One store serves both patterns.
    - Q values are always left aligned on their time intervals per the definition of the time coordinate variable. That is, Q is the average over the ***following*** 1 hour, 3 hours, 24 hours, 1
      calendar month, or 1 calendar year.
    - Units: cubic meters per second (m3/s), as `m3 s-1`.
    - Carries `aggregation_method` ("mean", or "max" on the maxima series) and `keepbits`.
- Ensemble summaries, forecasts only, reduced across `member` at each time step
    - **Qpercentiles** (percentiles, time, riverId): the ensemble distribution on the `percentiles` coordinate.
    - **Qmean** (time, riverId): the ensemble mean. Distinct from `Qpercentiles` at 50, which is the median. The two diverge whenever the ensemble is skewed, which is most of the time on a flood peak.
- Annual maximums, `maximums.zarr`
    - **hourly**, **daily**: the annual maximum of the retrospective series at each native step, one value per year. The series the return period fits are derived from, kept so the fits stay
      reproducible.
- Return Periods, `return-periods.zarr`
    - **max_simulated_hourly**, **max_simulated_daily**: the largest value in the retrospective record at each step. Split on the same hourly/daily axis as the fits, because the hourly record's
      maximum is by definition greater than or equal to the daily record's.
    - Each distribution is fit against both maximum series and stored as a separate array: **gumbel_hourly**, **gumbel_daily**, **logpearson3_hourly**, **logpearson3_daily**, **lognormal_hourly**,
      **lognormal_daily**, **weibull_hourly**, **weibull_daily**.
    - A distribution is always paired with the series it was fit to, so that a consumer never has to guess which one it holds.
- Flow Duration Curves, `fdc.zarr`
    - **hourly_annual**, **daily_annual** on the `p_exceed` axis. `hourly_annual[95]` is the flow the reach exceeds 95% of the time.

### Fill values and completeness

Discharge arrays are complete: every river on the `riverId` axis has a value at every step of the `time` axis, and the coordinate arrays themselves have no gaps. The fill value is `NaN`, so a NaN
indicates a real failure to be investigated, not an expected absence.

### Encoding and compression

- Write an explicit dtype on every array at store creation time. Discharge valued arrays are `float32`, coordinate and index arrays are `int32`.
- Bitround discharge to **15 keepbits**, round-to-nearest-even on the low mantissa bits. This bounds relative error at `2^-16` ≈ 1.5e-05.
- Write `keepbits` into each discharge array's attributes, and reapply the same value when appending. Bitrounding is idempotent only if keepbits is unchanged.
- Compress every array, discharge and coordinate alike, with `blosc(cname="zstd", clevel=5, shuffle="shuffle")`.
- **Blosc is the only permitted compressor.** Browser clients ship blosc alone in their WASM codec builds, so an array written with any other codec fails to decode there.
- Do not add a separate shuffle codec entry. Blosc applies shuffle inside its own frame.
- Leave `typesize` unset so each array derives it from its own itemsize.

### retrospective/hourly.zarr, daily.zarr

```text
daily.zarr/
├── zarr.json                 title, license
├── riverId/       int32   (riverId,)          chunks (all,)
├── time/          int32   (time,)             chunks (all,)          units "hours since 1940-01-01T...+00:00", proleptic_gregorian
└── Q/             float32 (time, riverId)     chunks (all_time, N)   units "m3 s-1", aggregation_method "mean", keepbits 15
                                               ^ whole series in one chunk: read pattern is one river, all time
dims: time=31602 (1940-01-01..now, daily) · riverId=N

hourly.zarr/                  identical shape/attrs, time on hourly axis — largest store
                              note the daily axis is still counted in hours, not days
```

### retrospective/monthly.zarr, yearly.zarr

```text
monthly.zarr/                 two chunkings of the same values, one store serves both access patterns
├── zarr.json
├── riverId/       int32   (riverId,)
├── time/          int32   (time,)             units "hours since 1940-01-01T00:00:00+00:00"
├── Q/             float32 (time, riverId)     chunks (all_time, N)   one river, whole series -> plots
└── Q_timesteps/   float32 (time, riverId)     chunks (few, N)        all rivers, few timesteps -> map styling
                                               ^ same values as Q, rechunked; not re-bitrounded
dims: time=1038 months · riverId=N

yearly.zarr/                  same, time = one value per year
```

### retrospective/return-periods.zarr, maximums.zarr

```text
return-periods.zarr/          no time dim — the interval is the axis. client zips recurrence_interval x gumbel_* -> {2: q, 5: q, ...}
├── zarr.json                            title, description (Gumbel/GEV-1, method of moments), license
├── riverId/                     int32   (riverId,)
├── recurrence_interval/         float32 (recurrence_interval,)   [1.5, 2, 5, 10, 25, 50, 100]
│                                        ^ float, not int — 1.5 is bankfull-ish, the most frequent alert tier
├── annual_exceedance_probability/ float32 (recurrence_interval,)   1 / recurrence_interval
├── gumbel_hourly/               float32 (recurrence_interval, riverId)      each of the four distributions is fit
├── gumbel_daily/                float32 (recurrence_interval, riverId)      against BOTH maximum series, so the
├── logpearson3_hourly/          float32 (recurrence_interval, riverId)      array name always carries the series
├── logpearson3_daily/           float32 (recurrence_interval, riverId)      it was fit to
├── lognormal_hourly/            float32 (recurrence_interval, riverId)
├── lognormal_daily/             float32 (recurrence_interval, riverId)
├── weibull_hourly/              float32 (recurrence_interval, riverId)
├── weibull_daily/               float32 (recurrence_interval, riverId)
├── max_simulated_hourly/        float32 (riverId,)                 largest value in the record, hourly series
└── max_simulated_daily/         float32 (riverId,)                 ...and daily — the hourly max is always >= it
                              there is no series-agnostic "gumbel" array — pick a series explicitly

maximums.zarr/                annual maxima the fit above is derived from, kept so it stays reproducible
├── zarr.json
├── riverId/         int32   (riverId,)
├── time/            int32   (time,)                      one value per year, still counted in hours
├── daily/           float32 (time, riverId)              aggregation_method "max", keepbits 15
└── hourly/          float32 (time, riverId)              aggregation_method "max", keepbits 15
```

### retrospective/fdc.zarr

```text
fdc.zarr/                     flow duration curves. no time dim — exceedance probability is the axis.
├── zarr.json                 title, description (percentile of the exceedance distribution), license
├── riverId/         int32   (riverId,)
├── p_exceed/        int32   (p_exceed,)              0..100 percent, 101 values -> Q95 is row 95
├── hourly_annual/   float32 (p_exceed, riverId)      chunks (all, N)   units "m3 s-1", keepbits 15
└── daily_annual/    float32 (p_exceed, riverId)      chunks (all, N)   units "m3 s-1", keepbits 15
                              ^ whole curve for one river in one chunk: the read pattern is one river,
                                every exceedance level — same shape of access as return-periods.zarr
```

`p_exceed` is *exceedance*, not a plotting position. `hourly_annual[95]` is the flow the reach exceeds 95% of the time, that is, a low flow threshold. The `below-q95` map styleset is built from
exactly that row.

### forecasts15/year=YYYY/month=MM/day=DD/discharge.zarr

```text
discharge.zarr/
├── zarr.json       title, initialization_time "2026-07-10T00:00:00Z"
├── riverId/        int32   (riverId,)                    chunks (all,)
├── member/         int32   (member,)                     1..51 — 50 perturbed + 1 control
├── time/           int32   (time,)                       chunks (all,)   units "hours since 2026-07-10T00:00:00+00:00"
├── lead_time/      int32   (time,)                       coordinate ON the time dim, NOT its own dim
│                                                         timedelta since init, units "hours" -> 0, 3, 6, ... 357
│                                                         left aligned like time: 357 covers hour 357..360, so the
│                                                         15 d horizon is 120 intervals, not 121 instants
├── percentiles/    int32   (percentiles,)                [0, 10, ... 100] deciles: 0 = min, 50 = median, 100 = max
├── Q/              float32 (member, time, riverId)       chunks (51, 120, N)   units "m3 s-1", keepbits 15
│                                                         ^ whole ensemble x whole horizon per river bucket = 1 request
├── Qpercentiles/   float32 (percentiles, time, riverId)  chunks (11, 120, N)   member-wise reduction of Q
└── Qmean/          float32 (time, riverId)               chunks (120, N)       NOT Qpercentiles[50] — mean != median
dims: member=51 · time=120 (15 d @ 3 h) · percentiles=11 · riverId=N

forecasts45/.../discharge.zarr    same layout, longer + coarser time
```

### flood-maps/lon=XXX/lat=YYY/fldpln.zarr

`fldpln.zarr` is not gridded and is not read with xarray. It is a flat, sorted, per river contiguous set of run arrays sliced by offsets carried in the group attributes, read by the flood worker
directly.

```text
fldpln.zarr/                  per-tile FLDPLN library
├── zarr.json                 attributes carry the index:
│                               schemaVersion  "tiles-1.0"   (reader rejects non tiles-1.x)
│                               grid           { gRow0, gCol0, ... }   tile origin in the global grid
│                               rivers         { comid[], visitStart[], visitCount[],   -> streams/ rows
│                                                pixStart[], pixCount[],                -> library/ per-pixel rows
│                                                relStart[], relCount[] }               -> library/ per-relation rows
├── streams/                  one row per stream-pixel visit, in baked headwater->outlet path order
│   ├── fsp_local/  (visit,)         flood source pixel index, tile-local
│   ├── row/ col/   (visit,)         pixel position, tile-local (+ grid origin -> global)
│   ├── bed/        (visit,)         bed elevation
│   ├── q_baseflow/ (visit,)         below this Q the reach is unflooded
│   ├── q/          (visit, stage)   discharge ladder (30-pt rating curve)
│   └── wse/        (visit, stage)   water-surface elevation per ladder step
└── library/                  per floodable pixel, then per (pixel, source) relation
    ├── pix_row/    (pix,)           floodplain pixel position, tile-local
    ├── pix_col/    (pix,)
    ├── fill_mm/    (pix,)   uint16  mm, lossless quantization -> client fround(mm/1000)
    ├── rel_count/  (pix,)           how many relation rows below belong to this pixel
    ├── fsp_local/  (rel,)           which stream pixel floods it
    └── dtf_mm/     (rel,)   uint16  depth-to-flood threshold, mm

kernel:  depth(fpp) = max over relations (DoF - DTF) + fill      no raster ships, bed elevation cancels out
```

## Input Datasets

### ECMWF IFS

See the MARS requests in [Flood Forecast Products](#flood-forecast-products) and [Extended Range Forecast Products](#extended-range-forecast-products).

### ERA5 and ERA6

<mark>TBD</mark>

## Implementation Details

### Project Organization

This suite expects a machine with a certain directory structure: a home directory with subdirectories for IFS, ERA5, forecasts which has subdirectories for each YMD, and retrospective. This is the
working layout on the compute machine. The final products it produces are uploaded to the layout described in [Organization on S3](#organization-on-s3). The computational unit is called a group there
and in the published data. The scripts and intermediate files below still use the older VPU naming.

```text
/$HOME
    /ifs
        yyyymmdd.grib
    /era5
        yyyymmdd.nc
    /forecasts
        /YYYYMMDD
            /vpus
                /vpu=101
                    /volumes
                        volumes_$vpu_$ens.nc  # 1 per ensemble member
                    /discharge
                        discharge_$vpu_$ens.nc  # 1 per ensemble member
                    nces_avg_$vpu.nc  # 1 per VPU, ensemble average
                    map_tables/
                        YYYYMMDDHH.parquet  # 1 per timestep of the forecast
            # Final products, uploaded to forecasts15/year=YYYY/month=MM/day=DD/
            discharge.zarr
            alerts.csv
            fim.geo.parquet
            maps/
                esri_animation_tables/
                    YYYYMMDDHH.csv  # 1 for each timestep of the forecast
                timeseries/
                max-flow/
                below-q95/
                time-to-peak/
            next-init-files/
                /group=XXX
                    warmstate_YYYYMMDDHHMM_groupXXX.parquet
        # For example
        /20250101
        /20250102
        /20250103
        /20250104
        /20250105
    /retrospective
        hourly.zarr
        daily.zarr
        monthly.zarr
        yearly.zarr
        maximums.zarr
        return-periods.zarr
        fdc.zarr
```

### Hydrography preparation

The hydrography is built once per release, not daily, by the `tdxhydro-postprocessing` repository (`scripts/pipeline.sh`). It turns the raw TDX-Hydro stream and basin geopackages into the products
in [Hydrography](#hydrography). Two directories, two environment variables:

```text
$TDXHYDRO_ROOT/                         the raw TDX-Hydro geoparquet the pipeline reads, an input rather than an output
    TDX_streamnet_<region>_01.parquet
    TDX_streamreach_basins_<region>_01.parquet
    global_basins/                      the frozen basin hierarchy, generated once from the raw data and shared by every release
$RFS_DATA_ROOT/
    hydrography/group=<id>/             the published dataset, uploaded as is to s3://river-forecast-system/v3/hydrography/
    hydrography-scratchfiles/           regions/, pmtiles/, logs/ — per region intermediates no consumer needs
```

Version controlled inputs live in the repository's `network_data/`: `groupIds_table.csv` (`outletRiverId` to `groupId`), `lake_table.csv` (inlet, outlet, lake id, endorheic flag, trace flag),
`dropped_watersheds/*.csv` (outlets to remove), and `tdxhydro_splits/` (the per region id offsets and the duplicated watersheds to drop where regions overlap).

The steps, in order. Steps 3 and 4 run per region in parallel; everything else is global.

| # | Script                    | Scope                | Produces                                                                                                                                                                                                                                                                                                                                                                   |
|---|---------------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | `1_translate_tdxhydro.py` | per region, once     | Raw geopackages to geoparquet. Offsets `LINKNO` by a per region constant so ids are globally unique, adds the geodesic length, region id, and outlet lon/lat, drops watersheds duplicated between overlapping regions, and nodes the source basin coverage so later dissolves are exact                                                                                    |
| 2 | `2_global_basins.py`      | all regions, once    | The frozen basin hierarchy: Pfafstetter-like codes on the raw network, then the source basins dissolved into one polygon set per level (2..8), each cut to its zoom band, holes closed. Kept beside the raw data, not rebuilt per release                                                                                                                                  |
| 3 | `3_simplify_streams.py`   | per region           | `streams_*`, `metadata_*`, `confluences_*` in the region's scratch directory. Applies the [network simplification](#network-simplification) and [lake edits](#lakes-and-reservoirs), assigns `groupId`, computes the Muskingum parameters, sets the nested-set row order, and records every edit in `mods/*.json`                                                          |
| 4 | `4_create_catchments.py`  | per region           | `catchments_*`: the source basins redirected along step 3's edits and dissolved into one polygon per surviving reach, projected, coverage simplified at 20 m, snapped to 1 m, in the published row order                                                                                                                                                                   |
| 5 | `5_concatenate_global.py` | all regions          | Orders the groups by `groupId`, stamps the global `riverIndex` by offset arithmetic, checks that no reach drains across a group, that `riverId` is globally unique, and that the nested-set property holds; writes `group=0/metadata.parquet` and `metadata.zarr`; splits every region's tables into `group=XXX/`, dropping `groupId`; writes the leaf catchment tile band |
| 6 | `6_publish_basins.py`     | all regions          | Stamps the frozen basins with this release's `riverId`/`riverIndex` (through the keeper map replayed from `mods/`), writes `group=0/basins_level{L}.geo.parquet` and the basin tile bands, and dissolves each group's catchments into `boundary_XXX.geo.parquet` and `group=0/groups.geo.parquet`                                                                          |
| 7 | `tile_streams.sh`         | per group, then join | `streams.pmtiles`: one tippecanoe run per group, largest first, joined with `tile-join`                                                                                                                                                                                                                                                                                    |
| 8 | `tile_catchments.sh`      | per band             | `catchments.pmtiles`: every basin level and the leaf band tiled twice, polygons and boundary lines, then joined                                                                                                                                                                                                                                                            |
| 9 | `tile_groups.sh`          | global               | `groups.pmtiles`                                                                                                                                                                                                                                                                                                                                                           |

Every step is idempotent at the granularity of its output files: a step whose outputs exist exits successfully without rewriting them, so a rebuild after a change touches only what depends on it.
Reordering the rows (step 5) never requires rerunning the simplification (step 3) or the catchments (step 4); it permutes rows and derives integers.

Optionally, `extras_identify_id_map.py` writes a two column table mapping every original TDX-Hydro reach in every region, about 16 million, to the v3 `riverId` that now represents it, or null when it
was dropped or is in a region v3 does not cover.

### Log/Status Feeds

In addition to writing logging messages to disc, status information is sent to the following locations:

- Teams channel webhook
- AWS CloudWatch logs

### Summary of computational steps

Each day, the following steps are performed in order:

#### Phase 0: Preparation

1. Set environment variables
    1. YMD - The date of the day to be processed, usually today. In YYYYMMDD format.

#### Phase 1: Download runoff data

Runoff data are cached. Check if the data are available, otherwise download them.

1. Download the latest ECMWF IFS runoff grid/mesh data for the date specified by YMD.
2. Download the latest ERA5 runoff grid/mesh data for the date specified by YMD.

#### Phase 2: Daily forecast computations

1. VPU level computations. Parallelize these jobs by VPU, but do not change the task order.
    1. Calculate catchment level volumes (python/calculate_catchment_volumes.py)
    2. Route the volumes (python/route.py)
    3. Concatenate ensemble members into a single file (bash/concat_member_discharges.sh)
    4. Generate a table of summarized flows used to style the web map layer (python/generate_vpu_map_tables.py)
    5. Issue alerts by filtering the map summary tables (TODO)
2. Concatenate the VPU level results. Tasks may be computed in any order and/or simultaneously.
    1. Concatenate the VPU level routed discharge. Makes a Zarr dataset. (python/concatenate_vpu_discharge.py)
    2. Concatenate the VPU level map tables. Makes a directory of CSVs. (python/concatenate_vpu_map_tables.py)
    3. Concatenate the VPU level alerts (TODO)

#### Phase 3: Update retrospective simulation

1. Download the latest day's ERA5 runoff data
2. Calculate catchment level volumes
3. Route the volumes, initialized from the last time step of the retrospective simulation.
4. Resample the hourly discharge to daily average and, when necessary, monthly and yearly averages.

#### Phase 4: Synchronize forecast initialization to retrospective simulation

1. Get the initialization from the last retrospective simulation updates
2. Get the first 24 hours of forecasted catchment volumes for the 5 days between the last retrospective timestep and the present date.
3. Reroute the forecasted volumes, but initialize at the last retrospective timestep.
4. Calculate the ensemble average of the rerouted forecasted volumes

#### Phase 5: Export results to S3 archives

All exports land under `s3://river-forecast-system/v3/`, see [Organization on S3](#organization-on-s3).

1. From the daily forecast computations, to `forecasts15/year=YYYY/month=MM/day=DD/`
    1. Routed discharge - `discharge.zarr`
    2. Esri animation tables and map stylesets - `maps/`
    3. Alerts - `alerts.csv`
    4. Vector flood extents - `fim.geo.parquet`
    5. Routing warm states per group - `next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet`
2. From the retrospective simulation update, to `retrospective/`
    1. Routed discharge in hourly, daily, monthly, yearly averages - Zarr
3. From the rerouted forecasted volumes
    1. Rerouted forecasted discharge, 5 days only - Zarr

#### Phase 6: Cleanup

1. Delete forecast directories dated __older than 5 days__
2. Delete runoff data dated __older than 5 days__

## Changelog
