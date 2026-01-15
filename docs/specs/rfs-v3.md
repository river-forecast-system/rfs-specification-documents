## RFS V3

This is a draft of the RFS v3 specifications. It is not complete and is subject to change.

Documentation for the River Forecast System version 3, RFS v3.

- Model Version: 3.0
- ==Launch Date: TBD==
- ==End Date: TBD==

| Property                       | Value                               |
|--------------------------------|-------------------------------------|
| Reference IFS Version          | 50r1                                |
| IFS Resolution                 | O1280 grid, 0.1 degree, 9km equator |
| Forecast Length                | 15 days                             |
| Forecast Time Step             | 3 hours                             |
| Retrospective Start            | 1940-01-01                          |
| Retrospective Update           | daily at 00:00 UTC                  |
| Retrospective Native Time Step | 1 hourly average                    |

### Summary of Changes

The main changes in v3 compared to v2 are:

- Anticipated use of HydroSHEDS v2 Hydrography
- Forecasts and retrospective simulations use a shared initializations
- Upgrading to use the newest IFS version, 50R1, including use of the native octahedral, or reduced gaussian grid, native to the IFS version.
- Discontinue the "52nd ensemble member" IFS HRES. HRES was discontinued after becoming identical to the control forecast.
    - [https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens](https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens)
- Upgrade computations to use river-route (Python) for matrix muskingum routing
- Begin producing global reference as well as daily forecasted flood maps
- Volumes are computed using the original reduced gaussian grid/mesh rather than resampled to a uniform lat/lon grid

### Possible changes

- Implement non-linear routing
- 1 hourly average forecast for flash flood warnings
- Discontinue the forecast records product?????

## List of Products

### Model Sources

1. Model Sources and Configuration Files by computational unit
    1. hydrography (divided into computational units)
        1. stream center lines (geoparquet)
        2. catchment boundaries (geoparquet)
        3. confluence points (geoparquet)
        4. lake polygons (geoparquet)
        5. simplified streams for mapping (geoparquet)
    2. Routing configs for river-route
        1. muskingum parameters (parquet)
        2. connectivity (parquet)

### Forecasted Discharge

1. Daily forecasts, 15-Day by 3-hourly average, and derivatives
    1. Discharge zarr (YYYYMMDDHH.zarr) with the variables
        1. Qens (member, time, river_id): 3 hourly discharge for each of the 50+1 IFS forecast members
        2. ==Qmean (time, river_id): 3 hourly discharge ensemble mean==
        3. ==Qmed (time, river_id): 3 hourly discharge ensemble median==
        4. ==Qmin (time, river_id): 3 hourly discharge ensemble minimum==
        5. ==Qmax (time, river_id): 3 hourly discharge ensemble maximum==
    2. Forecast Warnings and Summaries 
        1. Map styling tables used to produce animated maps
        2. ==Datetimes when rivers exceed warning thresholds (parquet)==
2. ==Monthly on 1st and 15th, 45-Day by 24-hourly average, and derivatives==
    1. ==Discharge zarr (YYYYMMDDHH.zarr) with the variables==
        1. ==Qens (member, time, river_id): daily average discharge for each of the 50+1 IFS forecast members==
        2. ==Qmean (time, river_id): daily average discharge ensemble mean==
        3. ==Qmed (time, river_id): daily average discharge ensemble median==
        4. ==Qmin (time, river_id): daily average discharge ensemble minimum==
        5. ==Qmax (time, river_id): daily average discharge ensemble maximum==

### Web Maps

1. Daily Forecasted Flood Maps
    1. [https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b](https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b)
2. ==Discharge max flood maps?==
3. ==Return period flood maps?==

### Retrospective Discharge

1. Retrospective discharge simulation and derivatives
    1. Hourly discharge (hourly.zarr) **<- native resolution**
    2. Daily discharge (daily.zarr)
    3. Monthly average discharge (monthly-timeseries.zarr)
    4. Yearly average discharge (yearly-timeseries.zarr)
    5. Annual maximums discharge (maximums.zarr)
    6. Return periods discharge (return-periods.zarr)
    7. Flow duration curves discharge (fdc.zarr)
2. Final states for each forecast initialization date (parquet)

### Flood Maps

1. ==Daily Forecasted Flood Maps==
    1. ==what is the structure and naming convention==
2. ==Return Period Indexed Flood Maps==
    1. ==what is the structure and naming convention==

## Data Access

### Summary

RFS data is available in the following ways:

1. [https://hydroviewer.geoglows.org](https://hydroviewer.geoglows.org)
2. ==~~https://geoglows.ecmwf.int~~==
3. AWS S3 buckets
4. geoglows Python package
5. ==riverforecastsystem JS==
6. ==Lambda functions deployable to user provided account at AWS, GCP, Netlify, Vercel, etc.==

| Product Type                | Category      | Format     | Update Frequency             | Updates Available   | Size      |
|-----------------------------|---------------|------------|------------------------------|---------------------|-----------|
| Hydrography (global)        | Model Sources | GeoParquet | None                         | N/A                 |           |
| Hydrography (by VPU)        | Model Sources | GeoParquet | None                         | N/A                 |           |
| Routing Configs (by VPU)    | Model Sources | Parquet    | None                         | N/A                 |           |
| Forecast 3-hourly Discharge | Forecasts     | Zarr v3    | Daily @ 00:00 UTC            | 6am-12pm UTC        | 150GB     |
| Map Summary Tables          | Forecasts     | CSV        | Daily @ 00:00 UTC            | 6am-12pm UTC        | 81x120 MB |
| Flood Warnings              | Forecasts     | Parquet    | Daily @ 00:00 UTC            | 6am-12pm UTC        | 500 MB    |
| Hourly Discharge            | Retrospective | Zarr v3    | Daily @ 00:00 UTC            | by 3am UTC same day | 10 TB     |
| Daily Discharge             | Retrospective | Zarr v3    | Daily @ 00:00 UTC            | by 3am UTC same day | 500 GB    |
| Monthly Average Discharge   | Retrospective | Zarr v3    | Monthly on 5th at 00:00 UTC  | by 3am UTC same day | ~20 GB    |
| Yearly Average Discharge    | Retrospective | Zarr v3    | Yearly on Jan 5 at 00:00 UTC | by 3am UTC same day | ~2 GB     |
| Annual Maximums Discharge   | Retrospective | Zarr v3    | Yearly on Jan 5 at 00:00 UTC | by 3am UTC same day | ~1 GB     |
| Return Periods              | Retrospective | Zarr v3    | None                         | N/A                 |           |
| Flow Duration Curves        | Retrospective | Zarr v3    | None                         | N/A                 |           |
| Flood Extent Maps           | Flood Maps    | COG        | Daily @ 00:00 UTC            | 6am-12pm UTC        | <5GB      |
| Return period flood maps    | Flood Maps    | COG        | None                         | N/A                 | <10GB     |

### Zarr Details

**Coordinate variables**

- **river_id**
    - Integers
    - The order of the river_id list is the same in all Zarrs. To get this list, check any zarr or refer to the hydrography datasets
    - Fill value: n/a
- **time**
    - Integer type
    - Left aligned time windows. That is, the corresponding value applies from this time step, t, to the next time step, t+1.
    - Uses a unit string of style "<interval> since <reference time>" e.g. hours since 1940-01-01 00:00:00
    - Fill value: n/a
- **returnperiod**
    - [1.5, 2, 5, 10, 25, 50, 75, 100]

**Discharge Variables**

- **Q**, **Qens**, **Qmean**, **Qmed**, **Qmin**, **Qmax**
    - Float numbers with 3 decimals stored as integers (multiplied by 1000)
    - Q values are "left-aligned" on their time intervals. That is, Q is the average over the **_following_** 1 hour, 3 hours, 24 hours, 1 calendar month, or 1 calendar year
    - Units: cubic meters per second (m3/s)
    - Fill value: missing values are not allowed.

**Return Periods**

- **max_simulated**
- **logpearson3**
- **lognormal**
- **gumbel**
- **weibull**

### Organization on S3

```markdown
s3://river-forecast-system/

- v3/
    - hydrography/
        - vpu
        - global
        - attribute-tables/
    - routing-configs/
    - map-tiles/
    - retrospective/
        - hourly.zarr
        - daily.zarr
        - monthly-timeseries.zarr
        - monthly-timesteps.zarr
        - yearly-timeseries.zarr
        - yearly-timesteps.zarr
        - maximums.zarr
        - return-periods.zarr
        - fdc.zarr
        - final-states/
            - vpu=*
                - YYYYMMDDHHMMSS.parquet
    - flood-maps/
        - 2years.tiff
    - forecasts/
        - forecast-records/
            - records_YYYY.zarr
        - daily-forecasts/
            - YYYYMMDD/
                - discharge.zarr
                - warnings.parquet
                - map_tables/
                    - YYYYMMDDHH.csv
                - flood-maps
                    - option 1: flood_extents.tiff
                    - option 2: directory called flood-extents/
                        - 1 tiff per map tile
```

## Input Datasets

### ECMWF IFS ENFO

Request parameters:

- Stream: enfo
- Types:
    - pf, perturbed forecast, 50 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid in the native resolution of reduced gaussian grid/mesh. No regridding or resampling.

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

### ECMWF IFS EEFO

Request parameters:

- Stream: eefo
- Types:
    - pf, perturbed forecast, 100 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid in the native resolution of reduced gaussian grid/mesh. No regridding or resampling.

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

### ERA5

[//]: # (todo)

### HydroSHEDS v2

[//]: # (todo)
