# GeoPressureAPI

[![Test Server](https://github.com/Rafnuss/GeoPressureAPI/actions/workflows/test-server.yml/badge.svg)](https://github.com/Rafnuss/GeoPressureAPI/actions/workflows/test-server.yml)

## Overview

GeoPressureAPI is a JSON API that enables computation of pressure mismatch between geolocator pressure timeseries and atmospheric pressure from [ERA5-LAND reanalysis data](https://cds.climate.copernicus.eu/cdsapp#!/dataset/reanalysis-era5-land).

This documentation describes how to use the GeoPressure API. Please [file an issue](https://github.com/Rafnuss/GeoPressureAPI/issues/new) if you find anything missing.

## Available Endpoints

The GeoPressure API provides four main endpoints:

| Endpoint                                                      | Description                                         |
| ------------------------------------------------------------- | --------------------------------------------------- |
| **[Pressure Map](#1-pressure-map)**                           | Compute pressure mismatch maps from geolocator data |
| **[Pressure Timeseries](#2-pressure-timeseries)**             | Extract pressure timeseries at specific locations   |
| **[Ground Elevation Path](#3-ground-elevation-along-a-path)** | Get elevation profiles along polylines              |
| **[Pressure Data Path](#4-pressure-data-along-a-path)**       | Extract atmospheric variables along paths           |

---

## 1. Pressure Map

**Endpoint:** `POST /glp.mgravey.com/GeoPressure/v2/map/`

### Overview

Compute maps of pressure mismatch from geolocator pressure timeseries. Returns GeoTIFF maps with pressure mismatch analysis.

### Features

- **MSE Layer**: Mean Square Error between geolocator and ERA5 pressure data (with mean error removed for altitude flexibility)
- **Mask Layer** _(optional)_: Proportion of timeseries within ground elevation range using barometric formula and SRTM data
- **Optimization**: Use `maskThreshold` to filter pixels and reduce computation time
- **Data Source**: ERA5-Land (1981 to 3 months from real-time), 1-hour resolution

### Output Layers

1. **MSE**: Pressure mismatch computed using [Mean Square Error](https://en.wikipedia.org/wiki/Mean_squared_error)
2. **Mask**: Altitude feasibility based on [barometric formula](https://en.wikipedia.org/wiki/Barometric_formula) and [SRTM-90](https://developers.google.com/earth-engine/datasets/catalog/CGIAR_SRTM90_V4) elevation data

### Request Parameters

| Parameter       | Type                 | Required | Default | Description                                                                           |
| --------------- | -------------------- | -------- | ------- | ------------------------------------------------------------------------------------- |
| `W`             | `number`             | ✅       |         | West coordinate (-180° to 180°)                                                       |
| `S`             | `number`             | ✅       |         | South coordinate (-90° to 90°)                                                        |
| `E`             | `number`             | ✅       |         | East coordinate (-180° to 180°)                                                       |
| `N`             | `number`             | ✅       |         | North coordinate (-90° to 90°)                                                        |
| `pressure`      | `number[]`           | ✅       |         | Atmospheric pressure values (Pascal)                                                  |
| `time`          | `number[]`           | ✅       |         | [UNIX timestamps](https://en.wikipedia.org/wiki/Unix_time) (seconds since 1970-01-01) |
| `label`         | `(string\|number)[]` | ✅       |         | Grouping labels for pressure data                                                     |
| `scale`         | `number`             |          | `10`    | Pixels per degree (10 = 0.1°/~10km, 4 = 0.25°/~30km)                                  |
| `maxSample`     | `number`             |          | `250`   | Maximum datapoints sampled per label; `max_sample` is also accepted                    |
| `margin`        | `number`             |          | `30`    | Altitude error margin in meters (1hPa ≈ 10m)                                          |
| `includeMask`   | `boolean`            |          | `true`  | Include mask layer in output                                                          |
| `maskThreshold` | `number`             |          | `0.9`   | Filter pixels by mask value (0-1, e.g., 0.9 for 90%+ feasibility)                     |
| `landDensityThreshold` | `number`       |          | `0`     | Minimum land-density threshold applied to ERA5 pixels                                 |

### Response Format

| Field           | Type                 | Description                                    |
| --------------- | -------------------- | ---------------------------------------------- |
| `status`        | `string`             | `"success"` or `"error"`                       |
| `taskID`        | `number`             | Unique task identifier                         |
| `labels`        | `(string\|number)[]` | Unique labels in same order as URLs            |
| `urls`          | `string[]`           | Download URLs for GeoTIFF files (`null` on failure) |
| `errors`        | `(string\|null)[]`   | Reason each `null` url failed, same order as `urls` |
| `resolution`    | `number`             | Map resolution in degrees                      |
| `size`          | `number[]`           | Map dimensions [width, height]                 |
| `bbox`          | `object`             | Bounding box coordinates                       |
| `includeMask`   | `boolean`            | Whether mask layer is included                 |
| `maskThreshold` | `number`             | Applied mask threshold                         |
| `errorMessage`  | `string`             | Error description (if status = "error")        |
| `advice`        | `string`             | Troubleshooting guidance (if status = "error") |

### GeoTIFF Content

Each URL returns a [GeoTIFF file](https://en.wikipedia.org/wiki/GeoTIFF) with:

- **Band 1**: MSE values
- **Band 2**: Mask values _(if `includeMask` = true)_

**Special values:**

- `-2`: Water/no data areas
- `-1`: Pixels below mask threshold

### Example Usage

```http
POST /glp.mgravey.com/GeoPressure/v2/map/
Content-Type: application/json

{
  "W": -18,
  "S": 4,
  "E": 16,
  "N": 51,
  "time": [1572075000, 1572076800, 1572078600],
  "pressure": [97766, 97800, 97833],
  "label": [1, 1, 1]
}
```

**Response:**

```json
{
  "status": "success",
  "taskID": 1639259414,
  "data": {
    "labels": [1],
    "urls": ["https://earthengine.googleapis.com/v1alpha/..."],
    "resolution": 0.25,
    "size": [136, 188],
    "bbox": { "W": -18, "S": 4, "E": 16, "N": 51 },
    "time2GetUrls": 11.61,
    "includeMask": true,
    "maskThreshold": 0.9
  }
}
```

---

## 2. Pressure Timeseries

**Endpoint:** `POST /glp.mgravey.com/GeoPressure/v2/timeseries/`

### Overview

Extract pressure timeseries at a specific location, with optional altitude computation when geolocator pressure data is provided.

### Features

- **Pressure extraction** at precise coordinates
- **Altitude computation** using barometric formula (when pressure provided)
- **Land fallback**: Automatically moves water coordinates to nearest land
- **Flexible time ranges**: Use time arrays or start/end timestamps

### Request Parameters

| Parameter   | Type       | Required | Description                                       |
| ----------- | ---------- | -------- | ------------------------------------------------- |
| `lon`       | `number`   | ✅       | Longitude coordinate (-180° to 180°)              |
| `lat`       | `number`   | ✅       | Latitude coordinate (-90° to 90°)                 |
| `pressure`  | `number[]` | ⚠️\*     | Geolocator pressure values (Pascal)               |
| `time`      | `number[]` | ⚠️\*     | UNIX timestamps (required if `pressure` provided) |
| `startTime` | `number`   | ⚠️\*\*   | Start timestamp (required if no `pressure`)       |
| `endTime`   | `number`   | ⚠️\*\*   | End timestamp (required if no `pressure`)         |

_\* Required together_  
_\*\* Required if `pressure` not provided_

### Response Format

| Field       | Type     | Description                                            |
| ----------- | -------- | ------------------------------------------------------ |
| `status`    | `string` | `"success"` or `"error"`                               |
| `taskID`    | `number` | Unique task identifier                                 |
| `url`       | `string` | CSV download URL                                       |
| `distInter` | `number` | Distance to nearest land (meters, if moved from water) |
| `lon`       | `number` | Actual longitude used                                  |
| `lat`       | `number` | Actual latitude used                                   |

### CSV Output Format

| Column     | Description                                                |
| ---------- | ---------------------------------------------------------- |
| `time`     | UNIX timestamps                                            |
| `pressure` | ERA5 pressure values                                       |
| `altitude` | Computed altitude _(only if geolocator pressure provided)_ |

### Example Usage

```http
POST /glp.mgravey.com/GeoPressure/v2/timeseries/
Content-Type: application/json

{
  "lon": 6,
  "lat": 46,
  "startTime": 1497916800,
  "endTime": 1500667800
}
```

**Response:**

```json
{
  "status": "success",
  "taskID": 1639259414,
  "data": {
    "url": "https://earthengine.googleapis.com/v1alpha/...",
    "format": "csv"
  }
}
```

---

## 3. Ground Elevation Along a Path

**Endpoint:** `POST /glp.mgravey.com/GeoPressure/v2/elevationPath/`

### Overview

Extract ground elevation statistics from [SRTM-90](https://developers.google.com/earth-engine/datasets/catalog/CGIAR_SRTM90_V4) data along a polyline path.

### Request Parameters

| Parameter       | Type       | Required | Default        | Description                                   |
| --------------- | ---------- | -------- | -------------- | --------------------------------------------- |
| `lon`           | `number[]` | ✅       |                | Longitude coordinates (-180° to 180°)         |
| `lat`           | `number[]` | ✅       |                | Latitude coordinates (-90° to 90°)            |
| `scale`         | `number`   | ✅       |                | Elevation data resolution (pixels per degree) |
| `samplingScale` | `number`   | ✅       |                | Path sampling resolution (pixels per degree)  |
| `percentile`    | `number[]` |          | `[10, 50, 90]` | Elevation percentiles to compute (0-100)      |

### Response Format

| Field            | Type       | Description                               |
| ---------------- | ---------- | ----------------------------------------- |
| `status`         | `string`   | `"success"` or `"error"`                  |
| `taskID`         | `number`   | Unique task identifier                    |
| `percentileData` | `object`   | Elevation statistics and path information |
| `scale`          | `number`   | Elevation data scale (meters per pixel)   |
| `samplingScale`  | `number`   | Path sampling scale (meters per pixel)    |
| `percentile`     | `number[]` | Computed percentiles                      |

### Percentile Data Object

| Array                  | Description                             |
| ---------------------- | --------------------------------------- |
| `"10"`, `"50"`, `"90"` | Elevation values for each percentile    |
| `distance`             | Cumulative distance along path (meters) |
| `lat`, `lon`           | Resampled coordinates                   |
| `stapId`               | Step position along path                |

### Example Usage

```http
POST /glp.mgravey.com/GeoPressure/v2/elevationPath/
Content-Type: application/json

{
  "lon": [8.47, 9.41, 9.01, -0.91, 14.24, 27.30, 34.39, 30.00],
  "lat": [48.89, 44.78, 40.07, 37.68, 17.33, 7.32, 8.09, -23.13],
  "scale": 10,
  "samplingScale": 1,
  "percentile": [10, 50, 90]
}
```

**Response:**

```json
{
  "status": "success",
  "taskID": 1639259414,
  "data": {
    "percentileData": {
      "10": [0, 0, 0, 305, 357, 289, 426, 399],
      "50": [237, 237, 587, 552, 551, 363, 569, 553],
      "90": [880, 880, 1260, 1138, 859, 480, 900, 756],
      "distance": [0, 462486, 986718, 1886940, 2997968],
      "lat": [44.927, 44.927, 34.943, 24.959, 14.976],
      "lon": [4.992, 4.992, -4.992, 4.992, 14.976],
      "stapId": [0, 1, 2, 3, 4]
    },
    "scale": 1111390.0,
    "samplingScale": 1111390.0,
    "percentile": [10, 50, 90]
  }
}
```

---

## 4. Pressure Data Along a Path

**Endpoint:** `POST /glp.mgravey.com/GeoPressure/v2/pressurePath/`

### Overview

Extract atmospheric variables from ERA5/ERA5-LAND data along a path with optional altitude computation from geolocator pressure data.

### Features

- **Multi-variable extraction**: Any ERA5 atmospheric variables
- **Dataset options**: ERA5 single-levels (default), ERA5-LAND, or combined
- **Altitude computation**: When geolocator pressure provided
- **Parallel processing**: Configurable workers for large datasets

> [!WARNING]
> **Use `dataset="single-levels"` (the default) whenever you need `altitude`.**
> ERA5-LAND's `surface_pressure` is not hydrostatically consistent with the orography
> ERA5-LAND publishes — the two disagree by up to ~10 hPa in steep terrain. The
> orography term therefore fails to cancel out of the barometric formula and the error
> passes straight into the retrieved altitude. Measured against 41,653 hourly
> station-pressure observations from 271 NOAA ISD stations (2–3576 m, Alps, July 2020)
> — every station-hour where both products have data:
>
> | `dataset` | bias | MAE | RMSE |
> | --------- | ---- | --- | ---- |
> | `"single-levels"` | −0.6 m | **9.0 m** | 27.4 m |
> | `"land"` / `"both"` | +2.5 m | **55.3 m** | 76.9 m |
>
> `"land"` and `"both"` are retained for backward compatibility and remain fine for
> variables other than `altitude`. When `pressure` is supplied together with one of
> them, the response carries an extra `warning` field. ERA5-LAND has no atmospheric
> analysis of its own — it is forced by ERA5 — so it holds no independent information
> about absolute altitude.

### Request Parameters

| Parameter  | Type       | Required | Default  | Description                                                                                    |
| ---------- | ---------- | -------- | -------- | ---------------------------------------------------------------------------------------------- |
| `lon`      | `number[]` | ✅       |          | Longitude coordinates (-180° to 180°)                                                          |
| `lat`      | `number[]` | ✅       |          | Latitude coordinates (-90° to 90°)                                                             |
| `time`     | `number[]` | ✅       |          | UNIX timestamps (must match coordinate array length)                                           |
| `variable` | `string[]` | ✅       |          | [ERA5 variable names](https://cds.climate.copernicus.eu/cdsapp#!/dataset/reanalysis-era5-land) |
| `pressure` | `number[]` |          |          | Geolocator pressure (Pascal, enables altitude computation)                                     |
| `dataset`  | `string`   |          | `"single-levels"` | Data source: `"single-levels"`, `"land"`, or `"both"`. Use `"single-levels"` for altitude (see warning above) |
| `workers`  | `number`   |          | `10`     | Parallel processing chunks                                                                     |

### Response Format

| Field    | Type     | Description                 |
| -------- | -------- | --------------------------- |
| `status`  | `string` | `"success"` or `"error"`                                              |
| `taskID`  | `number` | Unique task identifier                                                |
| `data`    | `object` | Variable arrays (see below)                                           |
| `warning` | `string` | _(only present)_ when `altitude` was computed from a `dataset` that cannot support it |

### Data Object Arrays

| Array        | Description                                 |
| ------------ | ------------------------------------------- |
| `time`       | UNIX timestamps (closest ERA5 match)        |
| `altitude`   | Computed altitudes _(if pressure provided)_ |
| `{variable}` | One array per requested variable            |

### Example Usage

```http
POST /glp.mgravey.com/GeoPressure/v2/pressurePath/
Content-Type: application/json

{
  "lon": [17.5, 17.5, 17.5, 41.6, 41.6],
  "lat": [48.5, 48.5, 48.5, 41.6, 41.6],
  "time": [1501113600, 1501115400, 1501117200, 1501745400, 1501747200],
  "variable": ["surface_pressure", "temperature_2m"],
  "dataset": "single-levels",
  "pressure": [98900, 99200, 99400, 100000, 100100],
  "workers": 1
}
```

**Response:**

```json
{
  "status": "success",
  "taskID": 1639259414,
  "data": {
    "time": [1501113600, 1501115400, 1501117200, 1501745400, 1501747200],
    "altitude": [1234.5, 1245.2, 1256.8, 1290.7, 1301.2],
    "surface_pressure": [98765, 98823, 98881, 99055, 99113],
    "temperature_2m": [285.4, 286.1, 286.8, 288.9, 289.6]
  }
}
```

---

## Installation

The map service currently uses the standard Google Earth Engine endpoint rather than
the high-volume endpoint.

### Server Setup

1. **Clone repository**

   ```bash
   git clone https://github.com/Rafnuss/GeoPressureAPI
   ```

2. **Add authentication**

   - Place Google Earth Engine service account JSON key in repository

3. **Configure server**

   - Update `bootServer.sh` with service address
   - Create `logs` directory

4. **Network routing** _(if needed)_

   ```bash
   sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 80
   sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 443
   ```

5. **Start server**
   ```bash
   bash bootServer.sh
   ```

---

## Data Sources & Limitations

- **ERA5-LAND**: 1981 to ~3 months from real-time
- **ERA5 single-levels**: 1940 to ~5 days from real-time
- **Time resolution**: 1 hour (use closest match for any timestamp)
- **Spatial coverage**: Global land areas
- **Coordinate systems**: WGS84 (EPSG:4326)

### Accuracy of retrieved altitude

Validated against real barometers (293 NOAA ISD stations, 2–3576 m, 44,163 hourly
observations). With `dataset="single-levels"` the error separates into two parts that
behave very differently:

| Component | Size | Notes |
| --------- | ---- | ----- |
| Static per-site offset | median 3.7 m, p90 15 m | station metadata + sub-grid terrain; constant in time |
| Temporal scatter | median SD **3.1 m**, p90 7.7 m | the actual reanalysis error |

So expect **~3 m for relative altitude changes at a fixed location** and **~10 m mean
absolute error for absolute altitude**, degrading to tens of metres in steep terrain —
and that last part is a fixed offset, not noise. Precision is nearly independent of
flight altitude: de-biased RMSE stays 2–7 m up to 1000 m above the model surface and
~12 m at 1000–3000 m above it.

For more information, see:

- [ERA5-LAND documentation](https://cds.climate.copernicus.eu/cdsapp#!/dataset/reanalysis-era5-land)
- [Google Earth Engine ERA5 dataset](https://developers.google.com/earth-engine/datasets/catalog/ECMWF_ERA5_LAND_HOURLY)
