---
name: trafikkdata
description: Query Norwegian traffic volume data via the Trafikkdata GraphQL API from Statens vegvesen. Use this skill ALWAYS when the user asks about traffic volumes (trafikkmengde), ÅDT (annual average daily traffic), traffic counts (trafikktelling), hourly traffic, daily traffic, seasonal traffic patterns, traffic registration points (tellepunkter), or traffic trends on Norwegian roads. Also trigger when user mentions trafikkdata, trafikkdata.no, ÅDT, døgntrafikk, timetrafikk, sykkeltrafikk.
---

# Trafikkdata — Norwegian Traffic Volume API

**Endpoint:** `https://trafikkdata-api.atlas.vegvesen.no/` (GraphQL, POST only)
**Auth:** None. **License:** NLOD — free for any purpose.
**Attribution (required):** "Inneholder data under norsk lisens for offentlige data (NLOD) tilgjengeliggjort av Statens vegvesen."
**Rate limit:** No documented limit. Avoid tight polling loops.
**Dates:** ISO 8601 with timezone: `2026-03-14T00:00:00+01:00` (CET=+01:00, CEST=+02:00)

## Find registration points

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
    location { coordinates { latLon { lat lon } } }
    trafficRegistrationType
    dataTimeSpan { firstData { volumeByDay } lastData { volumeByDay } }
  }
}
```

**Search filters:** `query` (free text), `roadCategoryIds` ([E, R, F, K]), `countyNumbers` ([46]), `isOperational` (boolean), `trafficType` (VEHICLE or BICYCLE), `registrationFrequency` (CONTINUOUS, PERIODIC)

**~1,624 points total.** Response includes `id` (use in volume queries), `name`, lat/lon.

## Get traffic volumes

### Daily
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      byDay(from: "2026-03-10T00:00:00+01:00", to: "2026-03-15T00:00:00+01:00") {
        edges { node { from to total { volumeNumbers { volume } coverage { percentage } } } }
      }
    }
  }
}
```

### Hourly
Same structure, use `byHour` instead of `byDay`.

### ÅDT (average daily traffic by year)
```graphql
{
  trafficData(trafficRegistrationPointId: "52742V2282262") {
    volume {
      average { daily { byYear { year total { volume { average } coverage { percentage } } } } }
    }
  }
}
```

Also available: `average.dayOfWeek`, `average.hourOfDay` for traffic patterns.

## Response structure

Uses Relay pagination: `edges[]` → `node` with `from`, `to`, `total.volumeNumbers.volume` (count), `total.coverage.percentage` (data completeness — 100% = full, lower = gaps).

## Making requests

```bash
curl -s -X POST "https://trafikkdata-api.atlas.vegvesen.no/" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ trafficRegistrationPoints(searchQuery: { roadCategoryIds: [E], countyNumbers: [46], isOperational: true, trafficType: VEHICLE }) { id name } }"}'
```

## Workflows

**Find ÅDT for a road:** Search points by road category + county → query `average.daily.byYear` for each point.
**Weekday vs weekend:** Query `average.dayOfWeek` → group Mon–Fri vs Sat–Sun.
**Busiest roads in county:** List all points → get latest year's ÅDT → sort by volume.
