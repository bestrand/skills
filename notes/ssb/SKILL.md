---
name: ssb
description: Query Statistics Norway (Statistisk sentralbyrå / SSB) via its open APIs. Use this skill ALWAYS when the user asks about Norwegian statistics, population data, economic indicators, employment, wages, GDP, inflation, housing prices, crime statistics, immigration, education statistics, birth rates, death rates, municipal statistics, or any quantitative data about Norway. Also trigger when user mentions SSB, Statistikkbanken, data.ssb.no, KLASS, or asks for demographic/economic data about Norwegian municipalities, counties, or the country as a whole. This covers the StatBank API (PxWebApi) for time series data and the KLASS API for classifications and code lists.
---

# SSB — Statistics Norway API

## StatBank API (PxWebApi)

**Base URL:** `https://data.ssb.no/api/v0/no/table/`
**Auth:** None. **Rate limit:** 30 req/60s. Max 800,000 cells per query.
**License:** CC BY 4.0 — credit "Statistisk sentralbyrå (SSB)" with link to ssb.no.

### Workflow: Always GET metadata first, then POST

**Step 1 — Get table metadata** to discover variable codes and valid values:
```bash
curl -s "https://data.ssb.no/api/v0/no/table/{TABLE_ID}"
```
Returns `variables[]` with `code`, `values[]`, and `valueTexts[]`.

**CRITICAL:** Variable codes and valid values differ between tables. Never assume a value exists — always check metadata first. For example, some tables use `"999"` for age totals, others have no total code at all.

**Step 2 — POST query:**
```bash
curl -s -X POST "https://data.ssb.no/api/v0/no/table/{TABLE_ID}" \
  -H "Content-Type: application/json" \
  -d '{"query":[...],"response":{"format":"json-stat2"}}'
```

### Query structure

Each variable needs a selection block:
```json
{
  "code": "VariableCode",
  "selection": { "filter": "item", "values": ["val1","val2"] }
}
```

**Filters:** `item` (exact values), `all` (`["*"]`), `top` (`["5"]` = last 5 time periods), `agg:KommSummer` (aggregate by classification).

**Response formats:** `json-stat2` (recommended), `csv`, `csv2`, `xlsx`

### JSON-stat2 response

```json
{
  "label": "Table title",
  "size": [1, 2, 5],
  "dimension": { "Region": { "category": { "index": {"0301":0}, "label": {"0301":"Oslo"} } } },
  "value": [123, 456, ...]
}
```
Values are a flat array. Use `size` to reshape into dimensions.

### Common tables

| ID | Description |
|---|---|
| `07459` | Population by municipality, sex, age, year |
| `08531` | Population changes (births, deaths, migration) |
| `09842` | Consumer Price Index (KPI) |
| `09171` | GDP quarterly national accounts |
| `06265` | Average monthly earnings |
| `07241` | Housing prices by region |
| `08921` | Crime — reported offences |

### Region codes

`0` = whole country (for tables that support it), 2-digit = county, 4-digit = municipality. Check KLASS for current codes.

### Main topic paths

`be` (population), `al` (labour/wages), `nr` (GDP), `pr` (prices/CPI), `bb` (housing), `sv` (crime), `ut` (education), `he` (health), `en` (energy), `va` (elections)

Browse: `GET https://data.ssb.no/api/v0/no/table/` returns all topics.

### Pitfalls

- **Error `"has an error"` on a variable** = you used an invalid value code. Re-check metadata.
- **Empty response** = 0-byte HTTP 200 from invalid combinations. Check `if data.strip():` before parsing.
- **Use `"filter":"top","values":["N"]`** for time dimension to get N most recent periods.
- **URL length limit:** ~2,100 chars for GET. Use wildcards (`*`) instead of listing all values.

## KLASS API — Classifications

**Base URL:** `https://data.ssb.no/api/klass/v1/`

```bash
# Current municipality codes
curl -s "https://data.ssb.no/api/klass/v1/classifications/131/codes?from=2024-01-01" \
  -H "Accept: application/json"

# Search classifications
curl -s "https://data.ssb.no/api/klass/v1/classifications/search?query=kommune" \
  -H "Accept: application/json"
```

**Key IDs:** 131 (municipalities), 104 (counties), 6 (NACE/SN2007), 36 (education), 2 (sex)

## Note on v2 API

SSB launched PxWebApi v2 at `https://data.ssb.no/api/pxwebapi/v2/` in autumn 2025. Supports GET queries with a more stable URL structure. The v0 API documented above still works.
