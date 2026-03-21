---
name: geonorge
description: Query Norwegian geographic data via Geonorge and Kartverket APIs. Use this skill ALWAYS when the user asks about Norwegian place names (stedsnavn), addresses (adresser), municipalities (kommuner), county info (fylker), coordinate lookups, geocoding, coordinate transformation, elevation (høyde), or any geographic/map data about Norway. Also trigger when user mentions Kartverket, Geonorge, ws.geonorge.no, or asks to look up a Norwegian address, find coordinates, or convert between coordinate systems (UTM, WGS84).
---

# Geonorge / Kartverket — Geographic APIs

**Auth:** None. **License:** CC BY 4.0 — credit "© Kartverket" and link to kartverket.no where possible.
**Rate limit:** No documented limit. Be reasonable with request frequency.

## 1. Stedsnavn (Place Names)

```bash
curl -s "https://ws.geonorge.no/stedsnavn/v1/navn?sok=Sogndal&fuzzy=true&treffPerSide=10"
```

**Parameters:** `sok` (search term), `fuzzy` (boolean), `treffPerSide` (results/page), `side` (page, 1-indexed), `navneobjekttype` (By, Tettsted, Bygd, Fjell, Fjord, Innsjø, Elv, Øy, Kommune), `fylkesnavn`, `kommunenavn`

**Response:** `metadata.totaltAntallTreff`, `navn[]` with `skrivemåte` (name string), `navneobjekttype`, `representasjonspunkt` (`nord`, `øst` — note: field names are in Norwegian), `kommuner[].kommunenavn/.kommunenummer`, `fylker[].fylkesnavn`

## 2. Adresser (Address Search)

```bash
curl -s "https://ws.geonorge.no/adresser/v1/sok?sok=Karl+Johans+gate+1+Oslo&treffPerSide=5"
```

**Parameters:** `sok` (free text), `adressenavn`, `nummer`, `postnummer`, `kommunenavn`, `kommunenummer`

**Response per address:** `adressetekst`, `adressenavn`, `nummer`, `postnummer`, `poststed`, `kommunenummer`, `kommunenavn`, `representasjonspunkt.lat/.lon`

**Reverse geocode:**
```bash
curl -s "https://ws.geonorge.no/adresser/v1/punktsok?lat=59.911&lon=10.749&radius=50&treffPerSide=5"
```

## 3. Kommuneinfo (Municipality)

```bash
# List all municipalities
curl -s "https://api.kartverket.no/kommuneinfo/v1/kommuner"

# Single municipality
curl -s "https://api.kartverket.no/kommuneinfo/v1/kommuner/4640"

# Coordinate → municipality
curl -s "https://api.kartverket.no/kommuneinfo/v1/punkt?nord=61.23&ost=7.10&koordsys=4258"

# All counties
curl -s "https://api.kartverket.no/kommuneinfo/v1/fylker"
```

Reverse lookup returns: `kommunenummer`, `kommunenavn`, `fylkesnummer`, `fylkesnavn`

## 4. Elevation

```bash
curl -s "https://ws.geonorge.no/hoydedata/v1/punkt?nord=59.91&ost=10.75&koordsys=4258"
```
Returns `{ "punkter": [{ "z": 42.5 }] }` — elevation in metres.

## 5. Coordinate Transformation

```bash
curl -s "https://ws.geonorge.no/transformering/v1/transformer?fra=4326&til=25833&x=10.75&y=59.91"
```

| EPSG | System |
|---|---|
| 4326 | WGS84 (GPS) |
| 4258 | EUREF89 (≈WGS84 for Norway) |
| 25832 | UTM zone 32N |
| 25833 | UTM zone 33N (most of Norway) |
| 25835 | UTM zone 35N (far north) |

## County codes (2024)

03 Oslo, 31 Østfold, 32 Akershus, 33 Buskerud, 34 Innlandet, 39 Vestfold, 40 Telemark, 42 Agder, 11 Rogaland, 46 Vestland, 15 Møre og Romsdal, 50 Trøndelag, 18 Nordland, 55 Troms, 56 Finnmark

## WFS (advanced)

`wfs.geonorge.no` hosts OGC WFS services returning GML/XML (not JSON). Useful for admin boundaries with geometry, road networks, protected areas. Best suited for GIS tools. Use the REST APIs above for typical queries.
