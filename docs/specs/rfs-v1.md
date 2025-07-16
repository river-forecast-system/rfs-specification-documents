# RFS V1

Documentation for the River Forecast System version 1, RFS v1.

- Model Version: 1
- Launch Date: Q3 2019 ??
- End Date: Current

| Property                       | Value                              |
|--------------------------------|------------------------------------|
| Reference IFS Version          | 46r1                               |
| IFS Resolution                 | 0640grid, 0.2 degree, 18km equator |
| Forecast Length                | 15 days                            |
| Forecast Time Step             | 3 to 6 hours                       |
| Retrospective Start            | 1979-01-01                         |
| Retrospective Update           | Monthly on 1st 00:00 UTC           |
| Retrospective Native Time Step | Daily average                      |

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

The main features of v1 compared to the Streamflow Prediction Tool (SPT) instances

- Globally uniform hydrography with about 1.3 million streams
- Hydrography based on HydroSHEDS
- Forecasts compute automated and run in ECMWF supercomputing facility in Reading, UK instead of various SPT app installs
- API, web app, and compute workflow decoupled into 3 separate projects
- Introduce the ArcGIS living atlas web map layer
- Introduce geoglows Python package

## Available Products

- Hydrography: HydroShare https://hydroshare.org/resource/9241da0b1166492791381b48943c2b4a/
- Flood Forecasts: ECMWF FTP or API
- Retrospective Simulation: ECMWF FTP or API

### Hydrography

| Property            | Value       |
|---------------------|-------------|
| Approximate Streams | 1.3 Million |
| Reference Product   | HydroSHEDS  |

### Flood Forecast Products

| Product Type       | Time Step        | Format | Frequency | Description                                                 |
|--------------------|------------------|--------|-----------|-------------------------------------------------------------|
| Forecast Discharge | 3 hourly average | netCDF | Daily     | Daily average forecasted discharge for 15 days              |
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
- Grid: grid=F640 (approximately 0.2 degree uniform full grid)

Example MARS request

### Retrospective Simulation Products

| Product Type         | Time Step     | Format | Frequency | Description                                      |
|----------------------|---------------|--------|-----------|--------------------------------------------------|
| Daily Discharge      | daily average | netCDF | Daily     | Daily average retrospective simulation           |
| Return Periods       |               | netCDF | Once      | Return periods of retrospective simulation       |
| Flow Duration Curves |               | netCDF | Once      | Flow duration curves of retrospective simulation |

## Changelog
