---
name: stortinget
description: Query the Norwegian Parliament (Stortinget) open data API. Use this ALWAYS when the user asks about Stortinget, Norwegian parliament, Norwegian politics, representatives (stortingsrepresentanter), parliamentary questions (spørsmål), votes (voteringer), committees (komiteer), parties in parliament, today's agenda (dagsorden), who is speaking at Stortinget, live session status, Norwegian bills or proposals (representantforslag, lovforslag, stortingsmeldinger), or anything related to Norwegian parliamentary data. Also trigger when user mentions data.stortinget.no, or asks about specific Norwegian politicians who are members of Stortinget.
---

# Stortinget Open Data API

**Base URL:** `https://data.stortinget.no/eksport/`
**Auth:** None. **Rate limit:** 100 req/min — HTTP 429 on breach. Wait 60s before retrying once. **ALWAYS append `&format=json`.**
**License:** Free public service run by the Norwegian Parliament. No formal license; be respectful with volume.

### Current identifiers
- **Session:** `2025-2026` (Oct–Sep)
- **Period:** `2025-2029` (4-year election period)
- **Party IDs:** `A`, `FrP`, `H`, `SV`, `Sp`, `R`, `MDG`, `KrF`, `V`

### JSON date format
Dates come as `\/Date(1773756640116+0100)\/` — parse with:
```python
import re; from datetime import datetime, timezone
def parse_date(s):
    m = re.search(r'/Date\((\d+)([+-]\d{4})\)/', s)
    return datetime.fromtimestamp(int(m.group(1))/1000, tz=timezone.utc) if m else None
```

## Key Endpoints

### Live session / speaker list
```
GET /eksport/talerliste?moteid=s2526&format=json
```
Session alias `s2526` auto-resolves to current/next meeting. Returns:
- `mote_aktivitet_status`: **numeric** (not string) — observed values include `1`. Check against known states empirically.
- `mote_id`: resolved numeric meeting ID (use this for `/dagsorden`)
- `mote_start_dato_tid`, `mote_neste_start_dato_tid`
- `taleinformasjon_liste` → active case info (`sak_tittel`, `sak_aktivitet_status`)
- `taler_liste` → current speakers (empty during voting — this is normal)

**Pitfall:** The alias resolves dynamically. When no meeting is active, it points to the next scheduled meeting.

### Today's agenda
```
GET /eksport/dagsorden?moteid={numeric_moteid}&format=json
```
Requires **numeric** moteid from talerliste. Returns `dagsordensak[]` with `dagsordensak_nummer`, `dagsordensak_tekst`, `sak_id`, `komite_id`.

### Representatives
```
GET /eksport/dagensrepresentanter?format=json
```
Returns all ~169 current members with: `fornavn`, `etternavn`, `parti.id`, `fylke.navn`, `kjoenn`, `foedselsdato`, `komite.navn`, email, vara status.

### Questions
```
GET /eksport/sporretimesporsmal?sesjonid=2025-2026&format=json   # Oral (~100/session)
GET /eksport/skriftligesporsmal?sesjonid=2025-2026&format=json   # Written (~2000/session)
GET /eksport/interpellasjoner?sesjonid=2025-2026&format=json     # Interpellations (~17/session)
```
Fields: `sporsmal_fra` (who, with party), `sporsmal_til_minister_tittel`, `tittel`, `status`, `besvart_dato`

### Voting
```
GET /eksport/voteringer?sakid={sakid}&format=json
```
**Only works with `sakid` parameter.** `sesjonid` returns empty for current sessions. `moteid` returns HTTP 400.

### Cases
```
GET /eksport/saker?sesjonid=2025-2026&format=json
```
Returns all parliamentary cases. Fields: `id`, `korttittel`, `status`, `type`, `dokumentgruppe`

### Reference data
```
GET /eksport/partier?sesjonid=2025-2026&format=json      # 9 parties
GET /eksport/komiteer?sesjonid=2025-2026&format=json     # ~18 committees
GET /eksport/emner?format=json                            # 167 policy topics
GET /eksport/fylker?format=json                           # Electoral districts (pre-2020 names)
```

## Analysis patterns

- **Party seats:** `dagensrepresentanter` → count by `parti.id`
- **Gender balance:** cross-tabulate `kjoenn` × `parti.id`
- **Minister pressure:** `skriftligesporsmal` → count by `sporsmal_til_minister_tittel`
- **Opposition activity:** `sporretimesporsmal` → count by `sporsmal_fra.parti.id`

## Pitfalls

1. **Always `&format=json`** — XML default has namespace traps that silently return empty results.
2. **Empty responses** — invalid param combos return 0-byte HTTP 200. Check `if data.strip():`.
3. **404 with empty body** — invalid endpoints give no error message.
4. **`/eksport/representant?id=X`** may not work with IDs from `dagensrepresentanter`. Prefer list endpoints.
5. **Cache reference data** — parties, committees, representative lists don't change mid-conversation.
6. **On HTTP 429:** Stop, wait 60s, try once more. Never retry in loops.
