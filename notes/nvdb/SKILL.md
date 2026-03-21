---
name: nvdb
description: Query the Norwegian Road Database (NVDB / Nasjonal vegdatabank) via its REST API. Use this skill ALWAYS when the user asks about Norwegian roads, speed limits (fartsgrenser), road width, tunnels, bridges, guardrails (rekkverk), road surface, road objects, vegsystemreferanse, road categories (E-veg, riksveg, fylkesveg, kommunal veg), or any road infrastructure data in Norway. Also trigger when user mentions NVDB, Statens vegvesen API, nvdbapiles, vegobjekter, or asks about specific road properties along a Norwegian road. Covers vegobjekter (road objects with 424+ types), position lookups, road network reference system, and road object type discovery.
---

# NVDB — Norwegian Road Database API

## Overview

NVDB contains all registered data about the Norwegian road network — speed limits, road widths, surfaces, barriers, tunnels, bridges, signs, and 420+ other object types. Maintained by Statens vegvesen.

**Base URL:** `https://nvdbapiles.atlas.vegvesen.no`
**Authentication:** None, but set `X-Client: your-app` header.
**Format:** JSON.

## Core Concepts

### Road Reference System (Vegsystemreferanse)

Every point on the Norwegian road network has a reference like `EV6S21D1m12345`:
- `EV6` = Europaveg 6 (road category + number)
- `S21` = Strekning (section) 21
- `D1` = Delstrekning (subsection) 1
- `m12345` = Meter value along the section

**Road categories:**
| Prefix | Category |
|---|---|
| `EV` | Europaveg |
| `RV` | Riksveg |
| `FV` | Fylkesveg |
| `KV` | Kommunal veg |
| `PV` | Privat veg |
| `SV` | Skogsbilveg |

## Endpoints

### 1. Position Lookup — Find road at a coordinate

```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/posisjon?lat=59.911&lon=10.750&maks_avstand=50&srid=4326" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

Returns the road reference, road category, and position on the nearest road.

### 2. Road Object Types — Discover what's available

```bash
# List all object types
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekttyper" \
  -H "X-Client: skill-test" -H "Accept: application/json"

# Get details for one type
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekttyper/105" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

**Key object type IDs:**

| ID | Name | Description |
|---|---|---|
| 105 | Fartsgrense | Speed limits |
| 538 | Bomstasjon | Toll stations |
| 581 | Tunnel | Tunnels |
| 60 | Bru | Bridges |
| 5 | Rekkverk | Guardrails/barriers |
| 67 | Vegbredde | Road width |
| 241 | Vegdekke | Road surface |
| 95 | Skiltplate | Road signs |
| 532 | Trafikkulykke | Traffic accidents |
| 45 | Belysningspunkt | Street lighting |
| 446 | Fortau | Sidewalks |
| 874 | Sykkelfelt | Bicycle lanes |
| 291 | Gangfelt | Pedestrian crossings |
| 100 | Stigning/fall | Gradient |
| 904 | Skredsikring | Avalanche protection |
| 607 | Midtrekkverk | Median barrier |
| 14 | Vegoppmerking | Road markings |

### 3. Query Road Objects

```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/105?inkluder=egenskaper,lokasjon&segmentering=true&antall=10" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `inkluder` | string | Comma-sep: `egenskaper`, `lokasjon`, `metadata`, `relasjoner`, `geometri` |
| `segmentering` | boolean | Split objects at road reference boundaries |
| `antall` | integer | Results per page (max 10000) |
| `vegsystemreferanse` | string | Filter by road ref (e.g. `EV6`, `RV5S14`) |
| `kommune` | integer | Municipality number |
| `fylke` | integer | County number |
| `vegkategori` | string | Road category: `E`, `R`, `F`, `K`, `P`, `S` |
| `trafikantgruppe` | string | `K` (car) or `G` (pedestrian/cyclist) |
| `egenskap` | string | Filter by property (e.g. `2021=50` for speed limit 50) |
| `overlapp` | string | Overlap filter with another object type |

### Filter by properties (egenskap)

The `egenskap` parameter uses the format `propertyTypeId=value`:
```
# Speed limits of exactly 80 km/h
?egenskap=2021=80

# Speed limits >= 70
?egenskap=2021>=70

# Multiple conditions
?egenskap=(2021=80 AND 8791=1)
```

Find property IDs via the object type endpoint.

### 4. Get Single Object

```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/105/78697179?inkluder=alle" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

### 5. Road Network (Vegnett)

```bash
# Road links near a point
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegnett/lenkesekvenser/segmentert?kommune=4640&antall=10" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

## Response Structure

### Road object response:
```json
{
  "objekter": [
    {
      "id": 78697179,
      "href": "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/105/78697179",
      "egenskaper": [
        { "id": 2021, "navn": "Fartsgrense", "verdi": 50, "datatype": "Tall" }
      ],
      "lokasjon": {
        "kommuner": [301],
        "fylker": [3],
        "vegsystemreferanser": [
          { "vegsystem": { "vegkategori": "E", "fase": "V", "nummer": 6 },
            "strekning": { "strekning": 1, "delstrekning": 1, "meter": 100, "retning": "MED" } }
        ],
        "geometri": { "wkt": "POINT (263000 6649000)", "srid": 5973 }
      }
    }
  ],
  "metadata": { "antall": 1, "returnert": 1, "neste": { "href": "..." } }
}
```

**Note:** Geometry is in SRID 5973 (EUREF89 / UTM zone 33). Convert to WGS84 (4326) for mapping using Geonorge transformer.

## Pagination

Follow `metadata.neste.href` to get the next page. Maximum 10000 objects per request.

## Common Recipes

### Get all speed limits on E6 through Trondheim (kommune 5001)
```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/105?vegsystemreferanse=EV6&kommune=5001&inkluder=egenskaper,lokasjon&segmentering=true&antall=1000" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

### Find all tunnels in Vestland county
```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/581?fylke=46&inkluder=egenskaper,lokasjon&antall=100" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

### Find toll stations
```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/538?inkluder=egenskaper,lokasjon&antall=100" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

### Which road is at this coordinate?
```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/posisjon?lat=61.23&lon=7.10&maks_avstand=100&srid=4326" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

## License, Attribution & Rate Limits

- **License:** Norwegian Licence for Open Government Data (NLOD). Free for any purpose including commercial use.
- **Attribution required:** When using NVDB data, you MUST include: "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
- **X-Client header recommended:** Set `X-Client: <your-app-name>` and `X-Client-Session: <contact-email>` headers. Not strictly required for V4, but strongly recommended.
- **Rate limiting:** No documented numeric limit. Prefer real-time queries over bulk downloads. Use the `/vegobjekter/{id}/endringer` endpoint for incremental updates.
- **API version:** V3 is being shut down. Use V4 at `https://nvdbapiles.atlas.vegvesen.no/`. V3 requires authorised X-Client headers since September 2025.
- **Data accuracy:** Data may contain errors and omissions. Statens vegvesen does not guarantee content or currency. Safety-critical data (barriers, avalanche protection) may be incomplete.
- **Geometry SRID:** Responses use SRID 5973 (EUREF89 / UTM zone 33) by default, not WGS84. Convert when mapping.
- **Registration:** Not required for read access.
