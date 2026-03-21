---
name: ssb
description: Query Statistics Norway (Statistisk sentralbyrå / SSB) via its open APIs. Use this skill ALWAYS when the user asks about Norwegian statistics, population data, economic indicators, employment, wages, GDP, inflation, housing prices, crime statistics, immigration, education statistics, birth rates, death rates, municipal statistics, or any quantitative data about Norway. Also trigger when user mentions SSB, Statistikkbanken, data.ssb.no, KLASS, or asks for demographic/economic data about Norwegian municipalities, counties, or the country as a whole. This covers the StatBank API (PxWebApi) for time series data and the KLASS API for classifications and code lists.
---

# SSB — Statistics Norway API

## Overview

SSB provides two main APIs:
1. **StatBank API (PxWebApi)** — Access to all tables in Statistikkbanken with population, economy, labour market, etc.
2. **KLASS API** — Classifications and code lists (municipality codes, NACE codes, education codes, etc.)

Both are free, open, and require no authentication.

## StatBank API (PxWebApi)

**Base URL:** `https://data.ssb.no/api/v0/no/table/`

### Step 1: Browse available topics

```bash
curl -s "https://data.ssb.no/api/v0/no/table/" | python3 -c "
import json, sys
for topic in json.load(sys.stdin):
    print(f'{topic[\"id\"]} | {topic[\"text\"]}')
"
```

**Main topic codes:**
| Code | Topic |
|---|---|
| `al` | Arbeid og lønn (Labour & wages) |
| `bf` | Bank og finansmarked (Finance) |
| `vf` | Bedrifter, foretak og regnskap (Business) |
| `be` | Befolkning (Population) |
| `bb` | Bygg, bolig og eiendom (Construction & housing) |
| `en` | Energi og industri (Energy & industry) |
| `he` | Helse (Health) |
| `if` | Inntekt og forbruk (Income & consumption) |
| `kf` | Kultur og fritid (Culture & recreation) |
| `nr` | Nasjonalregnskap (National accounts / GDP) |
| `nt` | Natur og miljø (Nature & environment) |
| `of` | Offentlig sektor (Public sector) |
| `pr` | Priser og prisindekser (Prices & CPI) |
| `sv` | Sosiale forhold og kriminalitet (Social & crime) |
| `tr` | Transport og reiseliv (Transport & tourism) |
| `ut` | Utdanning (Education) |
| `ur` | Utenriksøkonomi (Foreign economy) |
| `va` | Valg (Elections) |
| `vt` | Virksomheter, foretak og regnskap (Enterprises) |

### Step 2: Get table metadata

```bash
curl -s "https://data.ssb.no/api/v0/no/table/{TABLE_ID}"
```

Returns variables (dimensions) with their codes, value lists, and value texts. **You must inspect metadata first** to know which values are valid for the POST query.

**Key population table: `07459`** — Population by region, sex, age, year.

### Step 3: Query data (POST)

```bash
curl -s -X POST "https://data.ssb.no/api/v0/no/table/{TABLE_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": [
      {
        "code": "Region",
        "selection": { "filter": "item", "values": ["0"] }
      },
      {
        "code": "Kjonn",
        "selection": { "filter": "item", "values": ["1", "2"] }
      },
      {
        "code": "Alder",
        "selection": { "filter": "item", "values": ["999"] }
      },
      {
        "code": "ContentsCode",
        "selection": { "filter": "item", "values": ["Personer1"] }
      },
      {
        "code": "Tid",
        "selection": { "filter": "top", "values": ["5"] }
      }
    ],
    "response": { "format": "json-stat2" }
  }'
```

**Selection filters:**
| Filter | Description | Example |
|---|---|---|
| `item` | Exact values | `"values": ["0301", "4640"]` |
| `all` | All values | `"values": ["*"]` |
| `top` | Last N values | `"values": ["5"]` (last 5 time periods) |
| `agg:KommSummer` | Aggregate by classification | Requires KLASS classification ID |
| `vs:Kommuner2024` | Value set / codelist | Pre-defined groupings |

**Response format options:** `json-stat2` (recommended), `json-stat`, `csv`, `csv2`, `csv3`, `xlsx`

### JSON-stat2 response structure

```json
{
  "label": "07459: Befolkning, etter region, kjønn, alder, statistikkvariabel og år",
  "size": [1, 2, 1, 1, 5],
  "dimension": {
    "Region": { "category": { "index": {"0": 0}, "label": {"0": "Hele landet"} } },
    "Kjonn": { "category": { "index": {"1": 0, "2": 1}, "label": {"1": "Menn", "2": "Kvinner"} } },
    ...
  },
  "value": [2780000, 2847400, ...]
}
```

Values are a flat array. Use the `size` array to reshape into dimensions.

### Common tables

| Table ID | Description |
|---|---|
| `07459` | Population by municipality, sex, age, year |
| `08531` | Population changes (births, deaths, migration) |
| `07984` | Employment by industry |
| `11418` | Unemployment by county |
| `09842` | Consumer Price Index (KPI) |
| `09171` | GDP quarterly national accounts |
| `06265` | Average monthly earnings |
| `07241` | Housing prices by region |
| `08921` | Crime — reported offences |
| `12880` | Immigration by country of origin |
| `12202` | Completed education by municipality |
| `09817` | Election results (Storting) |
| `06913` | Municipal finances (KOSTRA) |

### Tips

- **Always GET metadata first** before POSTing a query. Variable codes and valid values differ between tables.
- **Use `"filter": "top", "values": ["N"]`** for time dimension to get the N most recent periods.
- **Region codes:** `0` = whole country, 2-digit = county (fylke), 4-digit = municipality. Check KLASS for current codes.
- **The `Alder` variable** uses 3-digit codes: `000` = 0 years, `001` = 1 year, ..., `099` = 99 years, `999` = total all ages. NOT all tables use `999` — check metadata.

## KLASS API — Classifications

**Base URL:** `https://data.ssb.no/api/klass/v1/`

### List classifications
```bash
curl -s "https://data.ssb.no/api/klass/v1/classifications?size=20" \
  -H "Accept: application/json"
```

### Get classification codes
```bash
# Classification 131 = municipalities (kommuner)
curl -s "https://data.ssb.no/api/klass/v1/classifications/131/codes?from=2024-01-01" \
  -H "Accept: application/json"
```

### Search classifications
```bash
curl -s "https://data.ssb.no/api/klass/v1/classifications/search?query=kommune" \
  -H "Accept: application/json"
```

### Useful classification IDs

| ID | Name |
|---|---|
| 2 | Standard for kjønn (Sex) |
| 6 | Standard for næringsgruppering (NACE/SN2007) |
| 36 | Standard for utdanningsgruppering (Education) |
| 131 | Standard for kommuneinndeling (Municipalities) |
| 104 | Standard for fylkesinndeling (Counties) |

## Common Recipes

### Get total population of Norway last 5 years
```bash
curl -s -X POST "https://data.ssb.no/api/v0/no/table/07459" \
  -H "Content-Type: application/json" \
  -d '{"query":[{"code":"Region","selection":{"filter":"item","values":["0"]}},{"code":"Kjonn","selection":{"filter":"item","values":["1","2"]}},{"code":"Alder","selection":{"filter":"all","values":["*"]}},{"code":"ContentsCode","selection":{"filter":"item","values":["Personer1"]}},{"code":"Tid","selection":{"filter":"top","values":["5"]}}],"response":{"format":"json-stat2"}}'
```

### Get CPI (consumer price index)
```bash
# First check metadata
curl -s "https://data.ssb.no/api/v0/no/table/09842"
# Then query
curl -s -X POST "https://data.ssb.no/api/v0/no/table/09842" \
  -H "Content-Type: application/json" \
  -d '{"query":[{"code":"Konsumgrp","selection":{"filter":"item","values":["TOTAL"]}},{"code":"ContentsCode","selection":{"filter":"item","values":["KpiIndMnd"]}},{"code":"Tid","selection":{"filter":"top","values":["12"]}}],"response":{"format":"json-stat2"}}'
```

### Get municipality list (current)
```bash
curl -s "https://data.ssb.no/api/klass/v1/classifications/131/codes?from=2024-01-01" \
  -H "Accept: application/json"
```

## License, Attribution & Rate Limits

- **License:** Creative Commons Attribution 4.0 International (CC BY 4.0). Free for any purpose including commercial use.
- **Attribution required:** You must credit "Statistisk sentralbyrå (SSB)" or "Statistics Norway (SSB)" with a link to ssb.no when using the data.
- **Rate limiting:** **30 requests per 60 seconds.** Exceeding this returns HTTP 429. Run large queries in sequence — wait for each response before sending the next.
- **Cell limit:** Maximum **800,000 cells** per query (including empty cells). The web Statistikkbanken has a lower limit of 300,000 cells. Exceeding returns HTTP 403.
- **URL length limit:** GET URLs cannot exceed ~2,100 characters. Use wildcards (`*`) instead of listing all values.
- **Registration:** Not required.
- **Data changes:** Tables may be structurally changed, replaced, or discontinued. Monitor SSB's change log.
- **New API version (v2):** SSB launched PxWebApi v2 in autumn 2025 at `https://data.ssb.no/api/pxwebapi/v2/`. It supports GET queries and has a more stable URL structure. The v0 API documented above still works.
