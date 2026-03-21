---
name: kartkatalog
description: Search Norway's national spatial data catalogue (Geonorge Kartkatalog) for geographic datasets, map services, WMS/WFS endpoints, and metadata about Norwegian geospatial data. Use this skill when the user asks about finding Norwegian map data, spatial datasets, WMS services, WFS services, or wants to discover what geospatial data is available for a topic in Norway (e.g. flood maps, soil data, land use, cultural heritage, nature areas, elevation models, orthophotos).
---

# Geonorge Kartkatalog — Dataset Discovery API

**Base URL:** `https://kartkatalog.geonorge.no/api/`
**Auth:** None. **License:** CC BY 4.0 / NLOD for metadata; individual datasets have their own licenses (check `DataAccess` field).
**Attribution:** Credit Geonorge / the originating agency when using discovered data.
**Rate limit:** No documented limit. Be reasonable.

## Search

```bash
curl -s "https://kartkatalog.geonorge.no/api/search?text=flom&limit=10" -H "Accept: application/json"
```

**Parameters:** `text`, `limit` (default 10), `offset`, `facets[0].name`/`facets[0].value` (filter by facet)

**Facets:** `type` (dataset, service, series), `organization` (Kartverket, NVE, Miljødirektoratet, NGU, NIBIO), `theme` (Basis geodata, Geologi, Natur, Samferdsel, Kulturminner), `dataaccess` (Åpne data, Begrenset)

**Example — NVE flood datasets:**
```bash
curl -s "https://kartkatalog.geonorge.no/api/search?text=flom&facets%5B0%5D.name=organization&facets%5B0%5D.value=NVE&limit=10"
```

**Response:** `NumFound`, `Results[]` with `Uuid`, `Title`, `Abstract`, `Type`, `Organization`, `Theme`, `DistributionProtocols` (WMS, WFS, GeoJSON — may be empty), `ShowMapUrl` (direct map viewer link)

## Get full metadata

```bash
curl -s "https://kartkatalog.geonorge.no/api/getdata/{Uuid}" -H "Accept: application/json"
```

Returns distribution URLs, download links, WMS/WFS endpoints.

## Common search themes

`flom` (flood), `skred` (landslide/avalanche), `grunnforhold` (soil), `naturtype` (habitats), `kulturminne` (heritage), `arealdekke` (land cover), `høydedata` (elevation/LiDAR), `ortofoto` (aerial photos), `eiendom` (property/cadastre), `veg` (roads), `sjøkart` (nautical), `fiskeri` (fisheries)

**Note:** This API discovers datasets — use the distribution URLs from results to access actual data.
