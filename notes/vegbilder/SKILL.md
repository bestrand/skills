---
name: vegbilder
description: Access Norwegian road imagery (Vegbilder) from Statens vegvesen. Use this skill ALWAYS when the user asks about road photos, street-level imagery of Norwegian roads, road condition images, visual inspection of roads, vegbilder, or wants to see what a specific road section looks like. Also trigger when user mentions vegbilder, road images, street view of Norwegian roads, or needs visual ground truth for road features. Covers the WFS service for finding image metadata and the S3 service for downloading actual images.
---

# Vegbilder — Norwegian Road Imagery

## Overview

Vegbilder is Statens vegvesen's collection of road imagery — systematic photographs taken along the Norwegian road network, typically several times per year. Useful for visual inspection, condition assessment, and verifying NVDB data.

**Two services:**
1. **WFS** (metadata/search): `https://ogckart-sn1.atlas.vegvesen.no/vegbilder_1_0/ows`
2. **S3** (actual images): `https://s3vegbilder.atlas.vegvesen.no`

**Authentication:** None required.
**Format:** GeoJSON (WFS), JPEG (S3).

## 1. Finding Images — WFS Query

**IMPORTANT:** This WFS has non-standard parameter casing. Use this exact format:

```bash
curl -s "https://ogckart-sn1.atlas.vegvesen.no/vegbilder_1_0/ows?service=WFS&version=2.0.0&request=GetFeature&typenames=vegbilder_1_0:Vegbilder_{YEAR}&count=100&srsname=urn:ogc:def:crs:EPSG::4326&outputformat=application%2Fjson&bbox={minLat},{minLon},{maxLat},{maxLon},urn:ogc:def:crs:EPSG::4326"
```

### Parameter requirements (case-sensitive!)

| Parameter | Value | Notes |
|---|---|---|
| `service` | `WFS` | |
| `version` | `2.0.0` | |
| `request` | `GetFeature` | |
| `typenames` | `vegbilder_1_0:Vegbilder_{YEAR}` | **NOT `typeName`**. Use 4-digit year (e.g. `2025`, `2024`). |
| `count` | integer | Max features to return (max 2000) |
| `srsname` | `urn:ogc:def:crs:EPSG::4326` | **Must be full URN format**, not `EPSG:4326` |
| `outputformat` | `application%2Fjson` | URL-encoded. Returns GeoJSON. |
| `bbox` | `{minLat},{minLon},{maxLat},{maxLon},urn:ogc:def:crs:EPSG::4326` | **CRS must be repeated in bbox suffix** |
| `startindex` | integer | For pagination (0-based) |

### Choosing the year

Current year may not have images yet. **Try current year first, fall back to previous year.**

### Example — Find images near Sogndal

```bash
curl -s "https://ogckart-sn1.atlas.vegvesen.no/vegbilder_1_0/ows?service=WFS&version=2.0.0&request=GetFeature&typenames=vegbilder_1_0:Vegbilder_2025&count=50&srsname=urn:ogc:def:crs:EPSG::4326&outputformat=application%2Fjson&bbox=61.15,7.00,61.30,7.20,urn:ogc:def:crs:EPSG::4326"
```

### Response (GeoJSON)

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [7.103, 61.228]
      },
      "properties": {
        "URL": "https://s3vegbilder.atlas.vegvesen.no/...",
        "VEGKATEGORI": "E",
        "VEGNUMMER": 55,
        "AAR": 2025,
        "TIDSPUNKT": "2025-06-15T10:23:45",
        "FELTKODE": "1",
        "VEGSYSTEMREFERANSE": "EV55S1D1m12345",
        "STREKNING": 1,
        "DELSTREKNING": 1,
        "METER": 12345,
        "RETNING": "Med"
      }
    }
  ]
}
```

**Key properties:**
- `URL` — Direct link to the image on S3. **URLs are signed and expire within hours — download promptly.**
- `VEGKATEGORI` — Road category (E, R, F, K)
- `VEGNUMMER` — Road number
- `VEGSYSTEMREFERANSE` — Full road reference string
- `AAR` — Year the image was taken
- `TIDSPUNKT` — Timestamp
- `FELTKODE` — Lane code (1 = lane 1, 2 = lane 2, etc.)
- `RETNING` — Direction ("Med" = with metering direction, "Mot" = against)
- `METER` — Metre position along the road section

**GeoJSON coordinates are `[longitude, latitude]`** (not lat, lon).

## 2. Downloading Images — S3

The `URL` field in the WFS response points directly to the JPEG image on S3:

```bash
curl -s -o road_image.jpg "https://s3vegbilder.atlas.vegvesen.no/..."
```

Images are typically 2048×1536 or similar resolution, taken from a vehicle-mounted camera.

## Known Issues and Workarounds

1. **FELTKODE type inconsistency:** The `FELTKODE` field is sometimes returned as an integer, sometimes as a string. Always compare with `str()` when filtering.

2. **Max 2000 features per request.** Paginate with `startindex`:
   ```
   &startindex=0     (first 2000)
   &startindex=2000  (next 2000)
   ```

3. **Start with a wide bbox (±0.1 degrees).** Too narrow a bbox often returns zero results. Expand gradually if needed.

4. **Image URLs expire.** They contain a signature that's only valid for a few hours. Don't cache URLs — re-query the WFS if you need fresh URLs.

5. **Current year may have no data.** Images are collected during driving season (typically spring-autumn). Try the previous year as fallback.

## Common Workflows

### Get images for a specific road section
1. Know the road reference (e.g. `EV55S1D1`) — get this from NVDB's `/posisjon` endpoint
2. Query WFS with a bbox around the area
3. Filter results by `VEGSYSTEMREFERANSE` and `METER` range
4. Download images from the `URL` field

### Visual condition assessment
1. Find the road segment's bbox coordinates
2. Query WFS for recent-year images
3. Download a selection of images along the segment
4. When many images are returned, pre-screen metadata in code before opening with vision — don't try to view hundreds of images manually

### Compare road changes over years
Query the same bbox for different years (`Vegbilder_2023`, `Vegbilder_2024`, `Vegbilder_2025`) and compare images at the same metre position.

## License, Attribution & Rate Limits

- **License:** NLOD (Norsk lisens for offentlige data). Free for any use including commercial.
- **Attribution:** Credit Statens vegvesen. "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
- **Rate limiting:** No documented numeric limit. The WFS can be slow for large bboxes — keep count reasonable (≤2000).
- **Image rights:** Images are part of the NLOD-licensed dataset. They may contain incidental captures of people/vehicles — handle with normal privacy considerations.
