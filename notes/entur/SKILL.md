---
name: entur
description: Query Norwegian public transport data via the Entur APIs. Use this skill ALWAYS when the user asks about public transport in Norway, bus/train/tram/metro/ferry schedules, departures, journey planning, travel routes between Norwegian places, real-time departures (sanntid), stop places (holdeplasser), or mentions Entur, Ruter, Skyss, AtB, Kolumbus, Vy, SJ Nord, Flytoget, or any Norwegian transit operator. Also trigger when user asks how to get from A to B in Norway by public transport, or asks about next departures from a stop. Covers Journey Planner (GraphQL), Geocoder (stop/place search), and NSR (National Stop Register) lookups.
---

# Entur — Norwegian Public Transport API

## Overview

Entur aggregates all public transport data in Norway — buses, trains, trams, metro, ferries, and more — into a single API. The Journey Planner uses GraphQL.

**Base URL (Journey Planner):** `https://api.entur.io/journey-planner/v3/graphql`
**Base URL (Geocoder):** `https://api.entur.io/geocoder/v1/`
**Authentication:** None, but **must set** `ET-Client-Name` header.
**Format:** GraphQL (Journey Planner), JSON (Geocoder).

## Required Header

```
ET-Client-Name: your-company-yourapp
```

## 1. Geocoder — Find stops and places

### Autocomplete (search while typing)
```bash
curl -s "https://api.entur.io/geocoder/v1/autocomplete?text=Oslo+S&size=5" \
  -H "ET-Client-Name: skill-test"
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `text` | string | Search text |
| `size` | integer | Max results (default 10) |
| `lang` | string | Language (`no`, `en`) |
| `layers` | string | Filter: `venue` (stops), `address`, `street`, `locality` |
| `focus.point.lat` / `focus.point.lon` | float | Bias results towards location |

**Response:** GeoJSON FeatureCollection. Key fields per feature:
- `properties.id` — NSR ID (e.g. `NSR:StopPlace:58366`)
- `properties.name` — Stop/place name
- `properties.label` — Full label with context
- `properties.category` — Categories (e.g. `onstreetBus`, `railStation`, `metroStation`)
- `geometry.coordinates` — [lon, lat]

### Reverse geocode
```bash
curl -s "https://api.entur.io/geocoder/v1/reverse?point.lat=59.911&point.lon=10.750&size=5&layers=venue" \
  -H "ET-Client-Name: skill-test"
```

## 2. Journey Planner — Trip search (GraphQL)

### Trip from A to B
```graphql
{
  trip(
    from: { place: "NSR:StopPlace:58366" }
    to: { place: "NSR:StopPlace:59872" }
    numTripPatterns: 3
  ) {
    tripPatterns {
      duration
      walkDistance
      legs {
        mode
        fromPlace { name }
        toPlace { name }
        expectedStartTime
        expectedEndTime
        line {
          publicCode
          name
          transportMode
        }
        fromEstimatedCall {
          expectedDepartureTime
          destinationDisplay { frontText }
        }
      }
    }
  }
}
```

**Using coordinates instead of stop IDs:**
```graphql
{
  trip(
    from: { coordinates: { latitude: 59.911, longitude: 10.750 } }
    to: { coordinates: { latitude: 60.391, longitude: 5.322 } }
    numTripPatterns: 3
  ) {
    tripPatterns { ... }
  }
}
```

**Trip parameters:**
- `from` / `to` — `{ place: "NSR:..." }` or `{ coordinates: { latitude, longitude } }`
- `dateTime` — ISO datetime (default: now)
- `arriveBy` — boolean, if true `dateTime` is arrival time
- `numTripPatterns` — Max results (default 5)
- `modes` — Filter by transport mode
- `walkSpeed` — metres per second (default 1.3)

### Departures from a stop
```graphql
{
  stopPlace(id: "NSR:StopPlace:58366") {
    name
    estimatedCalls(numberOfDepartures: 10) {
      expectedDepartureTime
      actualDepartureTime
      realtime
      destinationDisplay { frontText }
      serviceJourney {
        line {
          publicCode
          name
          transportMode
        }
      }
      quay { publicCode name }
    }
  }
}
```

### List all operators/authorities
```graphql
{
  authorities {
    id
    name
    lines { publicCode name transportMode }
  }
}
```

### Stop place details
```graphql
{
  stopPlace(id: "NSR:StopPlace:58366") {
    name
    latitude
    longitude
    transportMode
    quays {
      id
      publicCode
      name
      lines { publicCode name }
    }
  }
}
```

## Making GraphQL Calls with curl

```bash
curl -s -X POST "https://api.entur.io/journey-planner/v3/graphql" \
  -H "Content-Type: application/json" \
  -H "ET-Client-Name: skill-test" \
  -d '{"query":"{ stopPlace(id: \"NSR:StopPlace:58366\") { name estimatedCalls(numberOfDepartures: 5) { expectedDepartureTime destinationDisplay { frontText } serviceJourney { line { publicCode transportMode } } } } }"}'
```

**Important:** In the JSON body, the GraphQL query must have escaped quotes for NSR IDs.

## Common NSR Stop IDs

| Stop | NSR ID | Type |
|---|---|---|
| Oslo S (jernbanestasjon) | NSR:StopPlace:59872 | railStation |
| Jernbanetorget (Oslo) | NSR:StopPlace:58366 | bus/tram/metro |
| Bergen stasjon | NSR:StopPlace:40090 | railStation |
| Trondheim S | NSR:StopPlace:41742 | railStation |
| Stavanger stasjon | NSR:StopPlace:43526 | railStation |
| Oslo Bussterminal | NSR:StopPlace:6507 | busStation |
| Oslo lufthavn (Gardermoen) | NSR:StopPlace:58211 | airport |
| Bergen lufthavn (Flesland) | NSR:StopPlace:60602 | airport |

**Tip:** Always use the Geocoder to find the correct NSR ID for a stop — don't guess.

## Transport Modes

`bus`, `tram`, `rail`, `metro`, `water` (ferry), `air`, `coach`, `cableway`, `funicular`

## Workflow: Journey Planning

1. **Geocode** origin and destination to get NSR IDs:
   ```
   GET /geocoder/v1/autocomplete?text=<origin>&layers=venue&size=3
   GET /geocoder/v1/autocomplete?text=<destination>&layers=venue&size=3
   ```
2. **Search trips** using the NSR IDs:
   ```graphql
   { trip(from: {place: "<origin_id>"} to: {place: "<dest_id>"} numTripPatterns: 3) { ... } }
   ```
3. **Present results** — show departure time, duration, lines, transfers.

## Workflow: Real-time Departures

1. **Find the stop** with Geocoder
2. **Query departures** with `estimatedCalls`
3. Show `expectedDepartureTime`, line number, destination, and whether `realtime` is true

## Notes

- **Real-time data** is available for most operators. Check `realtime: true` on calls.
- **Date/time format** is ISO 8601 with timezone (e.g. `2026-03-20T14:30:00+01:00`).
- **The API is rate-limited** — don't make more than a few requests per second.
- **GraphQL** means you only get the fields you ask for — keep queries lean.

## License, Attribution & Rate Limits

- **License:** Norwegian Licence for Open Government Data (NLOD). Free for any purpose including commercial use.
- **ET-Client-Name MANDATORY:** Every request MUST include `ET-Client-Name` header with format `<company>-<application>`. Requests without this header will face strict rate limiting and may be blocked entirely.
- **Rate limiting:** Entur deploys strict rate-limiting on unidentified consumers. No specific numeric limit is published, but always identify yourself. Do not make excessive requests.
- **Registration:** Not required for open APIs (Journey Planner, Geocoder, Stop Places). Privileged endpoints require OAuth2 authentication via client credentials.
- **Real-time data:** Real-time departure data is provided by transport operators and may have varying quality/availability across operators.
- **GraphQL:** Keep queries lean — request only the fields you need.
