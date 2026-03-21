---
name: nve
description: Query data from NVE (Norges vassdrags- og energidirektorat) — Norwegian Water Resources and Energy Directorate. Use this skill ALWAYS when the user asks about Norwegian hydropower plants (vannkraftverk), wind power (vindkraft), electricity certificates (elsertifikater), power production, energy data, reservoir levels (magasinstatistikk), flood warnings (flomvarsling), landslide warnings (jordskredvarsling), avalanche warnings (snøskredvarsel), or any energy/water infrastructure in Norway. Also trigger when user mentions NVE, api.nve.no, kraftverk, GWh, MW, or Norwegian energy statistics.
---

# NVE — Norwegian Water Resources and Energy Directorate API

## Overview

NVE provides APIs for energy infrastructure (power plants, wind farms, electricity certificates) and natural hazard warnings (flood, landslide, avalanche).

**IMPORTANT:** NVE hosts docs at `api.nve.no/doc/` but the actual API has TWO different base URLs depending on the service:

| Service type | Base URL | Status |
|---|---|---|
| Energy databases (vannkraft, vindkraft, elsertifikater) | `https://api.nve.no/web/` | ✅ Works, no auth needed |
| Hazard warnings (flom, jordskred, snøskred) | `https://api01.nve.no/` | ⚠️ Separate domain — may not be in allowed list |
| Hydrology (HydAPI) | `https://hydapi.nve.no/` | 🔑 Requires registration + separate domain |

**Format:** JSON (set `Accept: application/json`).
**Authentication:** None for /web/ endpoints. HydAPI requires registered API key.

## 1. Hydropower Plants (Vannkraftdatabase)

### Get all hydropower plants
```bash
curl -s "https://api.nve.no/web/Powerplant/GetHydroPowerPlants?page=1&pageSize=50" \
  -H "Accept: application/json"
```

### Get plants currently in operation
```bash
curl -s "https://api.nve.no/web/Powerplant/GetHydroPowerPlantsInOperation?page=1&pageSize=50" \
  -H "Accept: application/json"
```

**Response fields per plant:**
- `VannKraftverkID` — Unique ID
- `Navn` — Plant name
- `HovedEier` — Main owner
- `HovedEier_OrgNr` — Owner org number
- `Kommune` / `KommuneNr` — Municipality
- `Fylke` / `FylkesNr` — County
- `MaksYtelse` — Maximum output (MW)
- `MidProd_91_20` — Average production 1991-2020 (GWh)
- `BruttoFallhoyde_M` — Gross head (metres)
- `Slukeevne` — Maximum intake (m³/s)
- `EnEkv` — Energy equivalent
- `ElspotomraadeNummer` — Elspot price area (1-5)
- `ErIDrift` — Currently in operation (boolean)
- `IDriftDato` — Date put into operation
- `Eiere` — Array of owners with percentage shares
- `Konsesjoner` — Array of related concessions

**Norway has ~1,853 hydropower plants in this database.**

### Pagination
Use `page` and `pageSize` parameters. `pageSize` can be large — the API returns all results on page 1 if pageSize is big enough.

## 2. Wind Power Plants (Vindkraftdatabase)

### Get all wind power plants
```bash
curl -s "https://api.nve.no/web/WindPowerplant/GetWindPowerPlants?page=1&pageSize=100" \
  -H "Accept: application/json"
```

### Get wind plants in operation
```bash
curl -s "https://api.nve.no/web/WindPowerplant/GetWindPowerPlantsInOperation?page=1&pageSize=100" \
  -H "Accept: application/json"
```

**Norway has ~72 wind power plants registered.**

## 3. Electricity Certificates (Elsertifikater)

### Get all certified plants
```bash
curl -s "https://api.nve.no/web/ElCert/GetApplications?page=1&pageSize=50" \
  -H "Accept: application/json"
```

**Response fields:**
- `KraftverksNavn` — Plant name
- `TypeAnlegg` — Plant type (Vannkraft, Vindkraft, etc.)
- `Status` — Godkjent (approved), Avslått, etc.
- `GWh` — Annual production
- `MW` — Installed capacity
- `Idrift` — Date put into operation

### Get summary values
```bash
curl -s "https://api.nve.no/web/ElCert/GetSummaryValues" -H "Accept: application/json"
curl -s "https://api.nve.no/web/ElCert/GetSummaryValuesPerQuarter" -H "Accept: application/json"
```

**914 plants with electricity certificates registered.**

## 4. Hazard Warning APIs (api01.nve.no)

**NOTE:** These endpoints are at `api01.nve.no`, NOT `api.nve.no`. This domain may not be accessible depending on your network configuration. If it's blocked, inform the user and suggest they check varsom.no directly.

### Flood warnings (Flomvarsling)
```
Base: https://api01.nve.no/hydrology/forecast/flood/v1.0.10/api/
Swagger: https://api01.nve.no/hydrology/forecast/flood/v1.0.10/swagger/ui/index
```

### Landslide warnings (Jordskredvarsling)
```
Base: https://api01.nve.no/hydrology/forecast/landslide/v1.0.10/api/
```

### Avalanche warnings (Snøskredvarsel)
```
Base: https://api01.nve.no/hydrology/forecast/avalanche/v6.3.0/api/
```

Attribution for warnings: "Varsler fra Flomvarslingen i Norge og www.varsom.no"

## 5. Hydrology (HydAPI) — Requires Registration

**Base URL:** `https://hydapi.nve.no/`
**Documentation:** https://hydapi.nve.no/UserDocumentation/
**Registration:** Required (free) — register at NVE's portal.
**Data:** ~1,800 stations, ~8,500 time series. Vannstand, vannføring, snødybde, grunnvann, vanntemperatur.
**License:** NLOD / CC BY 3.0

If the user asks for historical water flow, water levels, or hydrological observation data, inform them that HydAPI requires a registered API key.

## License, Attribution & Rate Limits

- **License:** NLOD (Norsk lisens for offentlige data) / CC BY 3.0 for most data.
- **Attribution:** Credit NVE. For warnings: "Varsler fra Flomvarslingen i Norge og www.varsom.no".
- **Rate limiting:** No documented numeric limit for /web/ endpoints. Be reasonable.
- **Data accuracy:** Data is provided "as is" and may contain errors or missing values.
- **Warning data completeness:** NVE prefers that all warning data is presented — do not selectively filter out warnings based on your own judgment.
- **Registration:** Not needed for /web/ endpoints. Required for HydAPI.

## Common Recipes

### Find largest hydropower plants in Norway
```bash
curl -s "https://api.nve.no/web/Powerplant/GetHydroPowerPlantsInOperation?page=1&pageSize=2000" \
  -H "Accept: application/json"
```
Then sort by `MaksYtelse` (MW) in code.

### Find all power plants in a municipality
Fetch all plants, filter by `KommuneNr` in code. No server-side municipality filter available.

### Get wind power capacity
```bash
curl -s "https://api.nve.no/web/WindPowerplant/GetWindPowerPlantsInOperation?page=1&pageSize=200" \
  -H "Accept: application/json"
```
Sum `MaksYtelse` for total installed wind capacity.
