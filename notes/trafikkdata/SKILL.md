---
name: trafikkdata
description: Query Norwegian traffic volume data via the Trafikkdata GraphQL API from Statens vegvesen. Use this skill ALWAYS when the user asks about traffic volumes (trafikkmengde), ÅDT (annual average daily traffic), traffic counts (trafikktelling), hourly traffic, daily traffic, seasonal traffic patterns, traffic registration points (tellepunkter), or traffic trends on Norwegian roads. Also trigger when user mentions trafikkdata, trafikkdata.no, ÅDT, døgntrafikk, timetrafikk, sykkeltrafikk, or asks how much traffic a Norwegian road carries.
---

# Trafikkdata — Norwegian Traffic Volume API

## Overview

Trafikkdata provides detailed traffic volume measurements from ~1,624 registration points across Norway's road network. It gives hourly, daily, and yearly volumes, average daily traffic (ÅDT), and multi-year trends. Managed by Statens vegvesen.

**Endpoint:** `https://trafikkdata-api.atlas.vegvesen.no/` (GraphQL, POST only)
**Authentication:** None required.
**Format:** JSON (GraphQL).

## Making Requests

All requests are POST with `Content-Type: application/json`:

```bash
curl -s -X POST "https://trafikkdata-api.atlas.vegvesen.no/" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ ... }"}'
```

## Top-Level Queries

| Query | Description |
|---|---|
| `trafficRegistrationPoints` | Find registration points (with optional filters) |
| `trafficData` | Get volume data for a specific point |
| `roadCategories` | List available road categories |
| `areas` | List available areas (counties, municipalities) |
| `trafficVolumeIndices` | Traffic volume index trends |
| `publishedAreaTrafficVolumeIndices` | Published area-level indices |

## 1. Find Traffic Registration Points

```graphql
{
  trafficRegistrationPoints(searchQuery: {
    roadCategoryIds: [E]
    countyNumbers: [46]
    isOperational: true
    trafficType: VEHICLE
  }) {
    id
    name
    location {
      coordinates { latLon { lat lon } }
    }
    trafficRegistrationType
    operationalStatus
    dataTimeSpan { firstData { volumeByDay } lastData { volumeByDay } }
  }
}
```

**Search filter fields (`searchQuery`):**

| Field | Type | Description |
|---|---|---|
| `query` | String | Free text search |
| `roadCategoryIds` | [RoadCategory] | `E` (europaveg), `R` (riksveg), `F` (fylkesveg), `K` (kommunal) |
| `countyNumbers` | [Int] | County numbers (e.g. `[46]` for Vestland) |
| `isOperational` | Boolean | Only active points |
| `trafficType` | Enum | `VEHICLE` or `BICYCLE` |
| `registrationFrequency` | Enum | `CONTINUOUS` or `PERIODIC` |

**Response fields per point:**
- `id` — Unique point ID (e.g. `52742V2282262`). Use this in `trafficData` queries.
- `name` — Location name
- `location.coordinates.latLon` — `{lat, lon}` in WGS84
- `trafficRegistrationType` — `VEHICLE` or `BICYCLE`
- `operationalStatus` — `OPERATIONAL`, `TEMPORARILY_OUT_OF_SERVICE`, `RETIRED`
- `dataTimeSpan` — First and last dates with data

**~1,624 points total** (mostly vehicle, some bicycle).

## 2. Get Traffic Volumes

### Daily volumes
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      byDay(from: "2026-03-10T00:00:00+01:00", to: "2026-03-15T00:00:00+01:00") {
        edges {
          node {
            from
            to
            total {
              volumeNumbers { volume }
              coverage { percentage }
            }
          }
        }
      }
    }
  }
}
```

### Hourly volumes
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      byHour(from: "2026-03-14T00:00:00+01:00", to: "2026-03-14T12:00:00+01:00") {
        edges {
          node {
            from
            to
            total {
              volumeNumbers { volume }
              coverage { percentage }
            }
          }
        }
      }
    }
  }
}
```

### Average daily traffic (ÅDT) by year
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      average {
        daily {
          byYear {
            year
            total {
              volume { average }
              coverage { percentage }
            }
          }
        }
      }
    }
  }
}
```

### Average by day of week
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      average {
        dayOfWeek {
          ...
        }
        hourOfDay {
          ...
        }
      }
    }
  }
}
```

## Volume Response Structure

The volume data uses Relay-style pagination with `edges` → `node`:

```json
{
  "edges": [
    {
      "node": {
        "from": "2026-03-14T00:00:00+01:00",
        "to": "2026-03-14T01:00:00+01:00",
        "total": {
          "volumeNumbers": { "volume": 101 },
          "coverage": { "percentage": 100.0 }
        }
      }
    }
  ]
}
```

**Key fields:**
- `from` / `to` — ISO datetime period
- `total.volumeNumbers.volume` — Vehicle/bicycle count for the period
- `total.coverage.percentage` — Data completeness (100% = full coverage, <100% = some data gaps)

## Date Format

All dates must be ISO 8601 with timezone: `2026-03-14T00:00:00+01:00`

CET (winter) = `+01:00`, CEST (summer) = `+02:00`

## Common Workflows

### Find ÅDT for a road
1. Find the registration point: search by `roadCategoryIds` and `countyNumbers` or `query`
2. Query `volume.average.daily.byYear` for multi-year ÅDT history

### Compare weekday vs weekend traffic
1. Query `volume.average.dayOfWeek` for the point
2. Group by Mon-Fri vs Sat-Sun

### Get hourly traffic pattern for a day
1. Query `byHour` with a 24-hour from/to range
2. Plot the hourly volumes

### Find busiest roads in a county
1. List all points in the county: `searchQuery: { countyNumbers: [46] }`
2. For each point, get latest year's ÅDT from `average.daily.byYear`
3. Sort by volume

## License, Attribution & Rate Limits

- **License:** NLOD (Norsk lisens for offentlige data). Free for any use including commercial.
- **Attribution:** Credit Statens vegvesen. Example: "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
- **Rate limiting:** No documented numeric limit. Be reasonable — avoid tight polling loops.
- **Data accuracy:** Coverage percentage indicates data completeness. Treat low-coverage periods with caution.
- **Data freshness:** Updated continuously. Recent days/hours should have data within a few hours of real time.
