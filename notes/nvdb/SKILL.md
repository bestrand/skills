---
name: nvdb
description: Query the Norwegian Road Database (NVDB / Nasjonal vegdatabank) via its REST API. Use this skill ALWAYS when the user asks about Norwegian roads, speed limits (fartsgrenser), road width, tunnels, bridges, guardrails (rekkverk), road surface, road objects, vegsystemreferanse, road categories (E-veg, riksveg, fylkesveg, kommunal veg), or any road infrastructure data in Norway. Also trigger when user mentions NVDB, Statens vegvesen API, nvdbapiles, vegobjekter.
---

# NVDB — Norwegian Road Database API

**Base URL:** `https://nvdbapiles.atlas.vegvesen.no`
**Auth:** None. Set `X-Client: your-app` and `X-Client-Session: contact@email` headers (recommended).
**License:** NLOD — free for any purpose.
**Attribution (required):** "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
**Rate limit:** No documented limit. Prefer real-time queries over bulk downloads.
**Geometry SRID:** Responses use SRID 5973 (UTM33) by default, not WGS84. Convert for mapping.

## Road Reference System

Every point has a reference like `EV6S21D1m12345`: `EV6` = road, `S21` = section, `D1` = subsection, `m12345` = metre.

| Prefix | Category |
|---|---|
| `EV` | Europaveg |
| `RV` | Riksveg |
| `FV` | Fylkesveg |
| `KV` | Kommunal veg |
| `PV` | Privat veg |

## Position lookup — Coordinate to road

```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/posisjon?lat=59.911&lon=10.750&maks_avstand=50&srid=4326" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```
Returns nearest road's reference and position. **Note:** `vegsystemreferanse` may be empty if the coordinate isn't on a classified road — increase `maks_avstand` or check the coordinate.

## Road Object Types

424 types available. Key IDs:

| ID | Name |
|---|---|
| 105 | Fartsgrense (speed limits) |
| 538 | Bomstasjon (toll stations) |
| 581 | Tunnel |
| 60 | Bru (bridges) |
| 5 | Rekkverk (guardrails) |
| 67 | Vegbredde (road width) |
| 241 | Vegdekke (road surface) |
| 95 | Skiltplate (road signs) |
| 446 | Fortau (sidewalks) |
| 874 | Sykkelfelt (bicycle lanes) |
| 904 | Skredsikring (avalanche protection) |

Discover all: `GET /vegobjekttyper`
Type details: `GET /vegobjekttyper/105` (shows property IDs for filtering)

## Query Road Objects

```bash
curl -s "https://nvdbapiles.atlas.vegvesen.no/vegobjekter/105?vegsystemreferanse=EV6&kommune=5001&inkluder=egenskaper,lokasjon&segmentering=true&antall=100" \
  -H "X-Client: skill-test" -H "Accept: application/json"
```

**Parameters:** `inkluder` (egenskaper, lokasjon, metadata, relasjoner, geometri), `segmentering` (boolean), `antall` (max 10000), `vegsystemreferanse` (e.g. `EV6`, `FV55`), `kommune` (number), `fylke` (number), `vegkategori` (E, R, F, K), `trafikantgruppe` (K=car, G=pedestrian), `egenskap` (property filter, e.g. `2021=80` for 80km/h)

### Property filter syntax
```
?egenskap=2021=80           # Speed exactly 80
?egenskap=2021>=70          # Speed >= 70
?egenskap=(2021=80 AND 8791=1)  # Multiple conditions
```
Find property IDs via `/vegobjekttyper/{id}`.

### Response
```json
{
  "objekter": [{
    "id": 78697179,
    "egenskaper": [{ "id": 2021, "navn": "Fartsgrense", "verdi": 50 }],
    "lokasjon": {
      "kommuner": [301],
      "vegsystemreferanser": [{ "vegsystem": { "vegkategori": "E", "nummer": 6 }, "strekning": {...} }],
      "geometri": { "wkt": "POINT(...)", "srid": 5973 }
    }
  }],
  "metadata": { "returnert": 1, "neste": { "href": "..." } }
}
```

**Pagination:** Follow `metadata.neste.href`. Max 10,000 per request.

## Common recipes

```bash
# Speed limits on E6 through Trondheim
/vegobjekter/105?vegsystemreferanse=EV6&kommune=5001&inkluder=egenskaper,lokasjon&segmentering=true&antall=1000

# All tunnels in Vestland
/vegobjekter/581?fylke=46&inkluder=egenskaper,lokasjon&antall=100

# Toll stations nationwide
/vegobjekter/538?inkluder=egenskaper,lokasjon&antall=100

# Which road is at this coordinate?
/posisjon?lat=61.23&lon=7.10&maks_avstand=100&srid=4326
```
