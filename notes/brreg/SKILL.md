---
name: brreg
description: Query the Norwegian company registry (Enhetsregisteret / Brønnøysundregistrene) via its REST API. Use this skill ALWAYS when the user asks about Norwegian companies, organisations, businesses (bedrifter, foretak, selskaper), organisasjonsnummer (org.nr), company roles (styremedlemmer, daglig leder, revisor), sub-units (underenheter/avdelinger), or anything related to Brønnøysundregistrene, Brreg, Enhetsregisteret, or Foretaksregisteret. Also trigger when looking up board members, CEOs, auditors, or ownership of Norwegian entities.
---

# Brønnøysundregistrene — Enhetsregisteret API

## Overview

The Enhetsregisteret (Entity Registry) is Norway's official registry of businesses and organisations. The API provides free, open access to data about all registered entities — companies, municipalities, NGOs, etc.

**Base URL:** `https://data.brreg.no/enhetsregisteret/api`
**Authentication:** None required.
**Format:** JSON (set `Accept: application/json`).

## Endpoints

### Search entities (enheter)

```
GET /enheter?navn=<name>&size=20
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `navn` | string | Search by entity name (partial match) |
| `organisasjonsnummer` | string | Exact org number lookup |
| `organisasjonsform` | string | Filter by org type code (e.g. `AS`, `ASA`, `ENK`, `NUF`, `FLI`) |
| `kommunenummer` | string | 4-digit municipality code |
| `fraRegistreringsdatoEnhetsregisteret` / `tilRegistreringsdatoEnhetsregisteret` | date | Registration date range (YYYY-MM-DD) |
| `fraStiftelsesdato` / `tilStiftelsesdato` | date | Founding date range |
| `konkurs` | boolean | Filter bankrupt entities |
| `underAvvikling` | boolean | Filter entities being dissolved |
| `underTvangsavviklingEllerTvangsopplosning` | boolean | Filter forced dissolution |
| `registrertIMvaregisteret` | boolean | Filter VAT-registered |
| `naeringskode` | string | NACE industry code (e.g. `06.100` for oil extraction) |
| `size` | integer | Results per page (default 20, max 100) |
| `page` | integer | Page number (0-indexed) |
| `sort` | string | Sort field (e.g. `navn,asc` or `registreringsdatoEnhetsregisteret,desc`) |

**Example:**
```bash
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?navn=Equinor&size=5" \
  -H "Accept: application/json"
```

**Response fields (per entity):**
- `organisasjonsnummer` — 9-digit org number
- `navn` — Official name
- `organisasjonsform.kode` / `.beskrivelse` — Type (AS, ASA, ENK, etc.)
- `registreringsdatoEnhetsregisteret` — Registration date
- `stiftelsesdato` — Founding date
- `hjemmeside` — Website
- `naeringskode1.kode` / `.beskrivelse` — Primary NACE code and description
- `antallAnsatte` — Number of employees
- `forretningsadresse` — Business address (adresse[], postnummer, poststed, kommunenummer, kommune, land)
- `postadresse` — Postal address
- `konkurs` — Bankrupt (boolean)
- `underAvvikling` — Being dissolved (boolean)
- `registrertIMvaregisteret` — VAT registered (boolean)
- `institusjonellSektorkode.kode` / `.beskrivelse` — Institutional sector

### Get single entity

```
GET /enheter/{organisasjonsnummer}
```

Returns full details for one entity.

### Search sub-units (underenheter)

Sub-units are branch offices, departments, etc.

```
GET /underenheter?overordnetEnhet={orgnr}&size=20
```

Supports the same filtering as `/enheter` plus:
- `overordnetEnhet` — Parent entity org number

### Get single sub-unit

```
GET /underenheter/{organisasjonsnummer}
```

### Get roles (roller) for an entity

```
GET /enheter/{organisasjonsnummer}/roller
```

Returns board members, CEO, auditor, etc. grouped by role type.

**Response structure:**
```json
{
  "rollegrupper": [
    {
      "type": { "kode": "DAGL", "beskrivelse": "Daglig leder" },
      "roller": [
        {
          "type": { "kode": "DAGL", "beskrivelse": "Daglig leder" },
          "person": {
            "fodselsdato": "1966-02-01",
            "navn": { "fornavn": "ANDERS", "etternavn": "OPEDAL" }
          },
          "fratraadt": false,
          "rekkefolge": 0
        }
      ]
    },
    {
      "type": { "kode": "STYR", "beskrivelse": "Styre" },
      "roller": [...]
    }
  ]
}
```

**Common role codes:** `DAGL` (daglig leder/CEO), `STYR` (styremedlem/board member), `LEDE` (styreleder/chair), `NEST` (nestleder/vice chair), `REVI` (revisor/auditor), `REGN` (regnskapsfører/accountant), `KONT` (kontaktperson), `EIKM` (eier i KS/ANS)

### Organisation form codes

| Code | Description |
|---|---|
| `AS` | Aksjeselskap |
| `ASA` | Allmennaksjeselskap |
| `ENK` | Enkeltpersonforetak |
| `NUF` | Norskregistrert utenlandsk foretak |
| `ANS` | Ansvarlig selskap |
| `DA` | Selskap med delt ansvar |
| `SA` | Samvirkeforetak |
| `STI` | Stiftelse |
| `FLI` | Forening/lag/innretning |
| `KF` | Kommunalt foretak |
| `KOMM` | Kommune |
| `FYLK` | Fylkeskommune |
| `STAT` | Statlig enhet |
| `SÆR` | Særlovselskap |
| `SF` | Statsforetak |

## Pagination

Responses include `page` metadata:
```json
{
  "page": {
    "size": 20,
    "totalElements": 175,
    "totalPages": 9,
    "number": 0
  }
}
```

Navigate with `?page=0`, `?page=1`, etc.

## Common Recipes

### Look up a company by org number
```bash
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter/923609016" \
  -H "Accept: application/json"
```

### Find all AS companies in a municipality
```bash
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?kommunenummer=4640&organisasjonsform=AS&size=100" \
  -H "Accept: application/json"
```

### Get board members and CEO
```bash
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter/923609016/roller" \
  -H "Accept: application/json"
```

### Find recently registered companies
```bash
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?fraRegistreringsdatoEnhetsregisteret=2026-03-01&organisasjonsform=AS&size=20&sort=registreringsdatoEnhetsregisteret,desc" \
  -H "Accept: application/json"
```

### Search by industry (NACE code)
```bash
# 62.010 = Computer programming
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?naeringskode=62.010&size=20" \
  -H "Accept: application/json"
```

## Notes

1. **Rate limiting:** No strict rate limit documented, but be respectful. Batch your needs.
2. **Data freshness:** Updated continuously as entities register changes.
3. **Roles endpoint** returns persons with birth dates — handle personal data responsibly.
4. **Org numbers** are always 9 digits. Sub-units have their own org numbers separate from the parent.
5. **NACE codes** follow the EU standard (Statistisk sentralbyrå maintains the Norwegian version, SN2007).

## License, Attribution & Rate Limits

- **License:** Norwegian Licence for Open Government Data (NLOD). Free for any purpose including commercial use.
- **Attribution:** Not explicitly required by NLOD for the Enhetsregisteret open data, but good practice to credit Brønnøysundregistrene.
- **Rate limiting:** No documented rate limit, but pagination is capped: `(page+1) * size` cannot exceed 10,000. For full dataset downloads, use the bulk download service instead of paginating the API.
- **Registration:** Not required.
- **Data accuracy:** Data may contain errors. Brreg does not guarantee correctness or currency.
- **Personal data:** The roles endpoint returns names and birth dates of real persons. Handle responsibly under GDPR — do not store or redistribute unnecessarily.
- **Versioning:** API is versioned via Accept header. If you don't specify a version, you get the latest (which may change). Pin your Accept header for stability.
- **Bulk downloads:** For full copies, use `https://data.brreg.no/enhetsregisteret/api/enheter/lastned` (CSV/JSON) instead of paginating. Updated daily.
