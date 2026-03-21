---
name: met-weather
description: Query the Norwegian Meteorological Institute (MET / Meteorologisk institutt) weather API. Use this skill ALWAYS when the user asks about weather forecasts in Norway or the Nordic region, weather data, temperature, precipitation, wind, sunrise/sunset times, or mentions api.met.no, yr.no, Meteorologisk institutt, or MET Norway.
---

# MET Weather API (api.met.no)

**Base URL:** `https://api.met.no/weatherapi/`
**Auth:** None, but **must set** `User-Agent: your-app/1.0 contact@example.com` on every request (HTTP 403 without it).
**License:** CC BY 4.0 — credit "Data from MET Norway".
**Rate limit:** No published numeric limit, but throttles (HTTP 429) on excessive traffic. Respect `Expires` header — don't refetch before expiry. Locationforecast updates ~hourly, Nowcast every 5min. HTTP 203 = deprecated endpoint version (log as warning). Deliberate ToS breach or impersonation → permanent ban.

## Locationforecast — Weather forecast

```bash
curl -s "https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```

**Parameters:** `lat`, `lon`, `altitude` (metres, optional — improves accuracy)

Returns GeoJSON with `properties.timeseries[]` — hourly for ~48h, then 6-hourly for 9 days.

**Per timeseries entry:**
- `time` — ISO datetime
- `data.instant.details` — `air_temperature` (°C), `wind_speed` (m/s), `wind_from_direction` (°), `relative_humidity` (%), `air_pressure_at_sea_level` (hPa), `cloud_area_fraction` (%)
- `data.next_1_hours` — `summary.symbol_code`, `details.precipitation_amount` (mm)
- `data.next_6_hours` — same as next_1_hours. **Note:** `air_temperature_max`/`_min` are only in the `complete` endpoint, not `compact`.

**For temp range forecasts**, use `complete` instead:
```bash
curl -s "https://api.met.no/weatherapi/locationforecast/2.0/complete?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```

**Common symbol codes:** `clearsky_day`, `fair_day`, `partlycloudy_day`, `cloudy`, `fog`, `lightrain`, `rain`, `heavyrain`, `lightsnow`, `snow`, `heavysnow`, `sleet`, `thunder`, `rainandthunder` (append `_night` for night variants)

**Current weather:** Parse `timeseries[0].data.instant.details`.
**Today's summary:** First entry's `next_6_hours`.

## Nowcast — Radar precipitation (next 2h)

```bash
curl -s "https://api.met.no/weatherapi/nowcast/2.0/complete?lat=59.91&lon=10.75" \
  -H "User-Agent: skill/1.0"
```
5-minute resolution, same structure. Check `precipitation_amount` in each step.

## Sunrise / Sunset

```bash
curl -s "https://api.met.no/weatherapi/sunrise/3.0/sun?lat=59.91&lon=10.75&date=2026-03-21&offset=+01:00" \
  -H "User-Agent: skill/1.0"
```
Returns `properties.sunrise.time`, `properties.sunset.time`, `properties.solarnoon`. Moon: use `/moon` endpoint.

## Common coordinates

| City | Lat | Lon |
|---|---|---|
| Oslo | 59.91 | 10.75 |
| Bergen | 60.39 | 5.32 |
| Trondheim | 63.43 | 10.40 |
| Stavanger | 58.97 | 5.73 |
| Tromsø | 69.65 | 18.96 |
| Sogndal | 61.23 | 7.10 |

## Frost API (historical observations)

Separate service at `frost.met.no`. **Requires free registration.** Observations from ~1,800 stations (1800s–present). Not usable without client_id.
