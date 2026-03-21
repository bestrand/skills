---
name: vegbilder
description: Access Norwegian road imagery (Vegbilder) from Statens vegvesen. Use this skill ALWAYS when the user asks about road photos, street-level imagery of Norwegian roads, road condition images, visual inspection of roads, vegbilder, or wants to see what a specific road section looks like.
---

# Vegbilder — Norwegian Road Imagery

**WFS (metadata/search):** `https://ogckart-sn1.atlas.vegvesen.no/vegbilder_1_0/ows`
**S3 (images):** `https://s3vegbilder.atlas.vegvesen.no`
**Auth:** None. **License:** NLOD — free for any purpose.
**Attribution (required):** "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
**Rate limit:** No documented limit. WFS can be slow for large bboxes — keep count ≤2000.
**Image privacy:** Images may capture people/vehicles incidentally — handle with normal privacy considerations.

## Finding images — WFS query

**CRITICAL: Parameter casing matters.** Use this exact format:

```bash
curl -s "https://ogckart-sn1.atlas.vegvesen.no/vegbilder_1_0/ows?service=WFS&version=2.0.0&request=GetFeature&typenames=vegbilder_1_0:Vegbilder_{YEAR}&count=100&srsname=urn:ogc:def:crs:EPSG::4326&outputformat=application%2Fjson&bbox={minLat},{minLon},{maxLat},{maxLon},urn:ogc:def:crs:EPSG::4326"
```

**Parameters:**
- `typenames`: `vegbilder_1_0:Vegbilder_{YEAR}` — use 4-digit year (2025, 2024). **Not** `typeName`.
- `srsname`: Must be full URN `urn:ogc:def:crs:EPSG::4326`, not `EPSG:4326`
- `bbox`: `{minLat},{minLon},{maxLat},{maxLon},urn:ogc:def:crs:EPSG::4326` — CRS repeated in suffix
- `count`: Max 2000. Paginate with `startindex` (0-based).
- `outputformat`: `application%2Fjson` (URL-encoded)

### Choosing the year
Current year may not have images yet (photos taken spring–autumn). **Try current year first, fall back to previous.**

### Response (GeoJSON)

```json
{
  "features": [{
    "geometry": { "coordinates": [7.103, 61.228] },
    "properties": {
      "URL": "https://s3vegbilder.atlas.vegvesen.no/...",
      "VEGKATEGORI": "E",
      "VEGNUMMER": 55,
      "AAR": 2025,
      "TIDSPUNKT": "2025-06-15T10:23:45",
      "FELTKODE": "1",
      "VEGSYSTEMREFERANSE": "EV55S1D1m12345",
      "METER": 12345,
      "RETNING": "Med"
    }
  }]
}
```

**Coordinates are [longitude, latitude]** (GeoJSON standard).

## Downloading images

```bash
curl -s -o road_image.jpg "https://s3vegbilder.atlas.vegvesen.no/..."
```
Images are typically 2048×1536 JPEG from vehicle-mounted cameras. **URLs contain signed tokens that expire within hours** — download promptly, don't cache URLs.

## Known issues

1. **`FELTKODE`** is sometimes integer, sometimes string. Compare with `str()`.
2. **Start with wide bbox (±0.1°).** Too narrow often returns zero results.
3. **Max 2000 features.** Paginate: `&startindex=0`, `&startindex=2000`, etc.
4. **Current year may be empty.** Fall back to previous year.

## Workflows

**Images for a road section:** Get bbox coordinates → query WFS → filter by `VEGSYSTEMREFERANSE` and `METER` range → download URLs.

**Compare years:** Query same bbox with `Vegbilder_2023`, `Vegbilder_2024`, `Vegbilder_2025` → match by `METER` position.

**Visual assessment:** Query area → pre-screen metadata in code → download selected images. Don't try to view hundreds manually.
