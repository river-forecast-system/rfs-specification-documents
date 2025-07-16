# RFS V2

Documentation for the River Forecast System version 2, RFS v2.

- Model Version: 2
- Launch Date: 2024-07-01
- End Date: Current

| Property                       | Value                               |
|--------------------------------|-------------------------------------|
| Reference IFS Version          | 48r1                                |
| IFS Resolution                 | O1280 grid, 0.1 degree, 9km equator |
| Forecast Length                | 15 days                             |
| Forecast Time Step             | 3 hours                             |
| Retrospective Start            | 1940-01-01                          |
| Retrospective Update           | 1xWeekly, Sunday 00:00 UTC          |
| Retrospective Native Time Step | 1 hourly average                    |

## Journal Papers

- <mark>TBD</mark>

## Conceptual explanation of the model

RFS produces 4 main datasets:

1. Hydrography (static)
2. Routing configs (static)
3. Daily forecasts
4. Retrospective simulation

RFS data is available in the following ways:

1. https://hydroviewer.geoglows.org
2. https://geoglows.ecmwf.int
3. AWS S3 buckets
4. geoglows Python package
5. ArcGIS Living Atlas web map layer

The forecasts were originally initialized from the retrospective hourly simulation and then run in an open loop. Each forecast is initialized from the
ensemble average of the +24 hour value on the previous day's forecast. Forecasts are computed at the ECMWF computing facility in Bologna, Italy.

The retrospective simulation is computed at hourly resolution each week on Sunday at 00:00 UTC. The retrospective simulation was initialized from zero
on 1940-01-01, It is available up to 5-12 days lag from present. ERA5 has a minimum 5 day lag from real time so it has 5 days of lag on Sunday. The lag
grows until the next Sunday for up to 12 days lag at the worst case. Retrospective simulations are computed on AWS EC2 in the us-east-1d region.

## Model Version Improvements

The main changes in v2 compared to v1 are:

- Switching from a HydroSHEDS-like hydrography to a TDX-Hydro derivative hydrography
- Increasing the total stream count from 1.3 million to 6.8 million
- Upgrading to use the newest IFS version, 48R1, including using a 0.1 uniform grid
- Forecasts have a uniform 3-hourly time step rather than 3-6 hour variable
- Retrospective data are now available at hourly average time step natively (previously daily)
- Retrospective discharge are precomputed as hourly, daily, monthly, and yearly averages (previously only daily)
- New annual maximums retrospective product so the entire dataset does not need to be downloaded and filtered
- All datasets are distributed as Zarr datasets
- All datasets are distributed through AWS S3 Open Data Program
- Most of the code was rewritten for major performance gains
- Overwhelmingly simpler and faster data access
- Large expansion of training materials and webinars
- Implementation of download analytics to estimate download volumes and behaviors, not users

## Available Products

- Hydrography, configs, retrospective: [s3://geoglows-v2](http://geoglows-v2.s3-website-us-west-2.amazonaws.com/)
- Forecasts: [s3://geoglows-v2-forecasts](http://geoglows-v2-forecasts.s3-website-us-west-2.amazonaws.com/)
- Forecast map tables and records [s3://geoglows-v2-forecast-products](http://geoglows-v2-forecast-products.s3-website-us-west-2.amazonaws.com/)

### Hydrography

| Property            | Value       |
|---------------------|-------------|
| Approximate Streams | 6.8 Million |
| Reference Product   | TDX-Hydro   |

### Flood Forecast Products

| Product Type       | Time Step        | Format | Frequency | Description                                                 |
|--------------------|------------------|--------|-----------|-------------------------------------------------------------|
| Forecast Discharge | 3 hourly average | Zarr   | Daily     | Daily average forecasted discharge for 15 days              |
| Map Summary Tables |                  | CSV    | Hourly    | Summary tables for web map styling, 1 per forecast timestep |

Request parameters for ECMWF IFS data:

- Stream: enfo
- Types:
    - pf, perturbed forecast, 50 members
    - cf, control forecast, 1 member
    - hres, high resolution, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid: grid=F1280 (approximately 0.1 degree uniform full grid)

Example MARS request

```text
retrieve,
        date=$ens_ymd,
        time=$ens_base,
        stream=enfo,
        $ens_mars_expver,
        step=0/to/144/by/3,
        levtype=sfc,
        class=od,
        type=pf,
        number=$ens_members,
        param=205.128,
        grid=F1280,
        target="tmp1.grb"

retrieve,
        date=$ens_ymd,
        time=$ens_base,
        stream=enfo,
        $ens_mars_expver,
        step=150/to/360/by/6,
        levtype=sfc,
        class=od,
        type=pf,
        number=$ens_members,
        param=205.128,
        grid=F1280,
        target="tmp2.grb"
EOF
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

## Changelog
