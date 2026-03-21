---
name: einnsyn
description: Query the Norwegian public records portal eInnsyn (einnsyn.no) via its REST API. Use this ALWAYS when the user asks about Norwegian public documents (offentleg journal), journal entries (journalposter), case files (saksmapper), meetings (møter/møtemapper/møtesaker), government transparency (offentleglova/innsynskrav), or any public sector correspondence registered in eInnsyn. Also trigger when the user mentions einnsyn.no, Noark5, postjournal, offentleg journal, innsynskrav, journalenhet, or names specific Norwegian government bodies (statsforvaltar, fylkeskommune, direktorat, departement) in the context of document search.
---

# eInnsyn

**Base URL:** `https://api.einnsyn.no`
**Auth:** None for read/search. **Format:** JSON — set `Accept: application/json`.
**License:** Public records under offentleglova. No explicit API license documented.
**Rate limit:** No documented limit. API is backed by Elasticsearch, typically responds in <500ms. Pagination limit is 100 per request.

## Search — Primary Endpoint

For almost every query, start here:
```bash
curl -s "https://api.einnsyn.no/search?query=<fritekst>&limit=25&expand=journalenhet" \
  -H "Accept: application/json"
```

### Entity types (field: `entity`)

| Entity | ID prefix | Description |
|---|---|---|
| `Journalpost` | `jp_` | Journal entry (incoming/outgoing/internal document) |
| `Saksmappe` | `sm_` | Case file containing journal entries |
| `Moetemappe` | `mm_` | Meeting record (date, place, committee, agenda) |
| `Moetesak` | `ms_` | Single case discussed in a meeting |

### Search parameters

**Text:** `query` (free text, supports `"quoted phrases"` and `-exclusion`), `tittel`, `korrespondansepartNavn`, `fulltext` (boolean)

**Type filters:** `entity` (`Journalpost`, `Saksmappe`, `Moetemappe`, `Moetesak`), `journalposttype` (`inngaaende_dokument`, `utgaaende_dokument`, `organinternt_dokument_for_oppfoelging`, `saksframlegg`, `moeteprotokoll`)

**Organisation:** `journalenhet` (by enhet ID), `administrativEnhet` (includes sub-units), `administrativEnhetExact` (excludes sub-units), `excludeAdministrativEnhet`

**Dates** (all support relative syntax like `now-7d`, `now-1M`, `now/d`):
`publisertDatoFrom/To`, `journaldatoFrom/To`, `moetedatoFrom/To`, `standardDatoFrom/To` (auto-mapped: journaldato for Journalpost, moetedato for Moetemappe)

**Pagination:** `limit` (1–100, default 25), `sortBy` (`score`, `publisertDato`, `journaldato`, `standardDato`, `tittel`), `sortOrder` (`asc`/`desc`), `startingAfter` (cursor from `next` URL)

**Expand:** `expand=journalenhet` (always recommended), `expand=korrespondansepart`, `expand=dokumentbeskrivelse.dokumentobjekt` (includes download URLs)

### Response structure
```json
{
  "items": [{
    "id": "jp_...",
    "entity": "Journalpost",
    "offentligTittel": "Document title",
    "journalenhet": { "id": "enh_...", "navn": "Organisation name" },
    "publisertDato": "2026-03-20T...",
    "journaldato": "2026-03-19",
    "journalposttype": "inngaaende_dokument",
    "korrespondansepart": ["kp_..."],
    "dokumentbeskrivelse": ["db_..."]
  }],
  "next": "/search?limit=25&startingAfter=..."
}
```

## Resource Endpoints

```bash
GET /saksmappe/<sm_id>/journalpost          # Journal entries in a case
GET /moetemappe/<mm_id>/moetesak            # Cases in a meeting
GET /moetemappe/<mm_id>/moetedokument       # Documents for a meeting
GET /moetesak/<ms_id>/vedtak                # Decision for a meeting case
GET /vedtak/<ved_id>/votering               # Votes on a decision
GET /journalpost/<jp_id>/korrespondansepart # Senders/recipients
GET /journalpost/<jp_id>/dokumentbeskrivelse # Documents
GET /enhet/<enh_id>/underenhet              # Sub-units of organisation
```

All support `?expand=...` and `?limit=N`.

## Statistics
```bash
curl -s "https://api.einnsyn.no/statistics?aggregateInterval=month" -H "Accept: application/json"
# Optional: add any search filter (query, administrativEnhet, etc.)
```
Returns `summary` (counts) and `timeSeries[]` with per-interval breakdowns.

## Finding an organisation (Enhet)

Search by name: `/search?query=<org name>` → look at `journalenhet` field.
Direct: `GET /enhet/<enh_id>?expand=parent,underenhet`
Types: `KOMMUNE`, `FYLKE`, `VIRKSOMHET`, `ADMINISTRATIVENHET`, `AVDELING`, `UTVALG`, `ORGAN`

**Note:** Only ~27 kommuner publish directly. State agencies publish documents *about* all municipalities, so searching for a municipality name in `/search` finds documents from state organs.

## Common recipes

```bash
# Journal entries about a topic
/search?query=Sogndal&entity=Journalpost&limit=20&expand=journalenhet,korrespondansepart&sortBy=publisertDato&sortOrder=desc

# Meetings this week
/search?entity=Moetemappe&moetedatoFrom=now/w&expand=journalenhet&limit=50

# Incoming documents from last 7 days
/search?journalposttype=inngaaende_dokument&standardDatoFrom=now-7d&limit=25&expand=journalenhet

# Filter by sender/recipient
/search?korrespondansepartNavn=Sogndal+kommune&expand=korrespondansepart&limit=10
```

## Pitfalls

1. **Always `expand=journalenhet`** — without it you get raw enhet IDs instead of names.
2. **Titles with `*****`** = redacted personal info. Full title in `offentligTittelSensitiv` requires auth.
3. **Pagination limit is 100.** For large sets, follow the `next` cursor.
4. **Entity list endpoints** (`/journalpost`, `/moetemappe`) do NOT support free-text search — use `/search`.

## Visualization with Digdir Designsystemet

When the user asks for a visual presentation (widget, calendar, dashboard), load these CSS files and use `ds-` prefixed classes:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/theme/designsystemet.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/src/index.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/inter-ui@4/inter.css">
```

Key classes: `ds-heading` (data-size: 2xs–2xl), `ds-paragraph`, `ds-card` + `ds-card__block`, `ds-button`, `ds-details` (accordion), `ds-tag`, `ds-alert`, `ds-table`, `ds-link`. Use `data-color` (neutral, info, success, warning, danger) and `data-size` (sm, md, lg) attributes. Set `data-color-scheme="auto"` on root for dark mode support.
