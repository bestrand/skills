---
name: stortinget
description: Query the Norwegian Parliament (Stortinget) open data API. Use this ALWAYS when the user asks about Stortinget, Norwegian parliament, Norwegian politics, representatives (stortingsrepresentanter), parliamentary questions (spørsmål), votes (voteringer), committees (komiteer), parties in parliament, today's agenda (dagsorden), who is speaking at Stortinget, live session status, Norwegian bills or proposals (representantforslag, lovforslag, stortingsmeldinger), or anything related to Norwegian parliamentary data. Also trigger when user mentions data.stortinget.no, or asks about specific Norwegian politicians who are members of Stortinget. This covers live session monitoring, historical data, representative lookups, voting records, and question time data.
---

# Stortinget Open Data API

Base URL: `https://data.stortinget.no/eksport/`

## Rate limiting

**100 requests per minute.** This is a free, open public service run by the Norwegian Parliament — treat it with respect.

### Principles
- **Batch your work**: If you need data from 3 endpoints, that's fine — but plan your calls, don't speculatively hit endpoints "just to see."
- **Never poll in loops**: No `setInterval`, no `while True` with `sleep`. For "live companion" use, make ONE request per user-initiated check.
- **Cache within a conversation**: Representative lists, committee data, party lists, and other reference data don't change mid-conversation. Fetch once, reuse from memory.
- A typical conversation should use **5–15 requests total**, not 50+.

### Handling HTTP 429 (Too Many Requests)
If you receive a 429 response:
1. **Stop immediately.** Do not retry right away.
2. **Wait at least 60 seconds** before the next request. If a `Retry-After` header is present, respect that value instead.
3. **Tell the user** what happened — don't silently retry in a loop.
4. **Reduce scope** — ask the user which data they need most and fetch only that.
5. **Never implement automatic retry loops.** One polite retry after a proper wait is acceptable. Hammering through a 429 is not.

```python
import subprocess, time, json

def fetch_stortinget(endpoint):
    url = f"https://data.stortinget.no/eksport/{endpoint}&format=json" \
          if '?' in endpoint else \
          f"https://data.stortinget.no/eksport/{endpoint}?format=json"
    result = subprocess.run(['curl', '-s', '-w', '\\n%{http_code}', url],
                            capture_output=True, text=True)
    lines = result.stdout.rsplit('\n', 1)
    body, status = lines[0], int(lines[1])
    if status == 429:
        # Respect rate limit — inform user, wait, try once more
        time.sleep(60)
        result = subprocess.run(['curl', '-s', '-w', '\\n%{http_code}', url],
                                capture_output=True, text=True)
        lines = result.stdout.rsplit('\n', 1)
        body, status = lines[0], int(lines[1])
        if status == 429:
            raise Exception("Rate limited twice — stop and wait longer")
    if not body.strip():
        return None
    return json.loads(body)
```

## Response format

**ALWAYS use JSON.** Append `&format=json` to every endpoint URL.

The API defaults to XML with a namespace (`http://data.stortinget.no`) that silently returns empty results if you forget the prefix. JSON avoids this entirely and is simpler to parse.

```python
import json, subprocess
data = subprocess.run(
    ['curl', '-s', 'https://data.stortinget.no/eksport/ENDPOINT?param=value&format=json'],
    capture_output=True, text=True
).stdout
result = json.loads(data)
```

**JSON date pitfall**: Dates come as Microsoft .NET format `\/Date(1773756640116+0100)\/` — not ISO 8601. Parse with:
```python
import re
from datetime import datetime, timezone, timedelta
def parse_dotnet_date(s):
    match = re.search(r'/Date\((\d+)([+-]\d{4})\)/', s)
    if match:
        ms = int(match.group(1))
        return datetime.fromtimestamp(ms / 1000, tz=timezone.utc)
    return None
```

**Empty responses**: Invalid parameter combos return 0-byte HTTP 200 responses. Always check `if data.strip():` before parsing.

## Key identifiers

- **sesjonid**: `"2025-2026"` format (parliamentary session, Oct–Sep)
- **stortingsperiodeid**: `"2025-2029"` format (4-year election period)
- **moteid**: numeric meeting ID (e.g., `11599`) or session alias (see below)
- **sakid**: numeric case ID (e.g., `107358`)
- **representant id**: short string code (e.g., `"MARNIL"`)
- **Party ids**: `A`, `FrP`, `H`, `SV`, `Sp`, `R`, `MDG`, `KrF`, `V`
- **Committee ids**: `FINANS`, `UFO`, `KONTROLL`, `JUSTIS`, `ENERGI`, `HELSDOMS`, `ARBSOS`, `FAMKULT`, `NÆRING`, `TRANSKCOM`, `KOMMFORV`, `UFK`

### Finding the current session and period

```
GET /eksport/sesjoner
GET /eksport/stortingsperioder
```

As of 2025-2026: session is `2025-2026`, period is `2025-2029`.

---

## Endpoints reference

### 1. Live session monitoring

#### Speaker list / session status (the "who is talking now" endpoint)
```
GET /eksport/talerliste?moteid={moteid}
```

Use a **session alias** like `moteid=s2526` to auto-resolve to the current or next meeting. This is the most useful pattern for "what's happening now" queries.

**Key fields returned:**
- `mote_aktivitet_status`: one of `debatt`, `votering`, `ikke_startet`, possibly others
- `mote_id`: the resolved numeric meeting ID
- `mote_start_dato_tid`: meeting start time
- `mote_neste_start_dato_tid`: next meeting start (useful when current is over)
- `taleinformasjon_liste` → `taleinformasjon`: current case info
  - `sak_tittel`: title of case being debated
  - `sak_aktivitet_status`: `aktiv`, `ikke_startet`, etc.
- `taler_liste` → `taler`: speakers (empty during voting or when no debate is active)

**Pitfall**: The session alias `s2526` dynamically resolves. When no meeting is active, it points to the *next scheduled* meeting. The `mote_id` it returns will differ from the meeting that just ended.

**Pitfall**: `taler_liste` is empty during `votering` status — there's no speaker at the podium during votes. Don't report "nobody is speaking" as an error.

### 2. Today's agenda
```
GET /eksport/dagsorden?moteid={numeric_moteid}
```

Returns the ordered list of cases for a specific meeting. Requires a **numeric** moteid (not the session alias).

**How to get today's moteid**: First call `talerliste?moteid=s2526` to get the resolved `mote_id`, then use that numeric ID for the agenda.

**Key fields per `dagsordensak`:**
- `dagsordensak_nummer`: item number on agenda
- `dagsordensak_tekst`: description of the case
- `dagsordensak_type`: `Innst.`, `REFERAT`, etc.
- `dagsordensak_henvisning`: parliamentary reference (e.g., `Innst. 160 S (2025-2026)`)
- `sak_id`: links to the full case detail endpoint
- `komite_id`: which committee prepared this

### 3. Meetings list
```
GET /eksport/moter?sesjonid={sesjonid}
```

Returns all meetings in a session (typically ~150 per session). Includes past and future scheduled meetings.

### 4. Representatives

#### Current representatives (today's active members)
```
GET /eksport/dagensrepresentanter
```

Returns all ~169 current representatives with: name, party, county (fylke), gender, birth date, committee assignment, email, whether they are a substitute (vara).

#### Representatives by period
```
GET /eksport/representanter?stortingsperiodeid={periodeId}
```

#### Single representative detail
```
GET /eksport/representant?id={representantId}
```

**Pitfall**: This endpoint returned empty/404 in testing with the ID format from `dagensrepresentanter`. The `id` field from that endpoint (e.g., `MARNIL`) may not work here. Use data from the list endpoints instead.

### 5. Questions

#### Question time (spørretime) — oral questions
```
GET /eksport/sporretimesporsmal?sesjonid={sesjonid}
```

Returns all oral question-time questions for the session. Rich data including:
- `sporsmal_fra`: who asked (with nested party info)
- `sporsmal_til_minister_tittel`: which minister was questioned
- `tittel`: the question text
- `status`: `besvart`, `til_behandling`, etc.
- `besvart_dato`: when answered

Good for analyzing: which parties ask the most questions, which ministers are most questioned.

#### Written questions
```
GET /eksport/skriftligesporsmal?sesjonid={sesjonid}
```

Much higher volume (~1900 per session). Same structure as oral questions.

#### Interpellations
```
GET /eksport/interpellasjoner?sesjonid={sesjonid}
```

Formal debates initiated by a representative. Lower volume (~17 per session).

### 6. Cases (saker)

#### List cases
```
GET /eksport/saker?sesjonid={sesjonid}
```

Optional filter: `&sakstypeid=alminneligsak`

Returns all parliamentary cases: bills, proposals, reports, budgets, etc.

**Key fields:** `id`, `korttittel`, `status` (`til_behandling`, `behandlet`), `type`, `dokumentgruppe`

#### Single case detail
```
GET /eksport/sak?sakid={sakid}
```

Full detail including: title, reference, status, committee, document group, session.

### 7. Voting

#### Votes for a case
```
GET /eksport/voteringer?sakid={sakid}
```

**Pitfall**: `voteringer?sesjonid=2025-2026` returned 0 results in testing for the current session — voting data may only be available per-case or after processing. Try `sakid` parameter instead.

**Pitfall**: `voteringer?moteid=11599` returned HTTP 400 — `moteid` is NOT a valid parameter for this endpoint.

#### Vote proposals
```
GET /eksport/voteringsforslag?voteringid={voteringId}
```

### 8. Reference data

#### Parties
```
GET /eksport/partier?sesjonid={sesjonid}
```

#### Committees
```
GET /eksport/komiteer?sesjonid={sesjonid}
```

Returns all committees including special ones (e.g., the COVID committee, the EOS committee).

#### Topics/themes
```
GET /eksport/emner
```

167 policy topics used to tag cases (energy, taxes, education, etc.).

#### Counties
```
GET /eksport/fylker
```

Electoral districts. Note: uses old county names (pre-2020 reform).

#### Sessions and periods
```
GET /eksport/sesjoner
GET /eksport/stortingsperioder
```

Sessions go back to 1986-87.

---

## Common pitfalls summary

1. **Always append `&format=json`**: The API defaults to XML with a tricky namespace. JSON is simpler and avoids silent empty-result bugs.

2. **Empty responses**: Invalid parameter combos return 0-byte HTTP 200 responses. Always check `if data.strip():` before parsing.

3. **404 with empty body**: Invalid endpoints return HTTP 404 with a 0-byte body — no error message.

4. **JSON dates**: `\/Date(1773756640116+0100)\/` format, not ISO 8601. See parsing helper above.

5. **Session alias quirks**: `moteid=s2526` (and other `s` + digits patterns) auto-resolve to the current/next meeting. The resolved `mote_id` changes once a meeting ends. Different session codes (e.g., `s2526` vs `s2425`) may resolve to the same meeting.

6. **Voting data gaps**: `voteringer?sesjonid=X` may return 0 results for the current session. Use `sakid` parameter for specific cases, or check older sessions.

7. **Representative detail endpoint**: `/eksport/representant?id=X` may not work with all ID formats. Prefer the list endpoints (`dagensrepresentanter` or `representanter?stortingsperiodeid=X`).

8. **Norwegian characters**: Response text is UTF-8 with Norwegian special characters (ø, æ, å). Handle encoding properly.

9. **Null fields in JSON**: Optional fields appear as `null` in JSON. Always use `.get()` or null-checks.

10. **`voteringer` does not accept `moteid`**: This returns HTTP 400. Use `sakid` instead.

---

## Useful analysis patterns

### Party seat distribution
`dagensrepresentanter` → count by `parti/id`

### Gender balance
`dagensrepresentanter` → cross-tabulate `kjoenn` × `parti/id`

### Age demographics
`dagensrepresentanter` → parse `foedselsdato`, compute age

### Opposition activity
`sporretimesporsmal` → count by `sporsmal_fra/parti/id` — opposition parties dominate question counts

### Minister pressure
`sporretimesporsmal` or `skriftligesporsmal` → count by `sporsmal_til_minister_tittel`

### Committee workload
`dagensrepresentanter` → count by `komite/navn` for size; combine with `saker` filtered by committee for throughput

### Live session status
`talerliste?moteid=s2526` → check `mote_aktivitet_status` for current state, `taleinformasjon_liste` for active case

---

## Network requirements

The domain `data.stortinget.no` must be in the allowed domains list. Use `curl` or Python `urllib`/`requests` to fetch data. The API requires no authentication or API key.
