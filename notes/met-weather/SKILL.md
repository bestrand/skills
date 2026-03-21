---
name: met-weather
description: Query the Norwegian Meteorological Institute (MET / Meteorologisk institutt) weather API. Use this skill ALWAYS when the user asks about weather forecasts in Norway or the Nordic region, weather data, temperature, precipitation, wind, weather warnings, sunrise/sunset times, or mentions api.met.no, yr.no, Meteorologisk institutt, or MET Norway. Also trigger for any location-based weather query where the location is in Scandinavia. Covers Locationforecast (hourly/daily forecasts), Nowcast (radar-based precipitation next 2 hours), Sunrise (sun/moon times), and Textforecast (Norwegian text forecasts).
---

# MET Weather API (api.met.no)

## Overview

MET Norway provides free weather APIs powering yr.no. All endpoints require a `User-Agent` header identifying your application.

**Base URL:** `https://api.met.no/weatherapi/`
**Authentication:** None, but **must set** `User-Agent` header.
**Format:** GeoJSON (Locationforecast), JSON, or XML.

## CRITICAL: User-Agent Required

Every request MUST include a descriptive User-Agent header:
```
User-Agent: my-skill/1.0 contact@example.com
```
Requests without a User-Agent will be blocked.

## Locationforecast 2.0 — Weather forecast

The primary endpoint. Returns hourly forecasts for ~48h and 6-hourly for 9 days.

### Compact (most common)
```bash
curl -s "https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```

### Complete (more parameters)
```bash
curl -s "https://api.met.no/weatherapi/locationforecast/2.0/complete?lat=59.91&lon=10.75&altitude=15" \
  -H "User-Agent: skill/1.0"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `lat` | float | Latitude (-90 to 90) |
| `lon` | float | Longitude (-180 to 180) |
| `altitude` | integer | Metres above sea level (optional, improves accuracy) |

**Response structure:**
```json
{
  "type": "Feature",
  "geometry": { "type": "Point", "coordinates": [10.75, 59.91, 15] },
  "properties": {
    "meta": { "updated_at": "2026-03-20T12:00:00Z", "units": {...} },
    "timeseries": [
      {
        "time": "2026-03-20T13:00:00Z",
        "data": {
          "instant": {
            "details": {
              "air_temperature": 5.4,
              "wind_speed": 2.7,
              "wind_from_direction": 210.0,
              "relative_humidity": 84.5,
              "air_pressure_at_sea_level": 1012.3,
              "cloud_area_fraction": 75.0,
              "dew_point_temperature": 3.1
            }
          },
          "next_1_hours": {
            "summary": { "symbol_code": "partlycloudy_day" },
            "details": { "precipitation_amount": 0.0 }
          },
          "next_6_hours": {
            "summary": { "symbol_code": "cloudy" },
            "details": {
              "precipitation_amount": 1.2,
              "air_temperature_max": 7.0,
              "air_temperature_min": 4.0
            }
          }
        }
      }
    ]
  }
}
```

**Key fields per timeseries entry:**
- `instant.details` — Current conditions at that time
  - `air_temperature` (°C), `wind_speed` (m/s), `wind_from_direction` (degrees)
  - `relative_humidity` (%), `air_pressure_at_sea_level` (hPa)
  - `cloud_area_fraction` (%), `dew_point_temperature` (°C)
  - `ultraviolet_index_clear_sky` (complete only)
- `next_1_hours` / `next_6_hours` — Forecast summaries
  - `symbol_code` — Weather icon (e.g. `clearsky_day`, `rain`, `heavysnow`)
  - `precipitation_amount` (mm)
  - `air_temperature_max` / `air_temperature_min` (6-hour only)

**Symbol codes** (common ones):
`clearsky_day`, `clearsky_night`, `fair_day`, `fair_night`, `partlycloudy_day`, `partlycloudy_night`, `cloudy`, `fog`, `lightrain`, `rain`, `heavyrain`, `lightrainshowers_day`, `rainshowers_day`, `heavyrainshowers_day`, `lightsleet`, `sleet`, `lightsnow`, `snow`, `heavysnow`, `thunder`, `rainandthunder`

## Nowcast 2.0 — Radar-based precipitation (next 2 hours)

Very high-resolution precipitation forecast for the Nordic region, updated every 5 minutes.

```bash
curl -s "https://api.met.no/weatherapi/nowcast/2.0/complete?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```

Same structure as Locationforecast but only covers ~2 hours ahead with 5-minute resolution.

## Sunrise 3.0 — Sun and moon times

```bash
curl -s "https://api.met.no/weatherapi/sunrise/3.0/sun?lat=59.91&lon=10.75&date=2026-03-20&offset=+01:00" \
  -H "User-Agent: skill/1.0"
```

**Parameters:** `lat`, `lon`, `date` (YYYY-MM-DD), `offset` (UTC offset, e.g. `+01:00`)

**Endpoints:** `/sun` (sunrise/sunset), `/moon` (moonrise/moonset/phase)

## Caching and Politeness

- Responses include `Expires` header — do NOT re-fetch before expiry.
- Locationforecast updates roughly hourly. Don't poll more often.
- Maximum 20 requests/second, but aim for much less.

## Common Recipes

### Get current weather for Oslo
```bash
curl -s "https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```
Parse `timeseries[0].data.instant.details` for current conditions.

### Get today's forecast summary
Take the first entry's `next_6_hours` for a quick overview (temp range, precipitation, symbol).

### Check if it will rain in the next 2 hours
Use Nowcast — check `precipitation_amount` in each 5-minute step.

### Get sunrise/sunset
```bash
curl -s "https://api.met.no/weatherapi/sunrise/3.0/sun?lat=59.91&lon=10.75&date=2026-03-20&offset=+01:00" \
  -H "User-Agent: skill/1.0"
```

## Coordinates for common Norwegian cities

| City | Lat | Lon |
|---|---|---|
| Oslo | 59.91 | 10.75 |
| Bergen | 60.39 | 5.32 |
| Trondheim | 63.43 | 10.40 |
| Stavanger | 58.97 | 5.73 |
| Tromsø | 69.65 | 18.96 |
| Bodø | 67.28 | 14.40 |
| Kristiansand | 58.15 | 8.00 |
| Drammen | 59.74 | 10.20 |
| Sogndal | 61.23 | 7.10 |
| Ålesund | 62.47 | 6.15 |

## License, Attribution & Rate Limits

- **License:** Creative Commons Attribution 4.0 International (CC BY 4.0), also NLOD. Free for any purpose including commercial use.
- **Attribution required:** You must credit MET Norway. Suggestions: "Data from MET Norway" or "Based on data from MET Norway". A link to the data source is appreciated.
- **User-Agent MANDATORY:** Every request MUST include a `User-Agent` header with a contact email or app name (e.g. `MyApp/1.0 user@example.com`). Requests without identification will be blocked (HTTP 403).
- **Rate limiting:** No fixed numeric limit published, but the service will throttle (HTTP 429) if you generate excessive traffic. Always respect `Expires` headers — do NOT re-fetch before expiry. High-traffic users must set up a caching proxy.
- **Caching:** Locationforecast updates roughly hourly. Nowcast updates every 5 minutes. Cache responses and use `If-Modified-Since` headers.
- **Deprecation signals:** HTTP 203 (instead of 200) means the product version is deprecated or in beta. Log this as a warning.
- **Privacy:** If calling the API directly from a browser/app, the user's IP and coordinates are logged. Use a backend proxy for privacy.
- **Weather icons:** Licensed under MIT License (© 2015-2017 Yr.no). Free to use.
- **Bans:** Deliberate breach of ToS, circumventing rate limits, or impersonating other clients will result in a permanent ban.

## Frost API (frost.met.no) — Historical Observations

The Frost API provides historical weather observations from ~1,800 stations. **This requires registration** — it is NOT covered by this skill's endpoints.

- **Registration:** Free at https://frost.met.no/auth/requestCredentials.html
- **Auth:** HTTP Basic with client_id as username, empty password
- **License:** CC BY 3.0 NO
- **Data:** Observations from the 1800s to present — temperature, precipitation, wind, snow depth, etc.
- **Domain:** `frost.met.no` (separate from `api.met.no`)

If the user asks about historical weather data or observations (not forecasts), inform them that Frost requires a registered client_id.
