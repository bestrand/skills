---
name: einnsyn
description: Query the Norwegian public records portal eInnsyn (einnsyn.no) via its REST API. Use this ALWAYS when the user asks about Norwegian public documents (offentleg journal), journal entries (journalposter), case files (saksmapper), meetings (møter/møtemapper/møtesaker), government transparency (offentleglova/innsynskrav), or any public sector correspondence registered in eInnsyn. Also trigger when the user mentions einnsyn.no, Noark5, postjournal, offentleg journal, innsynskrav, journalenhet, or names specific Norwegian government bodies (statsforvaltar, fylkeskommune, direktorat, departement) in the context of document search. Trigger for any request to find, search, list, or look up documents, meetings, decisions (vedtak), or voting records (voteringar) from Norwegian public sector organisations. This covers search, filtering, statistics, entity lookup, and document retrieval from api.einnsyn.no.
---

# eInnsyn

## Overview

eInnsyn is Norway's national transparency portal for public sector documents. It provides public access to journal entries (journalposter), case files (saksmapper), meeting records (møtemapper), meeting cases (møtesaker), and associated documents from government agencies, municipalities, and county administrations.

**Base URL:** `https://api.einnsyn.no`
**Authentication:** Not required for read/search operations. API key (`X-EIN-API-KEY: secret_...`) needed for write operations only.
**Format:** All responses are JSON. Set `Accept: application/json`.

## Quick Start — The Most Important Endpoint

For almost every user request, start with the **search endpoint**:

```
GET /search?query=<fritekst>&limit=25&expand=journalenhet
```

This is the primary entry point. It searches across all entity types (Journalpost, Saksmappe, Moetemappe, Moetesak) using Elasticsearch and supports rich filtering. No authentication needed.

**Example:**
```bash
curl -s "https://api.einnsyn.no/search?query=Sogndal&limit=10&expand=journalenhet" \
  -H "Accept: application/json"
```

## Entity Types

eInnsyn is built on Norway's Noark 5 archiving standard. The core searchable entities are:

| Entity | ID prefix | Description |
|---|---|---|
| **Journalpost** | `jp_` | A journal entry (incoming, outgoing, or internal document) |
| **Saksmappe** | `sm_` | A case file containing one or more journal entries |
| **Moetemappe** | `mm_` | A meeting record (date, place, committee, agenda) |
| **Moetesak** | `ms_` | A single case discussed in a meeting |

Supporting entities (not directly searchable, but expandable/fetchable):

| Entity | ID prefix | Description |
|---|---|---|
| **Enhet** | `enh_` | An organisational unit (kommune, fylke, direktorat, etc.) |
| **Korrespondansepart** | `kp_` | A sender or recipient on a journal entry |
| **Dokumentbeskrivelse** | `db_` | Document metadata (title, type, attachments) |
| **Dokumentobjekt** | `do_` | The actual file/document (URL reference) |
| **Vedtak** | `ved_` | A decision made in a meeting case |
| **Votering** | `vot_` | A vote cast by a meeting participant |
| **Utredning** | `utr_` | An investigation/report related to a meeting case |
| **Moetedokument** | `md_` | A document associated with a meeting |
| **Moetedeltaker** | `mdl_` | A meeting participant |
| **Skjerming** | `skj_` | Access restriction info (legal exemption basis) |

## Search Endpoint — Full Reference

### `GET /search`

**Query text parameters:**

| Parameter | Type | Description |
|---|---|---|
| `query` | string | Free text search. Supports quoted phrases (`"Sogndal kommune"`), exclusion with minus (`Sogndal -ungdomssenter`). |
| `tittel` | string[] | Filter by title (free text match) |
| `korrespondansepartNavn` | string[] | Filter by sender/recipient name |
| `skjermingshjemmel` | string[] | Filter by legal exemption basis |
| `fulltext` | boolean | If `true`, only return documents that have full-text content available |

**Entity/type filters:**

| Parameter | Type | Values |
|---|---|---|
| `entity` | string[] | `Journalpost`, `Saksmappe`, `Moetemappe`, `Moetesak` |
| `journalposttype` | string[] | `inngaaende_dokument`, `utgaaende_dokument`, `organinternt_dokument_for_oppfoelging`, `organinternt_dokument_uten_oppfoelging`, `saksframlegg`, `sakskart`, `moeteprotokoll`, `moetebok`, `ukjent` |

**Organisation filters:**

| Parameter | Type | Description |
|---|---|---|
| `journalenhet` | eInnsynId | Filter by the publishing organisation (enhet ID) |
| `administrativEnhet` | eInnsynId[] | Filter by enhet ID, **including sub-units** |
| `administrativEnhetExact` | eInnsynId[] | Filter by enhet ID, **excluding sub-units** |
| `excludeAdministrativEnhet` | eInnsynId[] | Exclude enhet and its sub-units |
| `excludeAdministrativEnhetExact` | eInnsynId[] | Exclude specific enhet only |

**Date filters (all support relative dates — see below):**

| Parameter | Description |
|---|---|
| `publisertDatoFrom` / `publisertDatoTo` | When the record was published to eInnsyn |
| `oppdatertDatoFrom` / `oppdatertDatoTo` | When the record was last updated |
| `journaldatoFrom` / `journaldatoTo` | The journal date (for Journalpost) |
| `dokumentetsDatoFrom` / `dokumentetsDatoTo` | The document date |
| `moetedatoFrom` / `moetedatoTo` | The meeting date (for Moetemappe) |
| `standardDatoFrom` / `standardDatoTo` | Auto-mapped: journaldato for Journalpost, moetedato for Moetemappe |

**Relative date syntax (Grafana/Elasticsearch style):**
- `now` — current time
- `now-7d` — 7 days ago
- `now-1M` — 1 month ago
- `now/d` — start of today
- `now-1d/d` — start of yesterday
- Units: `ms`, `s`, `m`, `h`, `d`, `w`, `M`, `y`

**Numeric filters:**

| Parameter | Type | Description |
|---|---|---|
| `saksaar` | string[] | Case year |
| `sakssekvensnummer` | string[] | Case sequence number |
| `saksnummer` | string[] | Full case number (e.g. "2025/1234") |
| `journalpostnummer` | string[] | Journal entry number |
| `journalsekvensnummer` | string[] | Journal sequence number |
| `moetesaksaar` | string[] | Meeting case year |
| `moetesakssekvensnummer` | string[] | Meeting case sequence number |

**Pagination and sorting:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer (1–100) | 25 | Number of results |
| `sortOrder` | `asc` / `desc` | `desc` | Sort direction |
| `sortBy` | string | `score` | Sort field (see below) |
| `startingAfter` | string[] | — | Pagination cursor (2-element array: sort value + ID) |
| `endingBefore` | string[] | — | Reverse pagination cursor |
| `expand` | string[] | — | Expand nested object references inline |

**sortBy values:** `score`, `administrativEnhetNavn`, `dokumentetsDato`, `entity`, `fulltekst`, `id`, `journaldato`, `journalpostnummer`, `journalposttype`, `korrespondansepartNavn`, `moetedato`, `oppdatertDato`, `publisertDato`, `standardDato`, `sakssekvensnummer`, `tittel`

**IDs filter (overrides other filters):**

| Parameter | Type | Description |
|---|---|---|
| `ids` | eInnsynId[] | Return specific records by their eInnsyn IDs |
| `externalIds` | string[] | Return specific records by their legacy external IDs |

## Expand System

All object references in responses are IDs by default. Use `expand` to inline the full objects.

```
?expand=journalenhet                          # Expand one level
?expand=journalenhet,korrespondansepart       # Expand multiple fields
?expand=journalpost.korrespondansepart        # Expand nested (on parent endpoints)
```

Common useful expansions for search:
- `expand=journalenhet` — shows the organisation name instead of just an enhet ID
- `expand=korrespondansepart` — shows sender/recipient names
- `expand=dokumentbeskrivelse` — shows document metadata
- `expand=dokumentbeskrivelse.dokumentobjekt` — includes download URLs

## Resource Endpoints (CRUD)

Every entity has standard endpoints. For read-only (no auth needed):

### List
```
GET /<entity>?limit=25&sortOrder=desc&expand=...
```
Lists the most recently created records. Supports `ListParameters` (limit, sortOrder, startingAfter, endingBefore, journalenhet, ids, externalIds, expand).

**NOTE:** These list endpoints do NOT support free-text search. They return records in creation order. For searching, always use `/search`.

### Get by ID
```
GET /<entity>/<id>?expand=...
```

### Sub-resource listing
```
GET /saksmappe/<sm_id>/journalpost         # Journalposter in a case
GET /moetemappe/<mm_id>/moetesak           # Cases in a meeting
GET /moetemappe/<mm_id>/moetedokument      # Documents for a meeting
GET /moetesak/<ms_id>/dokumentbeskrivelse  # Documents for a meeting case
GET /moetesak/<ms_id>/vedtak               # Decision for a meeting case
GET /moetesak/<ms_id>/utredning            # Investigation for a meeting case
GET /vedtak/<ved_id>/votering              # Votes on a decision
GET /vedtak/<ved_id>/vedtaksdokument       # Decision documents
GET /journalpost/<jp_id>/korrespondansepart  # Senders/recipients
GET /journalpost/<jp_id>/dokumentbeskrivelse # Documents
GET /enhet/<enh_id>/underenhet             # Sub-units
GET /enhet/<enh_id>/arkiv                  # Archives owned by unit
```

## Statistics Endpoint

### `GET /statistics`

Returns aggregated statistics. Accepts all `FilterParameters` (same as /search) plus:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `aggregateFrom` | date | 1 year ago | Start date |
| `aggregateTo` | date | today | End date |
| `aggregateInterval` | string | `hour` | `hour`, `day`, `week`, `month`, `year` |

**Response structure:**
```json
{
  "summary": {
    "createdCount": 591855,
    "createdWithFulltextCount": 50053,
    "createdInnsynskravCount": 532223,
    "downloadCount": 0
  },
  "metadata": {
    "aggregateInterval": "month",
    "aggregateFrom": "2025-03-18",
    "aggregateTo": "2026-03-17"
  },
  "timeSeries": [
    { "time": "2025-03-01T00:00:00.000Z", "createdCount": 2389, ... }
  ]
}
```

Can be combined with search filters:
```
GET /statistics?query=Sogndal&aggregateInterval=month
```

## Pagination

All list responses have this structure:
```json
{
  "items": [...],
  "next": "/search?limit=25&startingAfter=..."
}
```

To paginate, follow the `next` URL. For the `/search` endpoint, `startingAfter` is a 2-element array (sort value + record ID). For resource list endpoints, it's a single record ID.

## Enhet (Organisation) Lookup

Organisations are central. To find an enhet ID for filtering:

1. **Search by name:** Use `/search?query=<org name>` and look at the `journalenhet` field
2. **Direct lookup:** `GET /enhet/<enh_id>?expand=parent,underenhet`
3. **List all:** Paginate through `GET /enhet?limit=100` (cursor-based)

**Enhet types:** `KOMMUNE`, `FYLKE`, `VIRKSOMHET`, `ADMINISTRATIVENHET`, `AVDELING`, `BYDEL`, `SEKSJON`, `UTVALG`, `ORGAN`, `DUMMYENHET`

**Important:** Not all Norwegian municipalities are registered in eInnsyn. As of March 2026, only ~27 kommuner/fylkeskommuner publish directly. However, many state agencies (statsforvaltar, direktorat, departement) publish documents *about* all municipalities. So searching for a municipality name in `/search` will find documents from state organs even if the municipality itself doesn't publish.

## Common Recipes

### Find journal entries about a topic
```bash
curl -s "https://api.einnsyn.no/search?query=Sogndal&entity=Journalpost&limit=20&expand=journalenhet,korrespondansepart&sortBy=publisertDato&sortOrder=desc"
```

### Find meetings tomorrow
```bash
curl -s "https://api.einnsyn.no/search?entity=Moetemappe&moetedatoFrom=2026-03-18&moetedatoTo=2026-03-19&expand=journalenhet&limit=50"
```
Or use the list endpoint and filter client-side:
```bash
curl -s "https://api.einnsyn.no/moetemappe?limit=100&expand=journalenhet"
```

### Find incoming documents from last week
```bash
curl -s "https://api.einnsyn.no/search?journalposttype=inngaaende_dokument&standardDatoFrom=now-7d&limit=25&expand=journalenhet"
```

### Get full meeting details with agenda
```bash
# 1. Get the meeting
curl -s "https://api.einnsyn.no/moetemappe/mm_01kky1vrbpebds5tv4ep7fgfbs?expand=journalenhet,moetesak"
# 2. For each moetesak, get details
curl -s "https://api.einnsyn.no/moetesak/ms_01kky1vrdeeb8sx1zpe2etvqdr?expand=utredning,vedtak"
```

### Get voting records for a decision
```bash
curl -s "https://api.einnsyn.no/vedtak/<ved_id>/votering?expand=moetedeltaker,representerer"
```

### Exact phrase search
```bash
curl -s "https://api.einnsyn.no/search?query=%22Sogndal+kommune%22&limit=10"
```

### Search with exclusion
```bash
curl -s "https://api.einnsyn.no/search?query=Sogndal+-ungdomssenter&limit=10"
```

### Filter by sender/recipient
```bash
curl -s "https://api.einnsyn.no/search?korrespondansepartNavn=Sogndal+kommune&expand=korrespondansepart&limit=10"
```

### Statistics for an organisation
```bash
curl -s "https://api.einnsyn.no/statistics?administrativEnhet=enh_01j73r5z2ef4k8mca0cfd0zf7z&aggregateInterval=month"
```

## Implementation Notes

1. **Always use `/search` for user queries.** The entity list endpoints (`/journalpost`, `/moetemappe`, etc.) do NOT support free-text search — they only list records with basic filtering (journalenhet, IDs, pagination).

2. **The `search` parameter on list endpoints is ignored without auth.** Earlier API versions may have supported `?search=`, but the current API ignores unrecognised query parameters silently.

3. **Use `expand=journalenhet` by default** on search results to show meaningful organisation names instead of raw IDs.

4. **Enhet IDs don't change.** Once you find an enhet ID (e.g. `enh_01j73r5z2ef4k8mca0cfd0zf7z` = Statsforvaltaren i Vestland), you can cache and reuse it.

5. **Sensitive titles are redacted.** Fields marked with `*****` indicate redacted personal information. The full title is in `offentligTittelSensitiv` but requires authentication to view.

6. **Pagination limit is 100.** For large result sets, follow the `next` cursor.

7. **Response times are fast.** The API is backed by Elasticsearch and typically responds in <500ms.

## API Spec Source

The full API specification is available at:
- TypeSpec source: https://github.com/felleslosninger/einnsyn-api-spec/tree/main/typespec
- OpenAPI YAML: https://github.com/felleslosninger/einnsyn-api-spec/blob/main/openapi/einnsyn.openapi.yml
- TypeScript SDK: https://github.com/felleslosninger/einnsyn-sdk-typescript (`npm install https://github.com/felleslosninger/einnsyn-sdk-typescript`)
- Java SDK: https://github.com/felleslosninger/einnsyn-sdk-java

---

## Visualization with Digdir Designsystemet

**When to use:** Only when the user explicitly asks for a visualization, widget, calendar, dashboard, or visual presentation of eInnsyn data. If the user just asks a question, respond with plain text. If they ask to "show", "visualize", "make a calendar", "lag ein fin visning" etc., use Digdir Designsystemet for styling.

### Loading the design system

In Visualizer widgets (inline HTML) or React artifacts, load the CSS from CDN. These two files are required:

```html
<!-- Theme (design tokens / CSS variables) -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/theme/designsystemet.css">

<!-- Component styles -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/src/index.css">

<!-- Inter font (recommended) -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/inter-ui@4/inter.css">
```

Then set the font on your root element:

```html
<div style="font-family: 'Inter', sans-serif; font-feature-settings: 'cv05' 1;">
  <!-- content here -->
</div>
```

### Core CSS classes (pure HTML, no React needed)

Designsystemet uses CSS class names prefixed with `ds-` and `data-` attributes for configuration. Here are the most useful components:

**Typography:**
- `<h2 class="ds-heading" data-size="md">Title</h2>` — sizes: `2xs`, `xs`, `sm`, `md`, `lg`, `xl`, `2xl`
- `<p class="ds-paragraph" data-size="md">Text</p>` — sizes: `sm`, `md`, `lg`
- `<a class="ds-link" href="...">Link</a>`

**Card:**
```html
<div class="ds-card" data-color="neutral">
  <div class="ds-card__block">
    <h3 class="ds-heading" data-size="xs">Title</h3>
    <p class="ds-paragraph" data-size="sm">Description</p>
  </div>
</div>
```

**Button:**
```html
<button class="ds-button" data-variant="secondary" data-size="sm">Click me</button>
```
Variants: `primary`, `secondary`, `tertiary`. Sizes: `sm`, `md`, `lg`.

**Details / Accordion:**
```html
<details class="ds-details">
  <summary>Summary text</summary>
  <div>Content when expanded</div>
</details>
```

**Badge / Tag:**
```html
<span class="ds-tag" data-color="info" data-size="sm">Label</span>
```

**Alert:**
```html
<div class="ds-alert" data-color="info">
  <p class="ds-paragraph">Alert message</p>
</div>
```

**Table:**
```html
<table class="ds-table" data-size="sm">
  <thead><tr><th>Heading</th></tr></thead>
  <tbody><tr><td>Data</td></tr></tbody>
</table>
```

**Divider:**
```html
<hr class="ds-divider">
```

### Data attributes

- `data-color` — Sets component color. Values: `neutral`, `brand1`, `brand2`, `brand3`, `accent`, `info`, `success`, `warning`, `danger`
- `data-size` — Sets component size. Values vary by component but typically: `sm`, `md`, `lg` (typography also supports `xs`, `2xs`, `xl`, `2xl`)
- `data-variant` — Component variant. E.g. `tinted` on cards for lighter background, `primary`/`secondary`/`tertiary` on buttons
- `data-color-scheme` — Color mode: `light`, `dark`, `auto`

### CSS variables available from the theme

The theme provides CSS custom properties you can use for custom styling:

- `--ds-color-accent-*` — Accent color scale (base, text, surface, border variants)
- `--ds-color-neutral-*` — Neutral color scale
- `--ds-color-brand1-*` / `brand2-*` / `brand3-*` — Brand colors
- `--ds-color-info-*` / `success-*` / `warning-*` / `danger-*` — Semantic colors
- `--ds-font-family` — The configured font family
- `--ds-spacing-*` — Spacing scale (0–13)
- `--ds-border-radius-*` — Border radius tokens
- `--ds-shadow-*` — Shadow tokens

### Example: Meeting calendar widget

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/theme/designsystemet.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@digdir/designsystemet-css@1/dist/src/index.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/inter-ui@4/inter.css">

<div style="font-family: 'Inter', sans-serif; font-feature-settings: 'cv05' 1;">
  <h2 class="ds-heading" data-size="sm">Møte i dag</h2>
  <p class="ds-paragraph" data-size="sm" style="margin-bottom: 1rem;">Frå eInnsyn — offentlege møte 20. mars 2026</p>

  <div class="ds-card" data-color="neutral" style="margin-bottom: 0.75rem;">
    <div class="ds-card__block">
      <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 4px;">
        <span class="ds-tag" data-color="info" data-size="sm">09:00</span>
        <h3 class="ds-heading" data-size="2xs">Partssammensatt utvalg</h3>
      </div>
      <p class="ds-paragraph" data-size="sm">Buskerud fylkeskommune — Fylkeshuset Drammen</p>
    </div>
    <div class="ds-card__block">
      <details class="ds-details">
        <summary>7 saker på agendaen</summary>
        <div>
          <ul style="margin: 0.5rem 0; padding-left: 1.25rem;">
            <li><p class="ds-paragraph" data-size="sm">Evaluering av Fylkesbiblioteket</p></li>
            <li><p class="ds-paragraph" data-size="sm">Vurdering av tannlegevaktordningen</p></li>
          </ul>
          <a class="ds-link" href="https://einnsyn.no/..." target="_blank">Sjå på eInnsyn</a>
        </div>
      </details>
    </div>
  </div>
</div>
```

### Guidelines

- Always use `data-color="neutral"` as default card color for meeting/document cards.
- Use `data-color="info"` for time badges and metadata tags.
- Use `ds-details` for expandable sections (agenda items, document lists) to keep the view compact.
- Use `ds-tag` with `data-color` for entity type badges (Journalpost, Moetemappe, etc.).
- Link back to eInnsyn with `ds-link` so users can access full documents.
- Set `data-color-scheme="auto"` on the root wrapper to respect the user's dark/light mode preference.
- Keep text in Norwegian (nynorsk or bokmål matching the user's preference).
