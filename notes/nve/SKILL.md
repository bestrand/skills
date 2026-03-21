---
name: nve
description: Query data from NVE (Norges vassdrags- og energidirektorat) — Norwegian Water Resources and Energy Directorate. Use this skill ALWAYS when the user asks about Norwegian hydropower plants (vannkraftverk), wind power (vindkraft), electricity certificates (elsertifikater), power production, energy data, reservoir levels (magasinstatistikk), flood warnings (flomvarsling), landslide warnings (jordskredvarsling), avalanche warnings (snøskredvarsel), or any energy/water infrastructure in Norway. Also trigger when user mentions NVE, api.nve.no, kraftverk, GWh, MW, or Norwegian energy statistics.
---

# NVE — Energy and Hazard Warning APIs

**Auth:** None for /web/ endpoints. **License:** NLOD / CC BY 3.0 — credit NVE.
**Rate limit:** No documented limit for /web/ endpoints. Be reasonable.
**Warning data:** NVE requires that all warning data is presented — do not selectively filter out warnings.

Two base URLs depending on service:

| Service | Base URL | Notes |
|---|---|---|
| Energy databases | `https://api.nve.no/web/` | No auth, works |
| Hazard warnings | `https://api01.nve.no/` | Separate domain — may be blocked by network config |
| Hydrology (HydAPI) | `https://hydapi.nve.no/` | Requires registration |

## 1. Hydropower (Vannkraft)

```bash
# All plants in operation (~1,853 plants)
curl -s "https://api.nve.no/web/Powerplant/GetHydroPowerPlantsInOperation?page=1&pageSize=5" \
  -H "Accept: application/json"

# All registered plants (including decommissioned)
curl -s "https://api.nve.no/web/Powerplant/GetHydroPowerPlants?page=1&pageSize=5" \
  -H "Accept: application/json"
```

**IMPORTANT: Pagination is unreliable.** The `page` and `pageSize` parameters exist but the API often returns the full dataset regardless of `pageSize`. Expect ~2MB responses. Sort/filter client-side.

**Key response fields:** `Navn`, `HovedEier`, `HovedEier_OrgNr`, `Kommune`/`KommuneNr`, `Fylke`/`FylkesNr`, `MaksYtelse` (MW), `MidProd_91_20` (GWh avg 1991–2020), `BruttoFallhoyde_M`, `Slukeevne` (m³/s), `ElspotomraadeNummer` (price area 1–5), `ErIDrift`, `IDriftDato`, `Eiere[]` (with percentage shares)

**No server-side filtering.** To find plants in a specific municipality or county, fetch all and filter by `KommuneNr` or `FylkesNr` in code.

## 2. Wind Power (Vindkraft)

```bash
curl -s "https://api.nve.no/web/WindPowerplant/GetWindPowerPlantsInOperation?page=1&pageSize=200" \
  -H "Accept: application/json"
```

~65 wind power plants registered. Same pagination caveat as hydropower.

## 3. Electricity Certificates (Elsertifikater)

```bash
# All certified plants (~914 registered)
curl -s "https://api.nve.no/web/ElCert/GetApplications?page=1&pageSize=50" \
  -H "Accept: application/json"
```

**Fields:** `KraftverksNavn`, `TypeAnlegg` (Vannkraft, Vindkraft, etc.), `Status` (Godkjent, Avslått), `GWh`, `MW`, `Idrift`

**Note:** `GetSummaryValues` is broken (HTTP error via internal redirect). Use `GetSummaryValuesPerQuarter` instead:
```bash
curl -s "https://api.nve.no/web/ElCert/GetSummaryValuesPerQuarter" -H "Accept: application/json"
```
Returns quarterly array with `Year`, `Quarter`, `Values.IMaalet`, `Values.IOvergangsordningen`.

## 4. Hazard Warnings (api01.nve.no)

**Domain:** `api01.nve.no` — may not be in your allowed domains list. If blocked, direct users to varsom.no.

### Flood warnings
```
GET https://api01.nve.no/hydrology/forecast/flood/v1.0.10/api/Warning/County/{countyNr}/{dayOffset}
```
`dayOffset`: 0=today, 1=tomorrow, 2=day after

### Landslide warnings
```
GET https://api01.nve.no/hydrology/forecast/landslide/v1.0.10/api/Warning/County/{countyNr}/{dayOffset}
```

### Avalanche warnings
```
GET https://api01.nve.no/hydrology/forecast/avalanche/v6.3.0/api/
```

**Response key fields:** `ActivityLevel` (1=green, 2=yellow, 3=orange, 4=red), `MainText`, `ValidFrom`/`ValidTo`, `MunicipalityList[]`

Attribution: "Varsler fra Flomvarslingen i Norge og www.varsom.no"

## HydAPI — Historical Observations

Requires free registration at https://hydapi.nve.no/. Provides historical water flow, levels, snow depth, temperature from ~1,800 stations. Not usable without a client key.
