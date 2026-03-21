---
name: kartkatalog
description: Search Norway's national spatial data catalogue (Geonorge Kartkatalog) for geographic datasets, map services, WMS/WFS endpoints, and metadata about Norwegian geospatial data. Use this skill when the user asks about finding Norwegian map data, spatial datasets, WMS services, WFS services, or wants to discover what geospatial data is available for a topic in Norway (e.g. flood maps, soil data, land use, cultural heritage, nature areas, elevation models, orthophotos). Also trigger when user mentions kartkatalog, geonorge datasets, or asks about open geodata in Norway.
---

# Geonorge Kartkatalog — Dataset Discovery API

## Overview

The Kartkatalog is Norway's national catalogue for geographic datasets and map services. It indexes thousands of datasets from Kartverket, NVE, Miljødirektoratet, NGU, municipalities, and other agencies.

**Base URL:** `https://kartkatalog.geonorge.no/api/`
**Authentication:** None.
**Format:** JSON.

## Search

```bash
curl -s "https://kartkatalog.geonorge.no/api/search?text=flom&limit=10" \
  -H "Accept: application/json"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `text` | string | Free text search |
| `limit` | integer | Max results (default 10) |
| `offset` | integer | Pagination offset |
| `facets[0].name` | string | Facet filter name (e.g. `type`, `organization`, `theme`) |
| `facets[0].value` | string | Facet filter value |
| `mediatype` | string | Filter by distribution format |
| `orderby` | string | Sort field |

### Facet Filters

Common facets for narrowing results:

| Facet name | Example values |
|---|---|
| `type` | `dataset`, `service`, `software`, `series` |
| `organization` | `Kartverket`, `NVE`, `Miljødirektoratet`, `NGU` |
| `theme` | `Basis geodata`, `Geologi`, `Natur`, `Samferdsel`, `Kulturminner` |
| `nationaltheme` | `Energi`, `Forurensning`, `Kyst og fiskeri`, `Samfunnssikkerhet` |
| `dataaccess` | `Åpne data`, `Begrenset` |

**Example — find open flood datasets from NVE:**
```bash
curl -s "https://kartkatalog.geonorge.no/api/search?text=flom&facets%5B0%5D.name=organization&facets%5B0%5D.value=NVE&limit=10"
```

## Response Structure

```json
{
  "NumFound": 83,
  "Offset": 0,
  "Limit": 10,
  "Results": [
    {
      "Uuid": "d4dbce45-...",
      "Title": "Flomsonekart",
      "Abstract": "Kart som viser områder som kan oversvømmes...",
      "Type": "dataset",
      "Organization": "NVE",
      "Theme": "Natur",
      "DistributionProtocols": ["WMS", "WFS", "GeoJSON"],
      "ServiceDistributionUrlForDataset": "https://...",
      "ShowMapUrl": "https://kartkatalog.geonorge.no/...",
      "LegendDescriptionUrl": "...",
      "ProductSheetUrl": "...",
      "DatasetLanguage": "Norsk"
    }
  ],
  "Facets": [
    {
      "FacetField": "type",
      "FacetResults": [
        { "Name": "dataset", "Count": 72 },
        { "Name": "service", "Count": 11 }
      ]
    }
  ]
}
```

## Get Dataset Metadata

```bash
curl -s "https://kartkatalog.geonorge.no/api/getdata/d4dbce45-..." \
  -H "Accept: application/json"
```

Returns full metadata including distribution URLs (WMS, WFS, download links).

## Common Search Themes

| Search | Typical results |
|---|---|
| `flom` | Flood zone maps, flood risk (NVE) |
| `skred` | Landslide/avalanche hazard zones (NVE, NGU) |
| `grunnforhold` | Soil conditions, bedrock (NGU) |
| `naturtype` | Nature types, habitats (Miljødirektoratet) |
| `kulturminne` | Cultural heritage sites (Riksantikvaren) |
| `arealdekke` | Land cover / land use (NIBIO) |
| `høydedata` | Elevation models, LiDAR (Kartverket) |
| `ortofoto` | Aerial photos (Kartverket) |
| `eiendom` | Property/cadastre (Kartverket) |
| `veg` | Road data (Statens vegvesen) |
| `sjøkart` | Nautical charts (Kartverket) |
| `fiskeri` | Fisheries data (Fiskeridirektoratet) |

## Notes

1. This API is for **discovering** datasets, not for querying the data itself. Use the distribution URLs (WMS/WFS/API) from the results to access actual data.
2. Many datasets are available as WMS (view) and WFS (download features) services.
3. The `ShowMapUrl` field provides a direct link to view the dataset on Geonorge's map viewer.

## License, Attribution & Rate Limits

- **License:** The catalogue metadata itself is available under CC BY 4.0 / NLOD. Individual datasets listed in the catalogue have their own licenses (check `DataAccess` field).
- **Attribution:** Credit Geonorge / the originating agency when using data discovered through the catalogue.
- **Rate limiting:** No documented limit. Be reasonable with search requests.
- **Registration:** Not required.
