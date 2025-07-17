# RFS V3

This is a draft of the RFS v3 specifications. It is not complete and is subject to change.

Documentation for the River Forecast System version 3, RFS v3.

- Model Version: 3.0
- <mark>Launch Date: TBD</mark>
- <mark>End Date: TBD</mark>

| Property                       | Value                               |
|--------------------------------|-------------------------------------|
| Reference IFS Version          | 50r1                                |
| IFS Resolution                 | O1280 grid, 0.1 degree, 9km equator |
| Forecast Length                | 15 days                             |
| Forecast Time Step             | 3 hours                             |
| Retrospective Start            | 1940-01-01                          |
| Retrospective Update           | 1xWeekly, Sunday 00:00 UTC          |
| Retrospective Native Time Step | 1 hourly average                    |

## Journal Papers

- <mark>TBD</mark>

## Conceptual explanation of the model

RFS produces 6 main datasets:

1. Hydrography (static)
2. Routing configs (static)
3. Daily flood forecasts
4. Flash flood forecasts
5. Retrospective simulation
6. Flood maps

RFS data is available in the following ways:

1. https://hydroviewer.geoglows.org
2. https://geoglows.ecmwf.int
3. AWS S3 buckets
4. geoglows Python package
5. ArcGIS Living Atlas web map layer

The forecasts and retrospective simulations are now coupled together. Forecasts are an extension of the retrospective simulation. Each day the retrospective
simulation is updated, then 5 simulations are run for first 24 hours of runoff predictions in the days between the retrospective simulation's end and the
present. The initialization value of the forecast in each 5 day gap run is the ensemble average at the end of the 24 hours. The final initialization value of
the 5 days is used to kick off the daily 15-day forecast. The retrospective simulation covers 1940-01-01 to near real time with 5 days of lag. It was
initialized in 1940 from the average of all January 1 values in the period 1950-2019 or 70 full years.

**_Computations occur where???_**

## Model Version Improvements

The main changes in v3 compared to v2 are:

- Forecasts and retrospective simulations are tightly coupled using shared initializations
- Total streams about 6 Million. Modified the v2 streams to remove about 800k streams in the middle of deserts, within lakes, oceans, and other inappropriate areas.
- Upgrading to use the newest IFS version, 50R1, including use of the native octahedral, or reduced gaussian grid, native to the IFS version.
- Discontinue the forecast records product
- Discontinue the "52nd ensemble member" IFS HRES. HRES was discontinued after becoming identical to the control forecast.
    - [https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens](https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens)
- Begin producing global reference as well as daily forecasted flood maps
- <mark>Begin using river-route (python) instead of RAPID (user-compiled fortran) for matrix muskingum routing???</mark>
- <mark>Implement non-linear routing???</mark>
- <mark>1 hourly average forecast for flash flood warnings???</mark>
- <mark>WMO, Google, FFGS....???</mark>

## Available Products

<mark>Dataset repository locations TBD</mark>

### Hydrography

| Property            | Value       |
|---------------------|-------------|
| Approximate Streams | 6.0 Million |
| Reference Product   | TDX-Hydro   |

### Flood Forecast Products

| Product Type       | Time Step        | Format  | Frequency | Description                                                 |
|--------------------|------------------|---------|-----------|-------------------------------------------------------------|
| Forecast Discharge | 3 hourly average | Zarr    | Daily     | Daily average forecasted discharge for 15 days              |
| Map Summary Tables |                  | CSV     | Hourly    | Summary tables for web map styling, 1 per forecast timestep |
| Flood Warnings     |                  | Parquet | Daily     | Warnings issued based on forecasted discharge               |

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

### Retrospective Simulation Products

| Product Type                 | Time Step       | Format | Frequency | Description                                                                          |
|------------------------------|-----------------|--------|-----------|--------------------------------------------------------------------------------------|
| Hourly Discharge             | hourly average  | Zarr   | Daily     | Hourly average retrospective simulation                                              |
| Daily Discharge              | daily average   | Zarr   | Daily     | Daily average retrospective simulation                                               |
| Monthly Timeseries Discharge | monthly average | Zarr   | Monthly   | Monthly average retrospective simulation, timeseries chunks                          |
| Monthly Timesteps Discharge  | monthly average | Zarr   | Monthly   | Monthly average retrospective simulation, timestep chunks                            |
| Yearly Timeseries Discharge  | yearly average  | Zarr   | Yearly    | Yearly average retrospective simulation, timeseries chunks                           |
| Yearly Timesteps Discharge   | yearly average  | Zarr   | Yearly    | Yearly average retrospective simulation, timestep chunks                             |
| Maximums Discharge           | annual maximum  | Zarr   | Yearly    | Annual maximums of retrospective simulation from the hourly and daily average series |
| Return Periods               |                 | Zarr   | Once      | Return periods of retrospective simulation                                           |
| Flow Duration Curves         |                 | Zarr   | Once      | Flow duration curves of retrospective simulation                                     |

### Flood Maps

| Product Type | Time Step | Format | Frequency               | Description                                                      |
|--------------|-----------|--------|-------------------------|------------------------------------------------------------------|
| Flood Maps   |           | COG    | Daily w/ Flood Forecast | Flood maps for the forecasted discharge, 1 per forecast timestep |

## Changelog

## Implementation Details

### Project Organization

This suite expects a machine with a certain directory structure

home, with subdirectories for IFS, ERA5, forecasts which has subdirectories for each YMD, and retrospective

draw an ascii diagram of the directory structure

```
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
            # Final products
            discharge.zarr
            warnings.parquet
            map_tables/
                YYYYMMDDHH.csv  # 1 for each timestep of the forecast
        # For example
        /20250101
        /20250102
        /20250103
        /20250104
        /20250105
    /retrospective
        hourly.zarr
        daily.zarr
        monthly-timeseries.zarr
        monthly-timesteps.zarr
        yearly-timeseries.zarr
        yearly-timesteps.zarr
        maximums.zarr
        return-periods.zarr
        /final_states
            final_states.parquet
```

## Log/Status Feeds

In addition to writing logging message to disc, status information is sent to the following locations:

- Teams channel webhook
- AWS CloudWatch logs

## Summary of computational steps

Each day, the following steps are performed in order:

### Phase 0: Preparation

1. Set environment variables
    1. YMD - The date of the day to be processed, usually today. In YYYYMMDD format.

### Phase 1: Download runoff data

Runoff data are cached. Check if the data are available, otherwise download them.

1. Download the latest ECMWF IFS runoff grid/mesh data for the date specified by YMD.
2. Download the latest ERA5 runoff grid/mesh data for the date specified by YMD.

### Phase 1: Daily forecast computations

1. Download the latest ECMWF IFS runoff grid/mesh data.
2. VPU level computations. Parallelize these jobs by VPU, but do not change the task order.
    1. Calculate catchment level volumes (python/calculate_catchment_volumes.py
    2. Route the volumes (python/route.py)
    3. Concatenate ensemble members into a single file (bash/concat_member_discharges.sh)
    4. Generate a table of summarized flows used to style the web map layer (python/generate_vpu_map_tables.py)
    5. Issue warnings by filtering the map summary tables (TODO)
3. Concatenate the VPU level results. Tasks may be computed in any order and/or simultaneously.
    1. Concatenate the VPU level routed discharge. Makes a Zarr dataset. (python/concatenate_vpu_discharge.py)
    2. Concatenate the VPU level map tables. Makes a directory of CSVs. (python/concatenate_vpu_map_tables.py)
    3. Concatenate the VPU level warnings (TODO)

### Phase 2: Update retrospective simulation

1. Download the latest day's ERA5 Runoff data
2. Calculate catchment level volumes
3. Route the volumes, initialized from the last time step of the retrospective simulation.
4. Resample the hourly discharge to daily average and, when necessary, monthly and yearly averages.

### Phase 3: Synchronize forecast initialization to retrospective simulation

1. Get the initialization from the last retrospective simulation updates
2. Get the first 24 hours of forecasted catchment volumes for the 5 days between the last retrospective timestep and the present date.
3. Reroute the forecasted volumes, but initialize at the last retrospective timestep.
4. Calculate the ensemble average of the rerouted forecasted volumes

### Phase 5: Export results to S3 archives

1. From the daily forecast computations
    1. Routed discharge - Zarr
    2. Map summary tables - CSV
    3. Warnings - Parquet
2. From the retrospective simulation update
    1. Final states - Parquet
    2. Routed discharge in hourly, daily, monthly, yearly averages - Zarr
3. From the rerouted forecasted volumes
    1. Rerouted forecasted discharge, 5 days only - Zarr

### Phase 6: Cleanup

1. Delete forecast directories dated __older than 5 days__
2. Delete runoff data dated __older than 5 days__
