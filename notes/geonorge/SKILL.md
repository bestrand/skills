---
name: geonorge
description: Query Norwegian geographic data via Geonorge and Kartverket APIs. Use this skill ALWAYS when the user asks about Norwegian place names (stedsnavn), addresses (adresser), municipalities (kommuner), county info (fylker), coordinate lookups, geocoding, coordinate transformation, elevation (høyde), or any geographic/map data about Norway. Also trigger when user mentions Kartverket, Geonorge, ws.geonorge.no, matrikkel, grunnbok, or asks to look up a Norwegian address, find coordinates for a Norwegian place, or convert between coordinate systems (UTM, WGS84, EUREF89). Covers Stedsnavn (place names), Adresser (address search), Kommuneinfo (municipality data), and coordinate transformation.
---

# Geonorge / Kartverket — Geographic APIs

## Overview

Geonorge provides several REST APIs for Norwegian geographic data. All are free and require no authentication.

## 1. Stedsnavn (Place Names)

Search for named places in Norway — cities, mountains, lakes, farms, etc.

**Base URL:** `https://ws.geonorge.no/stedsnavn/v1/`

### Search
```bash
curl -s "https://ws.geonorge.no/stedsnavn/v1/navn?sok=Sogndal&fuzzy=true&treffPerSide=10"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `sok` | string | Search term |
| `fuzzy` | boolean | Enable fuzzy matching (default false) |
| `treffPerSide` | integer | Results per page (default 10) |
| `side` | integer | Page number (1-indexed) |
| `navneobjekttype` | string | Filter by type (see below) |
| `fylkesnavn` | string | Filter by county name |
| `kommunenavn` | string | Filter by municipality name |

**Common place types (`navneobjekttype`):**
`By`, `Tettsted`, `Bygd`, `Grend`, `Fjell`, `Fjord`, `Innsjø`, `Elv`, `Dal`, `Øy`, `Vik`, `Kommune`, `Fylke`, `Bruk`, `Gard`

**Response:**
```json
{
  "metadata": { "totaltAntallTreff": 164, "side": 1, "treffPerSide": 10 },
  "navn": [
    {
      "skrivemåte": "Sogndal",
      "navneobjekttype": "Tettsted",
      "kommuner": [{ "kommunenummer": "4640", "kommunenavn": "Sogndal" }],
      "fylker": [{ "fylkesnummer": "46", "fylkesnavn": "Vestland" }],
      "representasjonspunkt": { "øst": 7.1, "nord": 61.23, "koordsysKode": 4258 }
    }
  ]
}
```

## 2. Adresser (Address Search)

Search and geocode Norwegian addresses.

**Base URL:** `https://ws.geonorge.no/adresser/v1/`

### Search
```bash
curl -s "https://ws.geonorge.no/adresser/v1/sok?sok=Karl+Johans+gate+1+Oslo&treffPerSide=5"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `sok` | string | Free text search |
| `adressenavn` | string | Street name |
| `nummer` | integer | House number |
| `bokstav` | string | House letter (A, B, etc.) |
| `postnummer` | string | Postal code |
| `poststed` | string | Postal place |
| `kommunenummer` | string | 4-digit municipality code |
| `kommunenavn` | string | Municipality name |
| `treffPerSide` | integer | Results per page |
| `side` | integer | Page number |

### Reverse geocode (coordinates → address)
```bash
curl -s "https://ws.geonorge.no/adresser/v1/punktsok?lat=59.911&lon=10.749&radius=50&treffPerSide=5"
```

| Parameter | Type | Description |
|---|---|---|
| `lat` | float | Latitude (WGS84) |
| `lon` | float | Longitude (WGS84) |
| `radius` | integer | Search radius in metres |
| `koordsys` | integer | Coordinate system (default 4258 = EUREF89/WGS84) |

**Response per address:**
```json
{
  "adressetekst": "Karl Johans gate 1",
  "adressenavn": "Karl Johans gate",
  "nummer": 1,
  "postnummer": "0154",
  "poststed": "OSLO",
  "kommunenummer": "0301",
  "kommunenavn": "Oslo",
  "representasjonspunkt": { "lat": 59.911, "lon": 10.749 }
}
```

## 3. Kommuneinfo (Municipality Information)

Municipality and county reference data.

**Base URL:** `https://api.kartverket.no/kommuneinfo/v1/`

### List all municipalities
```bash
curl -s "https://api.kartverket.no/kommuneinfo/v1/kommuner"
```

### Get single municipality
```bash
curl -s "https://api.kartverket.no/kommuneinfo/v1/kommuner/4640"
```

Returns bounding box, county info, centre point coordinates, Sami administration status.

### Reverse lookup (coordinates → municipality)
```bash
curl -s "https://api.kartverket.no/kommuneinfo/v1/punkt?nord=61.23&ost=7.10&koordsys=4258"
```

| Parameter | Type | Description |
|---|---|---|
| `nord` | float | Latitude / northing |
| `ost` | float | Longitude / easting |
| `koordsys` | integer | Coordinate system (4258=WGS84/EUREF89, 25833=UTM33N) |

### List all counties
```bash
curl -s "https://api.kartverket.no/kommuneinfo/v1/fylker"
```

## 4. Coordinate Transformation

Convert between coordinate systems.

```bash
curl -s "https://ws.geonorge.no/transformering/v1/transformer?fra=4326&til=25833&x=10.75&y=59.91"
```

| Parameter | Type | Description |
|---|---|---|
| `fra` | integer | Source EPSG code |
| `til` | integer | Target EPSG code |
| `x` | float | X coordinate (longitude in WGS84) |
| `y` | float | Y coordinate (latitude in WGS84) |

**Common EPSG codes:**
| Code | System |
|---|---|
| 4326 | WGS84 (GPS) |
| 4258 | EUREF89 (≈WGS84 for Norway) |
| 25832 | UTM zone 32N |
| 25833 | UTM zone 33N (covers most of Norway) |
| 25835 | UTM zone 35N (far north) |
| 5972 | EUREF89 / NTM zone 12 |

## 5. Elevation (Høyde)

Get terrain elevation for a point.

```bash
curl -s "https://ws.geonorge.no/hoydedata/v1/punkt?nord=59.91&ost=10.75&koordsys=4258"
```

## Norwegian County Codes (2024)

| Code | County |
|---|---|
| 03 | Oslo |
| 31 | Østfold |
| 32 | Akershus |
| 33 | Buskerud |
| 34 | Innlandet |
| 39 | Vestfold |
| 40 | Telemark |
| 42 | Agder |
| 11 | Rogaland |
| 46 | Vestland |
| 15 | Møre og Romsdal |
| 50 | Trøndelag |
| 18 | Nordland |
| 55 | Troms |
| 56 | Finnmark |

## Common Recipes

### Find coordinates for a place
```bash
curl -s "https://ws.geonorge.no/stedsnavn/v1/navn?sok=Galdhøpiggen&treffPerSide=1"
```

### Geocode a Norwegian address
```bash
curl -s "https://ws.geonorge.no/adresser/v1/sok?sok=Storgata+1+Bergen&treffPerSide=1"
```

### Which municipality is this coordinate in?
```bash
curl -s "https://api.kartverket.no/kommuneinfo/v1/punkt?nord=61.23&ost=7.10&koordsys=4258"
```

### Convert GPS (WGS84) to UTM33N
```bash
curl -s "https://ws.geonorge.no/transformering/v1/transformer?fra=4326&til=25833&x=10.75&y=59.91"
```

## License, Attribution & Rate Limits

- **License:** Kartverket's free products are licensed under Creative Commons Attribution 4.0 International (CC BY 4.0).
- **Attribution required:** You must display "© Kartverket" in all contexts where the data is used (applications, web solutions, printed products, etc.) and link to kartverket.no where possible.
- **Registration:** Not required for the APIs listed here.
- **Rate limiting:** No documented numeric limits for the Stedsnavn, Adresser, or Kommuneinfo APIs, but some Kartverket APIs have technical limitations documented per-service. Be reasonable with request frequency.
- **WMS/WFS services:** Some map services show data from the Geovekst partnership at detailed zoom levels — Kartverket may not hold all rights to this data. External data sources in some APIs carry their own licenses.
- **Coordinate systems:** Data is provided in EUREF89/WGS84 by default. Use the transformer API if you need other projections.
- **Data accuracy:** Maps may not always match terrain. Use responsibly.

## 6. WFS Services (wfs.geonorge.no)

Geonorge hosts OGC Web Feature Services for downloading geographic features. These return GML/XML — **JSON output is NOT supported**.

**Example — Get municipalities as GML:**
```bash
curl -s "https://wfs.geonorge.no/skwms1/wfs.administrative_enheter?service=WFS&version=2.0.0&request=GetFeature&typenames=app:Kommune&count=5"
```

**Available WFS services include:**
- `wfs.administrative_enheter` — municipalities, counties, borders
- `wfs.stedsnavn` — place names with geometry
- `wfs.elveg2` — road network (from Kartverket, not NVDB)
- `wfs.naturvernomrader` — protected nature areas

**Usage notes:**
- Output is always GML/XML. Parse with XML libraries, not JSON.
- For JSON-based geographic queries, prefer the Stedsnavn and Adresser REST APIs above.
- WFS is best suited for GIS tools (QGIS, ArcGIS) or when you need actual geometries (polygons, lines).
- `openwms.statkart.no` is deprecated/DNS-dead. Use `wfs.geonorge.no` for features and Kartverket's cache services for map tiles.
