# RFS V3

This is a draft of the RFS v3 specifications. It is not complete and is subject to change.

Documentation for the River Forecast System version 3, RFS v3.

- Model Version: 3.0
- Launch Date: 1 March 2027

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
2. CloudFront Distribution
3. Code packages
    1. Python: `pyrfs`
    2. JavaScript: `jsrfs`
4. Web apps:
    1. https://apps.geoglows.org/rfs-datastore
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

- **`riverId` is the first dimension now, not `time`.** Every discharge array is `(riverId, time)` where v2 was `(time, riverId)`, so an indexed read becomes `Q[river, time]`. Code that reads through
  xarray by dimension name needs no change; code that indexes by position does. See [Chunking and sharding](#chunking-and-sharding)
- **Discharge is bitrounded `float32`. Precision is now relative rather than absolute, see [Encoding and compression](#encoding-and-compression)
- **`riverId` order is now a formal promise.** The axis is topologically sorted and identical across every product. It is not ascending id order, so an id cannot be located by binary search on the
  published array. The new `riverIndex` attribute on the streams gives a river's position directly, which avoids a lookup when downloading
- **Flow duration curves are `float32` and bitrounded** like every other discharge array. v2 stored them as `float64`
- Streams carry additional attributes, and some renamed ones, which make the data more self documenting and convenient to use. Hydrography geometry is now EPSG:3857, not EPSG:4326,
  see [Geometry storage](#geometry-storage)

### New in v3

- Confluences, catchments, and region outlines published per region, plus a global attribute table (`metadata.parquet`, `metadata.zarr`)
- The edits the pipeline made to TDX-Hydro are published alongside the network as `region=<id>/mods/`, so a source reach can be traced to the v3 reach that represents it
- `riverIndex` and `upstreamCount` on every reach: the reaches upstream of any reach are the contiguous row range `[riverIndex - upstreamCount, riverIndex]` in every table and every Zarr
- Lakes and reservoirs are handled in the network itself: interior reaches are removed, inlets point at the outlet, and the outlet reach carries a traced line through the lake. There is no separate
  lakes layer
- The 1.5 year recurrence interval and a derived `annual_exceedance_probability` array on the return periods

### Discontinued in v3

- The forecast records product
- The "52nd ensemble member" IFS HRES, so the `member` axis runs 1..51 with nothing off axis. HRES was discontinued after becoming identical to the control forecast
    - <https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens>
- The https://geoglows.ecmwf.int web portal is retired

### Model changes behind the numbers

These change the values themselves rather than how they are read, so a difference between v2 and v3 at a given river is expected.

- Forecasts and retrospective simulations are tightly coupled using shared initializations
- Reduction in total stream count from about 6.8 million to about 4.92 million, for computational efficiency and accuracy, better treatment of lakes and reservoirs, and reduction in total storage
  burden. Streams in the middle of deserts, within lakes, in oceans, on small islands, in terminal watersheds under 250 km2, and reaches shorter than 2 km were removed or merged, see
  [Network simplification](#network-simplification)
- Upgrading to use the newest IFS version, 50r1
- Runoff volumes are aggregated to catchment scale on the native octahedral, or reduced gaussian, mesh native to the IFS version instead of being resampled to a uniform lat/lon grid
- Upgrade computations to use river-route (Python) instead of RAPID (user compiled Fortran) for matrix muskingum routing, with synthetic reach subdivision for greater computational stability
- Application of bitrounding to 13 keepbits, which compresses much better, but data are modified after computation

## Data Organization

- All v3 data are under a single bucket, `s3://river-forecast-system-v3/`, served through the CloudFront CDN at `https://d2bu4ozwm6rcbq.cloudfront.net`.
- Within each product, subdivisions use hive partition style names
    - Hydrography is organized by HydroBASINS level 2 region, `region=<id>`, e.g. `region=1020000010`, with the products that span every region in `hydrography/global/`
    - Routing configurations are organized by the same regions in a tree of their own, `routing/region=<id>`, derived from the hydrography but not published in it
    - Forecasts are organized by a sequence of year, month, day dividers: `year=YYYY/month=MM/day=DD`
    - Flood maps are organized by 1x1 degree tiles labeled by latitude and longitude: `lat=YYY/lon=XXX`
- The runoff forcings the router reads are kept in the bucket under `forcings/`: ERA5 one Zarr per year, `era5/YYYY.zarr`, and IFS one directory of GRIB files per initialization,
  `ifs/YYYYMMDDHH/`
- Monthly and yearly averages are available in timeseries and timestep chunked forms in a single zarr
- The retrospective simulation is updated daily and is typically available by 01:00 UTC
- Forecast warnings are published as `alerts.csv`, formatted for CAP alerts, instead of a warnings parquet file
- Forecast flood extents are published as a single vector `fim.geo.parquet` rather than a raster set
- Reference flood maps are stored by tile as rasters
- Global hydrography for custom web maps is published as PMTiles, see [Vector tiles](#vector-tiles).
- Global hydrography a file geodatabase for Esri clients is TBD</mark>

## Available Products

### Product short names

Each product has a short name, unique within this model version and stable across releases. It is
how the product is referred to outside the bucket layout: the data store addresses a product's page
as `/rfs-data-store/datasets/v3/<short name>`, which makes the name a permalink other tools
can build without asking the site. Names are lower case and hyphenated, and say what the product is
rather than where it is filed — a product listed under two categories still has one name.

| Short name                 | Product                  | Categories                | Format              |
|----------------------------|--------------------------|---------------------------|---------------------|
| `streams`                  | Stream centerlines       | Hydrography               | GeoParquet          |
| `catchments`               | Catchment boundaries     | Hydrography               | GeoParquet          |
| `confluences`              | Confluence points        | Hydrography               | GeoParquet          |
| `lakes`                    | Lakes                    | Hydrography               | GeoParquet          |
| `metadata`                 | Stream attributes        | Hydrography               | Parquet, Zarr       |
| `routing-configs`          | River-route configs      | Model Configs             | Parquet, netCDF     |     
| `retrospective-hourly`     | Hourly discharge         | Retrospective             | Zarr                |
| `retrospective-daily`      | Daily discharge          | Retrospective             | Zarr                |
| `retrospective-monthly`    | Monthly discharge        | Retrospective             | Zarr                |
| `retrospective-yearly`     | Yearly discharge         | Retrospective             | Zarr                |
| `return-periods`           | Return periods           | Retrospective             | Zarr                |
| `flow-duration-curves`     | Flow duration curves     | Retrospective             | Zarr                |
| `annual-maximums-hourly`   | Annual maximums (hourly) | Retrospective             | Zarr                |
| `annual-maximums-daily`    | Annual maximums (daily)  | Retrospective             | Zarr                |
| `forecast-15day`           | 15-day ensemble forecast | Forecasts                 | Zarr                |
| `forecast-flood-maps`      | Forecast flood maps      | Flood maps, Forecasts     | GeoParquet, GeoTIFF |
| `return-period-flood-maps` | Return period flood maps | Flood maps, Retrospective | GeoTIFF             |
| `fldpln-libraries`         | FLDPLN libraries         | Flood maps                | Zarr, PMTiles       |

### Organization on S3

The bucket is `s3://river-forecast-system-v3/`. The same tree is served over HTTPS by the CloudFront CDN at `https://d2bu4ozwm6rcbq.cloudfront.net`, so `s3://river-forecast-system-v3/<key>` is read
as `https://d2bu4ozwm6rcbq.cloudfront.net/<key>`.

```text
s3://river-forecast-system-v3/
├── hydrography/
│   ├── global/                                 # the products that span every region
│   │   ├── metadata.parquet                    # every reach's attributes, all regions concatenated in riverIndex order
│   │   ├── metadata.zarr/                      # the network walking columns as chunked arrays, for browser clients
│   │   ├── streams.pmtiles                     # z0-11, for mapbox-gl-js, maplibre, leaflet, etc
│   │   ├── regions.pmtiles                     # z0-12, region outlines - NOT CURRENTLY BUILT
│   │   ├── tdxhydro_to_v3_id_map.parquet       # optional, every source TDX-Hydro reach to the v3 reach representing it
│   │   └── streams_map_optimized.gdb.zip       # for esri map layers - not produced by the hydrography pipeline, TBD
│   └── region=<id>/                            # all geometry epsg3857 snapped to a 1 m grid, geoarrow geoparquet 1.1
│       ├── streams_<id>.geo.parquet            # TDX-Hydro derived - the only streams product, full attribute table
│       ├── metadata_<id>.parquet               # the streams' attribute table without geometry, plus outlet lat/lon
│       ├── catchments_<id>.geo.parquet         # TDX-Hydro derived - one polygon per reach
│       ├── confluences_<id>.geo.parquet        # TDX-Hydro derived - junction points
│       ├── boundary_<id>.geo.parquet           # the region's outline
│       ├── mods/                               # the edits made to TDX-Hydro, the provenance of this network
│       │   ├── lake_edits.json
│       │   ├── zero_length_streams.json
│       │   ├── coastal_orphans.json
│       │   ├── headwater_dissolves.json
│       │   ├── branches_to_prune.json
│       │   └── short_consolidations.json
│       └── synthetic_rating_curve.parquet      # from ARC, for routing, FIM? Q_baseflow depends on a modeled value! - TBD
├── routing/                                    # routing configurations, derived from hydrography/ but not by its pipeline
│   └── region=<id>/                            # the regions of hydrography/, rivers in the same riverIndex order
│       ├── routing.parquet                     # Muskingum k/x and connectivity, for river-route
│       ├── gridweights_ERA5_<id>.nc            # runoff grid to catchment weights, one per forcing grid, the standard format
│       └── gridweights_ERA5_<id>.parquet       # an extra copy of the ERA5 weights, in the layout jsrr reads fastest
├── forcings/                                   # the runoff the router reads, see Input Datasets
│   ├── era5/
│   │   └── YYYY.zarr/                          # Zarr v3, ro only, chunks 16 x 16 on lat/lon and the whole year on time
│   └── ifs/
│       └── YYYYMMDDHH/                         # one directory per forecast initialization
│           └── <filename>.grib                 # the IFS GRIB files for that initialization
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
└── forecasts15/                                # 15 day @ 3 hour forecasts, lead_time coordinate alongside absolute time
    └── year=YYYY/
        └── month=MM/
            └── day=DD/
                ├── alerts.csv                  # for CAP alerts
                ├── maps/                       # summary files
                │   ├── esri_animation_tables/  # for animated forecast layer (arcgis living atlas)
                │   │   ├── YYYYMMDDHH.csv
                │   │   └── x120...
                │   ├── timeseries/
                │   │   ├── styles.bin
                │   │   └── styles.json
                │   ├── max-flow/
                │   │   ├── styles.bin
                │   │   └── styles.json
                │   ├── below-q95/
                │   │   ├── styles.bin
                │   │   └── styles.json
                │   └── time-to-peak/
                │       ├── styles.bin
                │       └── styles.json
                ├── next-init-files/
                │   └── group=XXX/
                │       └── warmstate_YYYYMMDDHHMM_groupXXX.parquet
                ├── discharge.zarr/
                └── fim.geo.parquet             # vector extent polygons, faster/smaller, single file
```

### Hydrography

| Property            | Value                                                                              |
|---------------------|------------------------------------------------------------------------------------|
| Streams (reaches)   | ~4.92 million (4,917,183 in the current build)                                     |
| Partition           | 47 TDX-Hydro regions (HydroBASINS level 2 ids, e.g. `1020000010`)                  |
| Terminal watersheds | 12,446                                                                             |
| Reference product   | TDX-Hydro                                                                          |
| Preparation code    | `tdxhydro-postprocessing`, see [Hydrography preparation](#hydrography-preparation) |

Hydrography is divided by **HydroBASINS level 2 region**, written into the bucket as a `region=<id>` hive partition, with the products that span every region in `hydrography/global/`.

Global products, in `hydrography/global/`:

| File                            | Format  | Description                                                                              |
|---------------------------------|---------|------------------------------------------------------------------------------------------|
| `metadata.parquet`              | Parquet | Every reach's attribute table, all regions concatenated in `riverIndex` order            |
| `metadata.zarr/`                | Zarr v3 | The network walking columns of `metadata.parquet` as chunked arrays, for browser clients |
| `streams.pmtiles`               | PMTiles | Global stream tiles, z0-11, for mapbox-gl-js, maplibre, leaflet, and similar clients     |
| `regions.pmtiles`               | PMTiles | Region outline tiles, z0-12. see [Vector tiles](#vector-tiles)                           |
| `tdxhydro_to_v3_id_map.parquet` | Parquet | Optional. Every source TDX-Hydro reach mapped to the v3 `riverId` that now represents it |

Per region products, in `region=<id>/`:

| File                             | Format     | Description                                                                                |
|----------------------------------|------------|--------------------------------------------------------------------------------------------|
| `streams_<id>.geo.parquet`       | GeoParquet | Stream center lines with the full attribute table, a modified copy of the TDX-Hydro region |
| `metadata_<id>.parquet`          | Parquet    | The same rows and columns as `streams_<id>` without the geometry                           |
| `catchments_<id>.geo.parquet`    | GeoParquet | Drainage area of each reach, one polygon per reach, TDX-Hydro derived                      |
| `confluences_<id>.geo.parquet`   | GeoParquet | Junctions of the stream network, TDX-Hydro derived                                         |
| `boundary_<id>.geo.parquet`      | GeoParquet | The region's outline                                                                       |
| `mods/*.json`                    | JSON       | The edits made to the source TDX-Hydro, see [Modification records](#modification-records)  |
| `synthetic_rating_curve.parquet` | Parquet    | Synthetic rating curves from ARC, used for routing and flood inundation mapping            |

The routing configurations derived from this network are published apart from it, see [Routing Configurations](#routing-configurations).

#### Row order, `riverIndex`, and `upstreamCount`

Every hydrography table is in one global row order, and that order is the `riverId` axis of every Zarr store. `riverIndex` is a reach's row position in that order; `upstreamCount` is the number of
reaches strictly upstream of it.

**The ordering is topological first.** Every reach lands after every reach that drains into it. Everything below is a tie-break, consulted only where topology constrains nothing. The global order is
the regions concatenated in **ascending level 2 region number** — that is the whole of it, there is no coarser key and nothing to look up — and within a region the order is nested two levels deep:

1. **Terminal watersheds, along a Hilbert curve through their outlet points** (16 bit index on the outlet lon/lat). Watersheds are disjoint components, so their order among themselves is
   unconstrained, and putting neighboring basins next to each other in the file is what keeps the geometry compressible and the tiles coherent.
2. **Reaches within a watershed, depth first post-order from the outlet, descending the largest subtree first.** Post-order emits a reach after everything upstream of it, so this is still a valid
   topological sort, and a subtree occupies a contiguous interval that ends at its root. Descending the largest subtree first is the exact minimizer of total gather distance over all post-order
   linearizations, not a heuristic: a child sits one plus the total size of every sibling subtree emitted after it ahead of the parent it feeds, minimized independently at every junction.

Three promises follow, and each is asserted by the build rather than assumed:

- **Upstream before downstream.** Every reach is ordered after every reach that drains into it.
- **Every upstream subset is one contiguous row range**, exactly `[riverIndex - upstreamCount, riverIndex]`. An upstream query is a range filter, on any table or any Zarr.
- **Every region is one contiguous row range**, so a region's rows are a slice of the global tables and a routing engine can treat a region file as a dense array whose local index is
  `riverIndex - riverIndexStart`. The same holds for every terminal watershed.

Because it is a topological order and not ascending id order, an id cannot be found by binary search on the published `riverId` array. Read `riverIndex` off the streams or metadata instead.
Descending the largest subtree first puts a reach a mean of 3.5 rows from the reach it drains into, against 135.3 unordered, and the 99th percentile at 28 rows against 1,451, measured over all 3.89
million edges of the global network. Both the compressor and the Muskingum routing kernel exploit that locality.

`riverIndex` is positional and **not stable across rebuilds**. `riverId` is the only stable identifier; it is the TDX-Hydro `LINKNO` plus a per region offset that makes it globally unique.

#### Attribute schema

`metadata_<id>.parquet` and `streams_<id>.geo.parquet` carry the same rows in the same order; the streams add `geometry` and omit `lat`/`lon`. `global/metadata.parquet` is every region's metadata
concatenated, the same columns. Column order puts the network walking columns first so a projected read of them is one contiguous run of column chunks.

| Column            | Type     | Description                                                                                |
|-------------------|----------|--------------------------------------------------------------------------------------------|
| `riverId`         | int32    | Unique reach id, the TDX-Hydro `LINKNO` offset per region.                                 |
| `nextRiverId`     | int32    | Downstream reach id, -1 at an outlet                                                       |
| `outletRiverId`   | int32    | The terminal outlet reach of this reach's watershed                                        |
| `riverIndex`      | int32    | Row position in the global order, see above.                                               |
| `upstreamCount`   | int32    | Reaches strictly upstream; they are rows `[riverIndex - upstreamCount, riverIndex]`        |
| `strahlerOrder`   | int32    | Strahler stream order, from TDX-Hydro `strmOrder`, taking the max when reaches were merged |
| `shreveOrder`     | int32    | Shreve magnitude, the upstream headwater count recomputed on the simplified network.       |
| `USContArea`      | float64  | Contributing area at the reach's upstream end, m2, from TDX-Hydro                          |
| `DSContArea`      | float64  | Contributing area at the reach's downstream end, m2, from TDX-Hydro                        |
| `areaM2`          | float64  | The reach's own catchment area, m2, `DSContArea - USContArea`                              |
| `Length`          | float64  | Reach length, m, the TDX-Hydro TauDEM length summed over merged reaches                    |
| `TDXHydroRegion`  | string   | Source TDX-Hydro region id                                                                 |
| `musk_k`          | int64    | Muskingum K, seconds, `geodesic length / velocity_factor` rounded to an integer            |
| `musk_x`          | float64  | Muskingum X, constant 0.20                                                                 |
| `velocity_factor` | float64  | Reference velocity, m/s, `exp(0.10 ln(DSContArea) - 4.68) + 0.1`                           |
| `lat`             | float64  | Outlet point of the reach in degrees, EPSG:4326.                                           |
| `lon`             | float64  | Outlet point of the reach in degrees, EPSG:4326.                                           |
| `geometry`        | geoarrow | The reach line, MultiLineString, EPSG:3857. **Streams only**                               |

#### Confluences

`confluences_<id>.geo.parquet`, one row per reach that has at least one reach draining into it, in the same `riverIndex` order as the reaches, row groups of 2,000.

| Column         | Type     | Description                                                           |
|----------------|----------|-----------------------------------------------------------------------|
| `riverId`      | int32    | The reach that begins at this junction, i.e. the downstream reach     |
| `upstream_ids` | string   | Comma separated `riverId`s of the reaches meeting here                |
| `geometry`     | geoarrow | Point, EPSG:3857, the outlet point of the first listed upstream reach |

#### Catchments

`catchments_<id>.geo.parquet`, one polygon per reach, in the same order as the reaches, so one selector addresses both.

| Column       | Type     | Description                                                                                                  |
|--------------|----------|--------------------------------------------------------------------------------------------------------------|
| `riverId`    | int32    | The reach this catchment drains to                                                                           |
| `riverIndex` | int32    | The reach's row position, identical to the streams                                                           |
| `geometry`   | geoarrow | MultiPolygon, EPSG:3857, the TDX-Hydro source basins of every reach merged into this one, dissolved together |

The catchments are the TDX-Hydro `streamreach_basins` polygons dissolved along exactly the edits made to the stream network, so a catchment covers the drainage of everything that was merged into its
reach and the catchment coverage conserves area. They are then coverage simplified at 20 m with the outer boundary pinned. That is not a rendering decision: it is a few times the 3.4 m cell of the
1/9 arcsec DEM the basins were polygonized from, so it removes the raster staircase and little else, and it stays sub pixel until z13, two zooms past the deepest the tiles carry. Nothing zoom
dependent is decided in the published catchments.

#### Region outlines

`boundary_<id>.geo.parquet` is a single MultiPolygon, the union of every catchment in the region, with interior holes exactly where the release dropped watersheds. The outlines are
exact rather than simplified: adjacent regions share edges instead of overlapping, and the build logs any overlapping pair.

#### Geometry storage

Every geometry product in the bucket — streams, catchments, confluences, outlines — is stored the same way.

| Property                   | Value                                                                                                                    |
|----------------------------|--------------------------------------------------------------------------------------------------------------------------|
| CRS                        | EPSG:3857 (web mercator)                                                                                                 |
| Coordinate grid            | snapped to 1 m                                                                                                           |
| GeoParquet version         | 1.1                                                                                                                      |
| Geometry encoding          | geoarrow (native), **not** WKB                                                                                           |
| Geometry types             | streams MultiLineString, confluences Point, catchments and outlines MultiPolygon                                         |
| Coordinate column encoding | `BYTE_STREAM_SPLIT`, dictionary encoding off                                                                             |
| Compression                | zstd level 3                                                                                                             |
| Row group size             | 500 rows for streams and catchments; 2,000 for metadata and confluences; pyarrow's default for `global/metadata.parquet` |

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
5. folds orphaned coastal order 1 reaches into the neighbor they shared a confluence with;
6. dissolves order 2 headwaters whose upstreams are all order 1 into a single reach;
7. folds order 1 reaches that join an order 2 or higher reach into a sibling at the same confluence, moving area but not geometry;
8. consolidates reaches shorter than 2 km into a neighbor that is not separated from them by a confluence, so routing is numerically stabler and fewer results are stored.

Every edit is recorded as a keeper to members mapping and replayed on the catchments, so the catchment coverage matches the published network exactly, and an original TDX-Hydro reach can be mapped to
the v3 reach that now represents it. Those records are published, see below.

#### Modification records

`region=<id>/mods/` holds the edits the pipeline made to the source TDX-Hydro. They are published rather than kept as build scratch, because the v3 network is a **modification of a previous dataset**
and these files are its provenance: they are the only record of which source reach a published reach absorbed, and nothing in the published network can reconstruct them. A reader holding the dataset
can answer "which v3 reach now carries this TDX-Hydro reach's water" without the build tree that produced it.

| File                        | Shape                                 | Written by                                                   |
|-----------------------------|---------------------------------------|--------------------------------------------------------------|
| `lake_edits.json`           | `{outlet: {delete: [...], ...}}`      | The lake edits, interior reaches folded into the lake outlet |
| `zero_length_streams.json`  | `{case1\|case2\|case3: {ids: [...]}}` | Zero length reach repair, by case                            |
| `coastal_orphans.json`      | `{keeper: [members]}`                 | Orphaned coastal order 1 reaches                             |
| `headwater_dissolves.json`  | `{keeper: [members]}`                 | Order 2 headwater dissolves                                  |
| `branches_to_prune.json`    | `{keeper: [members]}`                 | Pruned order 1 branches                                      |
| `short_consolidations.json` | `{keeper: [members]}`                 | Reaches shorter than 2 km folded into a neighbour            |

Every id is a source TDX-Hydro reach id in the same namespace as `riverId`, so the files join directly against the published network. Each reach is removed at most once across every edit, so the
union of the maps cannot collide. A keeper may itself be a member of a later edit, so chains must be followed transitively to a terminal survivor; the edits only ever fold a reach downstream, so the
chains form a DAG and the walk terminates. Three outcomes are possible for a source id and a consumer has to tell them apart: it is published unchanged, it resolves through a chain to a keeper, or it
resolves to nothing because it was removed outright — a dropped watershed, a sub-250 km2 terminal, a zero length reach. Only the last is an error, and only the consumer knows whether it is fatal.

`global/tdxhydro_to_v3_id_map.parquet` is that resolution precomputed for every source reach in every region, about 16 million rows, `riverId` null where the reach has no v3 representation. It is
optional and is not written by default.

<mark>The `mods/` files are raw JSON, ~165 MB uncompressed across the release, in directories where everything else is zstd geoparquet tuned for range fetches. They cannot be range-fetched and a
consumer wanting the survivor map parses all 282 of them. A single parquet per region — `{member, keeper, editKind}`, dictionary encoded — would be a fraction of the size and read with the same
tooling as the rest of the partition. Deferred, not urgent.</mark>

#### Vector tiles

Two tilesets are specified in `global/`, all built with tippecanoe from the parquet products above, so there is no second simplified copy of any geometry.

<mark>**Only `streams.pmtiles` is currently built.** `tile_regions.sh` is commented out of `pipeline.sh`. Decide whether it returns or is dropped from v3; the description below is the design.</mark>

| Tileset           | Zooms | Layers    | Notes                                                                                                                                                                                                                                                                                              |
|-------------------|-------|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `streams.pmtiles` | 0-11  | `streams` | Tiled per region then joined. Carries every attribute except `musk_k`, `musk_x`, `velocity_factor`, and `USContArea`. Reaches appear by Strahler order: 7+ at every zoom, 6+ from z5, 4+ from z7, 2+ from z9; order 1 reaches are not tiled at any zoom; the densest tiles drop features as needed |
| `regions.pmtiles` | 0-12  | `regions` | The region outlines with `TDXHydroRegion`. **Not currently built**                                                                                                                                                                                                                                 |

Every catchment band is tiled twice: `catchments` holds the polygons for fills and hit testing, and `catchment_lines` holds their boundaries for strokes. Clipping a polygon to a tile has to close the
ring along the tile edge, so stroking the polygon layer draws the tile grid across the map; clipping a line does not. **Style fills from `catchments` and strokes from `catchment_lines`, never stroke
the polygon layer.** Each band's geometry is cut at a quarter pixel of the band's finest zoom, rounded to a power of two (32 m for the z10 leaf, 64 m at z9, and so on), and tippecanoe generalizes
again per zoom on top of that.

### Routing Configurations

The files the router needs to route a region, short name `routing-configs`, are published under `routing/`, partitioned by the same `region=<id>` as the hydrography and kept apart from it.
They are the exact files used to generate official routing outputs. They are derived from the published hydrography after it is built, by the model scripts (`v3-model-scripts`,
`retrospective/1_prepare_hydrography.py`), which only read it, so a new forcing grid changes `routing/` and never `hydrography/`. The one exception is the Muskingum `musk_k` and `musk_x`, which
the hydrography pipeline computes so that they are attributes of the GIS files. Every other routing configuration is made here.

Per region products, in `routing/region=<id>/`:

| File                               | Format  | Description                                                                                    |
|------------------------------------|---------|------------------------------------------------------------------------------------------------|
| `routing.parquet`                  | Parquet | Muskingum routing parameters for river-route: `river_id`, `next_river_id`, `k`, `x`            |
| `gridweights_<forcing>_<id>.nc`    | NetCDF  | Runoff grid cell to catchment intersection weights for ERA5, IFS, GLDAS, etc., the standard    |
| `gridweights_ERA5_<id>.parquet`    | Parquet | An extra copy of the ERA5 weights for jsrr, the browser router. It does not replace the netCDF |

Every file lists the region's rivers in `riverIndex` order, which is topological. The router pairs weights with rivers by position, not by id: lateral inflow column i is routed into river i, so
a region's files must agree with each other and with the hydrography's row order.

`routing.parquet` is an exact duplicate of the metadata and streams tables' `riverId`, `nextRiverId`, `musk_k` and `musk_x`, renamed to the columns river-route reads, one row per reach in a
single row group. Nothing in it is recomputed. `gridweights_<forcing>_<id>.nc` is one row per runoff-cell-to-catchment intersection — `river_id`, `x_index`, `y_index`, `x`, `y`, `area_sqm`,
`proportion` — stamped with the grid file and the catchments it was cut from. It is named for the forcing grid, so each grid needs its own.

`gridweights_ERA5_<id>.parquet` holds the netCDF's rows without `x` and `y`, which jsrr does not read: `river_id`, `x_index` and `y_index` as int32, `area_sqm` and `proportion` as float32,
snappy compressed, without dictionary encoding, in one row group. That is the layout jsrr was measured to read fastest, 1.14 MB in 12 ms for region 7020014250's 95,030 weights. jsrr refuses a
region unless every river in `routing.parquet` has a weight and every weight names one of its rivers.

### Flood Forecast Products

The daily 15-day forecast is published under `forecasts15/`, partitioned `year=YYYY/month=MM/day=DD/`. Each day's partition holds

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

1. `Q` (riverId, member, time): 3 hourly discharge for each of the 50+1 IFS forecast members
2. `Qpercentiles` (riverId, percentiles, time): 3 hourly discharge ensemble at deciles 0 (min), 10, 20 ... 50 (median) ... 90, 100 (max)
3. `Qmean` (riverId, time): 3 hourly discharge ensemble mean

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
number=1/to/50/by/1,
param=205.128,
step=0/1/2/3/4/5/6/7/8/9/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27/28/29/30/31/32/33/34/35/36/37/38/39/40/41/42/43/44/45/46/47/48/49/50/51/52/53/54/55/56/57/58/59/60/61/62/63/64/65/66/67/68/69/70/71/72/73/74/75/76/77/78/79/80/81/82/83/84/85/86/87/88/89/90/93/96/99/102/105/108/111/114/117/120/123/126/129/132/135/138/141/144/150/156/162/168/174/180/186/192/198/204/210/216/222/228/234/240/246/252/258/264/270/276/282/288/294/300/306/312/318/324/330/336/342/348/354/360,
stream=enfo,
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

Routing warm states are not stored under `retrospective/`. They are written per computational unit with the forecast that consumes them, at
`forecasts15/year=YYYY/month=MM/day=DD/next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet`.

<mark>**This partition names a unit that no longer exists.** The hydrography computational group was removed, so a warm state is no longer partitioned by anything the hydrography publishes. Decide
what the router writes instead — `region=<id>` to match the hydrography partition, or a unit of its own choosing. The same question applies to `fim.geo.parquet` "for all groups" and to the working
layout under [Project Organization](#project-organization).</mark>

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

### Summary Table

Note: All times are given in UTC.

| Product Type                | Category      | Format            | Update Frequency         | Updates Available | Size                |
|:----------------------------|:--------------|:------------------|:-------------------------|:------------------|:--------------------|
| Stream tiles (`global/`)    | Model Sources | PMTiles           | None                     | N/A               | ~3.2 GB             |
| Region tiles (`global/`)    | Model Sources | PMTiles           | None                     | N/A               | not currently built |
| Global metadata (`global/`) | Model Sources | Parquet + Zarr v3 | None                     | N/A               | ~215 MB + ~70 MB    |
| Hydrography (by region)     | Model Sources | GeoParquet        | None                     | N/A               | ~15 GB all regions  |
| Modification records        | Model Sources | JSON              | None                     | N/A               | ~165 MB             |
| Routing Configs (by region) | Model Sources | Parquet + NetCDF  | None                     | N/A               | ~365 MB             |
| Forecast 3-hourly Discharge | Forecasts     | Zarr v3           | Daily @ 00:00            | 6am-12pm          | 150 GB              |
| Esri Animation Tables       | Forecasts     | CSV               | Daily @ 00:00            | 6am-12pm          | 120 x 120 MB        |
| Map Stylesets               | Forecasts     | bin + JSON        | Daily @ 00:00            | 6am-12pm          |                     |
| Alerts                      | Forecasts     | CSV               | Daily @ 00:00            | 6am-12pm          | 500 MB              |
| Warm States                 | Forecasts     | Parquet           | Daily @ 00:00            | 6am-12pm          |                     |
| Hourly Discharge            | Retrospective | Zarr v3           | Daily @ 00:00            | by 1am same day   | 10 TB               |
| Daily Discharge             | Retrospective | Zarr v3           | Daily @ 00:00            | by 1am same day   | 500 GB              |
| Monthly Average Discharge   | Retrospective | Zarr v3           | Monthly on 5th at 00:00  | by 1am same day   | ~20 GB              |
| Yearly Average Discharge    | Retrospective | Zarr v3           | Yearly on Jan 5 at 00:00 | by 1am same day   | ~2 GB               |
| Annual Maximums Discharge   | Retrospective | Zarr v3           | Yearly on Jan 5 at 00:00 | by 1am same day   | ~1 GB               |
| Return Periods              | Retrospective | Zarr v3           | None                     | N/A               |                     |
| Flow Duration Curves        | Retrospective | Zarr v3           | None                     | N/A               |                     |
| Forecast Flood Extents      | Flood Maps    | GeoParquet        | Daily @ 00:00            | 6am-12pm          | <5 GB               |
| Flood Map Tiles (ARC)       | Flood Maps    | GeoTIFF           | None                     | N/A               | <10 GB              |
| FLDPLN Libraries            | Flood Maps    | Zarr v3           | None                     | N/A               |                     |

## Zarr structuring
Every store is **Zarr v3**. The river axis is named `riverId`, discharge is named `Q`, and **`riverId` is the first dimension of every array** — `(riverId, time)`, not the `(time, riverId)` v2 used.
Chunking and sharding are fixed for every store, see [Chunking and sharding](#chunking-and-sharding); the schematics below repeat what that section sets.

### Datatypes

Only discharge valued arrays are bitrounded.

| Array                                                   | Dtype     | Bitrounded | Notes                                    |
|---------------------------------------------------------|-----------|------------|------------------------------------------|
| `riverId`                                               | `int32`   | no         | ids exceed int16, well inside int32      |
| `time`                                                  | `int32`   | no         | integer hours, see below                 |
| `member`                                                | `int32`   | no         | forecasts only, 1..51                    |
| `lead_time`                                             | `int32`   | no         | forecasts only, `units "hours"`          |
| `Q`, `Q_timesteps`                                      | `float32` | **yes**    | m3 s-1, 13 keepbits                      |
| `daily`, `hourly`                                       | `float32` | **yes**    | maximums.zarr annual maxima              |
| `{gumbel,logpearson3,lognormal,weibull}_{hourly,daily}` | `float32` | **yes**    | return-periods.zarr, 8 arrays            |
| `max_simulated_hourly`, `max_simulated_daily`           | `float32` | **yes**    | return-periods.zarr                      |
| `Qpercentiles`, `Qmean`                                 | `float32` | **yes**    | forecasts, reduced across `member`       |
| `percentiles`                                           | `int32`   | no         | forecasts only, deciles 0..100 by 10     |
| `recurrence_interval`                                   | `float32` | no         | **float**, not int, the axis carries 1.5 |
| `annual_exceedance_probability`                         | `float32` | no         | derived `1 / recurrence_interval`        |
| `p_exceed`                                              | `int32`   | no         | fdc.zarr, percent 0..100                 |
| `hourly_annual`, `daily_annual`                         | `float32` | **yes**    | fdc.zarr flow-duration curves            |

### Coordinate variables

- `riverId`
    - int32
    - The order of the ids is the same in every Zarr. To get this list, read the coordinate array off any store or refer to the hydrography datasets.
    - **The axis is in topological order, not ascending id order.** It is the hydrography row order described in [Row order, `riverIndex`, and
      `upstreamCount`](#row-order-riverindex-and-upstreamcount): level 2 regions ascending, watersheds along a Hilbert curve within a region, reaches in depth first post-order within a watershed. A
      river's *position* on the axis is therefore meaningful, every region, every terminal watershed and every upstream subset is one contiguous slice, and an id cannot be located by binary search on
      the array as published. That position is the `riverIndex`, and it is the join key shared by the vector tiles, the map style tables, the hydrography parquet, and every Zarr.
- `time`
    - int32, counted in hours, including on the daily, monthly and yearly axes, which are still expressed in hours rather than in their own step unit.
    - Left aligned time windows. That is, the corresponding value applies from the stated time step, t, until the start of the next time step, t+1.
    - Uses a unit string of style "\<interval\> since \<reference time\>", e.g. `hours since 1940-01-01T00:00:00+00:00`, with `calendar: proleptic_gregorian`.
    - **The reference time is per store, not shared.** Read `units` off the array and parse it, do not assume a common epoch. The string always carries an explicit UTC offset, which a reader must
      honor rather than letting a local time default apply.
    - `time` is written as an integer type, never as `datetime64[ns]`. Writing the latter re-encodes the integers into the datetime container, lands `units` in the attributes, and blocks any later
      rewrite of the store.
- `member`
    - int32. 1..51 for the 15-day forecast (50 perturbed + 1 control).
- `lead_time`
    - int32, units `hours`. A coordinate **on the time dimension, not a dimension of its own**. The time delta since initialization.
    - Left aligned on the same convention as `time`: **120 values**, running 0, 3, 6, ... 357, where the value 357 covers hour 357 through hour 360. The 15-day horizon is 120 intervals, not 121
      instants.
- `recurrence_interval`
    - Are exactly the float values: [1.5, 2, 5, 10, 25, 50, 100].
- `annual_exceedance_probability`
    - float32 on the `recurrence_interval` axis, `1 / recurrence_interval`.
- `p_exceed`
    - int32, exceedance probability in percent, 0..100. 101 values, so Q95 is row 95.
- `percentiles`
    - Are exactly the integer values: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100].

### Data variables

- Discharge
    - **Q** in every store. In the forecasts it carries a `member` dimension, there is no separate variable name for ensemble discharge.
    - **Q_timesteps** in the monthly and yearly stores: the same values as `Q`, rechunked for all-rivers-few-timesteps access (map styling) where `Q` is chunked for one-river-whole-series access
      (plots). One store serves both patterns.
    - Q values are always left aligned on their time intervals per the definition of the time coordinate variable. That is, Q is the average over the ***following*** 1 hour, 3 hours, 24 hours, 1
      calendar month, or 1 calendar year.
    - Units: cubic meters per second (m3/s), as `m3 s-1`.
    - Carries `aggregation_method` ("mean", or "max" on the maxima series) and `keepbits`.
    - Carries `long_name` "Discharge at catchment outlet", `standard_name` "discharge" and `units` "m3 s-1". Every discharge array in every store carries all three, including `Q_timesteps` and the
      maxima series, so a store is self describing without the reader consulting this document.
- Ensemble summaries, forecasts only, reduced across `member` at each time step
    - **Qpercentiles** (riverId, percentiles, time): the ensemble distribution on the `percentiles` coordinate.
    - **Qmean** (riverId, time): the ensemble mean. Distinct from `Qpercentiles` at 50, which is the median. The two diverge whenever the ensemble is skewed, which is most of the time on a flood
      peak.
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
- Bitround discharge to **13 keepbits**, round-to-nearest-even on the low mantissa bits. This bounds relative error at `2^-14` ≈ 6.1e-05.
- **13 is the only keepbits in the system.** The router rounds the discharge it writes to it, and every store derived from that discharge is rounded to the same grid again, which leaves a value that
  is already on it untouched. That is what makes a peak read out of `maximums.zarr` the same float as the peak in `hourly.zarr`, a daily mean reproducible by resampling the published hourly store,
  and re-rounding on a daily append a no-op. A second keepbits anywhere would break all three.
- Write `keepbits` into each discharge array's attributes, and reapply the same value when appending. Bitrounding is idempotent only if keepbits is unchanged.
- Bitround by rounding the values before they are handed to Zarr, **not** with a `numcodecs.bitround` filter in the array's codec chain. A filter puts a codec name in the metadata that a browser
  client cannot decode, for the same reason only blosc is permitted below.
- Compress every array, discharge and coordinate alike, with `blosc(cname="zstd", clevel=5, shuffle="shuffle")`.
- **Blosc is the only permitted compressor.** Browser clients ship blosc alone in their WASM codec builds, so an array written with any other codec fails to decode there.
- Do not add a separate shuffle codec entry. Blosc applies shuffle inside its own frame.
- Leave `typesize` unset so each array derives it from its own itemsize.
- **Consolidate every store's metadata.** A client that had to list the store and fetch a `zarr.json` per array would pay a round trip for each one before reading a value; consolidated metadata makes
  opening a store a single request. The one exception is `hydrography/global/metadata.zarr`, which is not consolidated because a client pulls single arrays out of it by name.

### Chunking and sharding

Chunking is fixed, not a per store choice, and every discharge array is **sharded** — new in v3. A chunk is the unit a reader decompresses; a shard is the file it lives in, `sharding_indexed` with
`blosc` inside, and a reader that wants one chunk range-GETs it out of the shard using the shard index. Browser clients need the sharding codec alongside blosc.

- **`riverId` is the first dimension of every array** — `(riverId, time)`, `(riverId, member, time)`, `(riverId, recurrence_interval)`. This is new in v3: v2 put time first. It is the layout the
  router produces, so nothing between the router and a published store transposes anything, and it is the only change here a v2 reader has to make. It does not change a single byte on disk: with one
  river per chunk a `(1, n_time)` chunk and an `(n_time, 1)` chunk both hold that river's whole series contiguously, so the two orders shard byte for byte identically — same compression, same file
  count, same cost to read one river. What it buys is on the write side, where a time-first store made every writer transpose a buffer to fill it.
- **One river per chunk.** The read that matters is one river's whole series, and a chunk holding several rivers makes that read decompress its neighbors. Chunks are stored C-order, so a wider chunk
  also interleaves rivers in the byte stream and breaks up the temporal autocorrelation zstd feeds on: measured on a real group, 2.85x compression at one river per chunk against 2.16x at ten, and
  1.5 ms against 2.8 ms to read one river.
- **250 rivers per shard.** One file on disk, one object in S3, and one whole cacheable GET answers a 250 river bulk read. Unsharded, one river per chunk would put millions of objects in the bucket.
- **The time axis is cut at `2025-01-01`.** The first chunk holds the 85 years the record was built from, 1940-01-01 through 2024-12-31, and everything from 2025 on falls in the next chunk. The
  historical chunk never changes again, so the daily append rewrites only the recent chunk, and only the shards the new steps land in. A store whose record ends before the split is a single chunk.
  Shards are one chunk wide on the time axis for the same reason: a shard spanning both chunks would drag the historical one into every append.
- **`Q_timesteps` is chunked the other way and is not sharded:** `(250000, 1)`, one timestep across 250,000 rivers. It serves the opposite read, every river at one step for map styling, and one chunk
  already answers it.
- **No axis other than `time` and `riverId` is ever split.** `member`, `lead_time`, `percentiles`, `recurrence_interval` and `p_exceed` each sit whole in one chunk, so a forecast chunk is one river's
  entire ensemble over the entire horizon, and a return period chunk is one river's entire curve.

| Array                                                               | chunks                     | shards                         |
|---------------------------------------------------------------------|----------------------------|--------------------------------|
| `Q` on hourly, daily, monthly, yearly; `hourly`/`daily` on maximums | `(1, 85y)`                 | `(250, 85y)`                   |
| `Q_timesteps` on monthly, yearly                                    | `(250000, 1)`              | none                           |
| `Q` on the forecasts                                                | `(1, 51, 120)`             | `(250, 51, 120)`               |
| `Qpercentiles`, `Qmean` on the forecasts                            | `(1, 11, 120)`, `(1, 120)` | `(250, 11, 120)`, `(250, 120)` |
| `gumbel_*`, `logpearson3_*`, `lognormal_*`, `weibull_*`, `*_annual` | `(1, all)`                 | `(250, all)`                   |
| `max_simulated_hourly`, `max_simulated_daily`                       | `(250000,)`                | none                           |
| `riverId`, `time`, and every other coordinate array                 | `(all,)`                   | none                           |

`85y` is the number of steps 85 years spans at that store's own step: 745,128 hourly, 31,047 daily, 1,020 monthly, 85 yearly.

### retrospective/hourly.zarr, daily.zarr

```text
daily.zarr/
├── zarr.json                 title, license, metadata consolidated
├── riverId/       int32   (riverId,)          chunks (all,)
├── time/          int32   (time,)             chunks (all,)          units "hours since 1940-01-01T...+00:00", proleptic_gregorian
└── Q/             float32 (riverId, time)     chunks (1, 85y)  shards (250, 85y)
                                               units "m3 s-1", long_name, standard_name, aggregation_method "mean", keepbits 13
                                               ^ one river a chunk: read pattern is one river, all time. 85y = 1940..2024,
                                                 2025-present is the second chunk, so an append leaves the first alone
dims: riverId=N · time=31602 (1940-01-01..now, daily)

hourly.zarr/                  identical shape/attrs, time on hourly axis — largest store
                              note the daily axis is still counted in hours, not days
```

### retrospective/monthly.zarr, yearly.zarr

```text
monthly.zarr/                 two chunkings of the same values, one store serves both access patterns
├── zarr.json
├── riverId/       int32   (riverId,)
├── time/          int32   (time,)             units "hours since 1940-01-01T00:00:00+00:00"
├── Q/             float32 (riverId, time)     chunks (1, 85y)      shards (250, 85y)   one river, whole series -> plots
└── Q_timesteps/   float32 (riverId, time)     chunks (250000, 1)   unsharded           all rivers, one timestep -> map styling
                                               ^ same values as Q, rechunked; not re-bitrounded
dims: riverId=N · time=1038 months

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
│                                        every array below: chunks (1, all), shards (250, all) -- one river's whole curve a chunk
├── gumbel_hourly/               float32 (riverId, recurrence_interval)      each of the four distributions is fit
├── gumbel_daily/                float32 (riverId, recurrence_interval)      against BOTH maximum series, so the
├── logpearson3_hourly/          float32 (riverId, recurrence_interval)      array name always carries the series
├── logpearson3_daily/           float32 (riverId, recurrence_interval)      it was fit to
├── lognormal_hourly/            float32 (riverId, recurrence_interval)
├── lognormal_daily/             float32 (riverId, recurrence_interval)
├── weibull_hourly/              float32 (riverId, recurrence_interval)
├── weibull_daily/               float32 (riverId, recurrence_interval)
├── max_simulated_hourly/        float32 (riverId,)                 largest value in the record, hourly series
└── max_simulated_daily/         float32 (riverId,)                 ...and daily — the hourly max is always >= it
                              there is no series-agnostic "gumbel" array — pick a series explicitly

maximums.zarr/                annual maxima the fit above is derived from, kept so it stays reproducible
├── zarr.json
├── riverId/         int32   (riverId,)
├── time/            int32   (time,)                      one value per year, still counted in hours
├── daily/           float32 (riverId, time)              aggregation_method "max", keepbits 13
└── hourly/          float32 (riverId, time)              aggregation_method "max", keepbits 13
                              both chunked (1, 85y) in shards of (250, 85y) like every other timeseries array.
                              Neither cascades: the annual maximum of the hourly series cannot be recovered from
                              daily means, so both are reduced where the hourly values are, in one pass.
```

### retrospective/fdc.zarr

```text
fdc.zarr/                     flow duration curves. no time dim — exceedance probability is the axis.
├── zarr.json                 title, description (percentile of the exceedance distribution), license
├── riverId/         int32   (riverId,)
├── p_exceed/        int32   (p_exceed,)              0..100 percent, 101 values -> Q95 is row 95
├── hourly_annual/   float32 (riverId, p_exceed)      chunks (1, all), shards (250, all)   units "m3 s-1", keepbits 13
└── daily_annual/    float32 (riverId, p_exceed)      chunks (1, all), shards (250, all)   units "m3 s-1", keepbits 13
                              ^ whole curve for one river in one chunk: the read pattern is one river,
                                every exceedance level — same shape of access as return-periods.zarr
```

`p_exceed` is *exceedance*, not a plotting position. `hourly_annual[95]` is the flow the reach exceeds 95% of the time, that is, a low flow threshold. The `below-q95` map styleset is built from
exactly that row.

### forecasts15/year=YYYY/month=MM/day=DD/discharge.zarr

```text
discharge.zarr/
├── zarr.json       title, initialization_time "2026-07-10T00:00:00Z", metadata consolidated
├── riverId/        int32   (riverId,)                    chunks (all,)
├── member/         int32   (member,)                     1..51 — 50 perturbed + 1 control
├── time/           int32   (time,)                       chunks (all,)   units "hours since 2026-07-10T00:00:00+00:00"
├── lead_time/      int32   (time,)                       coordinate ON the time dim, NOT its own dim
│                                                         timedelta since init, units "hours" -> 0, 3, 6, ... 357
│                                                         left aligned like time: 357 covers hour 357..360, so the
│                                                         15 d horizon is 120 intervals, not 121 instants
├── percentiles/    int32   (percentiles,)                [0, 10, ... 100] deciles: 0 = min, 50 = median, 100 = max
├── Q/              float32 (riverId, member, time)       chunks (1, 51, 120)   shards (250, 51, 120)
│                                                         units "m3 s-1", keepbits 13
│                                                         ^ whole ensemble x whole horizon for one river = 1 chunk.
│                                                           member and time are never split, the horizon is 120 steps
├── Qpercentiles/   float32 (riverId, percentiles, time)  chunks (1, 11, 120)   shards (250, 11, 120)   reduction of Q
└── Qmean/          float32 (riverId, time)               chunks (1, 120)       shards (250, 120)
                                                          NOT Qpercentiles[50] — mean != median
dims: riverId=N · member=51 · time=120 (15 d @ 3 h) · percentiles=11
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

The runoff forcings are kept at the top level of the bucket under `forcings/`, one tree per source, see [Organization on S3](#organization-on-s3).

### ECMWF IFS

See the MARS requests in [Flood Forecast Products](#flood-forecast-products).

IFS runoff is stored as GRIB at `forcings/ifs/YYYYMMDDHH/<filename>.grib`, one directory per forecast initialization, named for the initialization date and hour.

### ERA5 and ERA6

ERA5 runoff is stored as one Zarr v3 store per year at `forcings/era5/YYYY.zarr`. Each store holds a single data variable, `ro`, the ERA5 runoff.

| Axis      | Chunk        |
|-----------|--------------|
| latitude  | 16           |
| longitude | 16           |
| time      | -1, the year |

The time axis is never split, so one chunk is a 16 x 16 block of cells over the store's entire year, and a region's whole runoff series is read in one pass over the chunks its cells fall in. The
chunking is set by that read pattern, not by [Chunking and sharding](#chunking-and-sharding), which governs the discharge stores.

<mark>ERA6 TBD</mark>

## Implementation Details

### Project Organization

This suite expects a machine with a certain directory structure: a home directory with subdirectories for IFS, ERA5, forecasts which has subdirectories for each YMD, and retrospective. This is the
working layout on the compute machine. The final products it produces are uploaded to the layout described in [Organization on S3](#organization-on-s3). The scripts and intermediate files below still
use the older VPU naming.

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
$RFS_DATA_ROOT/
    hydrography/region=<id>/            the published dataset, uploaded as is to s3://river-forecast-system-v3/hydrography/
    hydrography/global/                 the products that span every region
    hydrography-scratchfiles/           regions/, pmtiles/, logs/ — per region intermediates no consumer needs
```

Version controlled inputs live in the repository's `network_data/`: `lake_table.csv` (inlet, outlet, lake id, endorheic flag, trace flag), `dropped_watersheds/*.csv` (outlets to remove),
and `tdxhydro_splits/` (the per region id offsets and the duplicated watersheds to drop where regions overlap).

The published tree holds one build input: `region=<id>/mods/`, which step 3 writes and step 4 reads back. That is the price of the edit records having exactly one copy rather than a scratch original
and a published duplicate, and it is why step 4 refuses to run when they are absent rather than treating a missing file as "nothing was edited".

The steps, in order. Steps 3 and 4 run per region in parallel; everything else is global.

| # | Script                    | Scope                 | Produces                                                                                                                                                                                                                                                                                                                                    |
|---|---------------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | `1_translate_tdxhydro.py` | per region, once      | Raw geopackages to geoparquet. Offsets `LINKNO` by a per region constant so ids are globally unique, adds the geodesic length, region id, and outlet lon/lat, drops watersheds duplicated between overlapping regions, and nodes the source basin coverage so later dissolves are exact                                                     |
| 3 | `3_simplify_streams.py`   | per region            | `streams_*`, `metadata_*`, `confluences_*` in the region's scratch directory. Applies the [network simplification](#network-simplification) and [lake edits](#lakes-and-reservoirs), computes the Muskingum parameters, sets the nested-set row order, and writes every edit to the region's **published** `mods/*.json`                    |
| 4 | `4_create_catchments.py`  | per region            | `catchments_*`: the source basins redirected along step 3's edits and dissolved into one polygon per surviving reach, projected, coverage simplified at 20 m, snapped to 1 m, in the published row order                                                                                                                                    |
| 5 | `5_concatenate_global.py` | all regions           | Concatenates the regions in ascending region number, stamps the global `riverIndex` by offset arithmetic, checks that no reach drains across a region, that `riverId` is globally unique, and that the nested-set property holds; writes `global/metadata.parquet` and `metadata.zarr`; publishes every region's tables into `region=<id>/` |
| 6 | `6_publish_regions.py`    | all regions           | Dissolves each region's published catchments into `boundary_<id>.geo.parquet`                                                                                                                                                                                                                                                               |
| 7 | `tile_streams.sh`         | per region, then join | `streams.pmtiles`: one tippecanoe run per region, largest first, joined with `tile-join`                                                                                                                                                                                                                                                    |
| 9 | `tile_regions.sh`         | global                | `regions.pmtiles`. **Not currently run**                                                                                                                                                                                                                                                                                                    |

Every step is idempotent at the granularity of its output files: a step whose outputs exist exits successfully without rewriting them, so a rebuild after a change touches only what depends on it.
Reordering the rows (step 5) never requires rerunning the simplification (step 3) or the catchments (step 4); it permutes rows and derives integers.

Then, outside the numbered steps: optionally, `extras_identify_id_map.py` writes `global/tdxhydro_to_v3_id_map.parquet`, a two column table mapping every original TDX-Hydro
reach in every region, about 16 million, to the v3 `riverId` that now represents it, or null when it was dropped or is in a region v3 does not cover.

The pipeline writes no routing configuration except the Muskingum `musk_k` and `musk_x` step 3 computes, which are kept in the hydrography so that they are attributes of the GIS files.
`routing.parquet` and the grid weights are derived from its published output afterwards, by the model scripts, into `routing/region=<id>/` beside `hydrography/`, see
[Routing Configurations](#routing-configurations).

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

All exports land under `s3://river-forecast-system-v3/`, see [Organization on S3](#organization-on-s3).

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

### 2026-09-27 (forcings)

- **Added `forcings/` at the top level of the bucket**, the runoff the router reads. See [Input Datasets](#input-datasets).
    - `forcings/era5/YYYY.zarr`: one Zarr v3 store per year holding only `ro`, chunked 16 x 16 on latitude and longitude with the time axis unsplit, so a cell's whole series is one chunk read.
    - `forcings/ifs/YYYYMMDDHH/<filename>.grib`: the IFS GRIB files, one directory per forecast initialization.

### 2026-09-26 (routing)

- **Routing configurations moved out of the hydrography into `routing/region=<id>/`.** `routing.parquet` and `gridweights_ERA5_<id>.nc` were published in `hydrography/region=<id>/`; they now
  live in a tree of their own beside it, partitioned by the same regions. See [Routing Configurations](#routing-configurations).
- **The hydrography pipeline holds no routing configuration** except `musk_k` and `musk_x`, which it still computes so that they are attributes of the GIS files. The routing files are derived
  from the published hydrography by the model scripts, which only read it, so the release of one no longer waits on or rewrites the other.
- **Added `gridweights_ERA5_<id>.parquet`**, an extra copy of the ERA5 weights in the layout jsrr, the browser router, reads fastest. The netCDF stays the weights' standard format.
- Readers of `routing.parquet` or the grid weights must read them from `routing/region=<id>/`. Routing configs are ~365 MB, the parquet weights included.

### 2026-09-20

- **`riverId` is now the first dimension of every array**, where v2 and every earlier draft of this spec put `time` first. `Q` is `(riverId, time)`, forecast `Q` is `(riverId, member, time)`,
  `Qpercentiles` is `(riverId, percentiles, time)`, the return period and flow duration arrays are `(riverId, recurrence_interval)` and `(riverId, p_exceed)`. Chunk and shard shapes flip with them:
  `(1, 85y)` in shards of `(250, 85y)`, `Q_timesteps` `(250000, 1)`.
- **No bytes change.** One river per chunk means a `(1, n_time)` chunk and an `(n_time, 1)` chunk hold the same river's series contiguously, so the shards are byte for byte identical: the
  compression, the object count and the cost of reading one river are all exactly what they were. This is a metadata change for readers and nothing else.
- **Why:** the router writes `(riverId, time)`, so a time-first store made every step of the pipeline transpose a buffer to fill it — once on the way out of the router, again on the way into the
  concatenated store. Nothing between the router and a published store transposes anything now.
- **Readers must be updated.** A client that indexes `Q[t, r]` now wants `Q[r, t]`. Anything reading a v3 store through xarray by dimension name is unaffected.

### 2026-09-17 (hydrography)

- **The computational group partition is gone.** Hydrography is partitioned by HydroBASINS level 2 region, `region=<id>`, and the global products moved from `group=0/` to `global/`. 127 groups
  became 47 regions; `groupId`, `groupIds_table.csv` and `groups.geo.parquet` no longer exist. The row order lost its outermost key and is now two levels deep inside a
  region, with the regions concatenated in ascending region number.
- **Added [Modification records](#modification-records).** The edits the pipeline makes to TDX-Hydro are published as `region=<id>/mods/`, as the provenance of a network that is a modification of a
  previous dataset. They are the only copy, so the published tree holds one build input.
- **One of the two tilesets is not currently produced.** The region tileset is not built. It is marked rather than deleted, pending a decision.
- Reach count 5,452,029 to 4,917,183; source regions 46 to 47.
- `routing.parquet` and `gridweights_ERA5_<id>.nc`, written by river-route, are documented in the region partition where they now live.
- Row group sizes corrected: 500 for streams and catchments, 2,000 for metadata and confluences.

### 2026-09-17

- **Bitrounding is 13 keepbits everywhere**, was 15. One keepbits across the router and every derived store is what makes a value found in one store the same float in every other, makes a daily mean
  reproducible by resampling the published hourly store, and makes re-rounding on an append a no-op. Bitrounding is applied to the values before they are written, never as a `numcodecs.bitround`
  filter in the codec chain, which a browser client cannot decode.
- **Added [Chunking and sharding](#chunking-and-sharding).** Sharding is new in v3 and was previously undocumented, and the chunk shapes in the schematics were marked as placeholders. Both are now
  fixed: one river per chunk, 250 rivers per shard, the time axis cut at 2025-01-01 so the 1940-2024 record is an immutable chunk, `Q_timesteps` chunked `(250000, 1)` and unsharded, and no other axis
  ever split.
- **Every store's metadata is consolidated**, except `hydrography/global/metadata.zarr`.
- Every discharge array carries `long_name` and `standard_name` alongside `units`, `aggregation_method` and `keepbits`.
- The source of truth for encodings is `rfs_spec.py` in the `rfs-v3-scripts` repository, not `generate_v3_examples_data.py`. The retrospective products are built by the numbered scripts beside it.
