---
name: brreg
description: Query the Norwegian company registry (Enhetsregisteret / Brønnøysundregistrene) via its REST API. Use this skill ALWAYS when the user asks about Norwegian companies, organisations, businesses (bedrifter, foretak, selskaper), organisasjonsnummer (org.nr), company roles (styremedlemmer, daglig leder, revisor), sub-units (underenheter/avdelinger), or anything related to Brønnøysundregistrene, Brreg, Enhetsregisteret, or Foretaksregisteret.
---

# Brønnøysundregistrene — Enhetsregisteret API

**Base URL:** `https://data.brreg.no/enhetsregisteret/api`
**Auth:** None. **Format:** JSON (`Accept: application/json`).
**License:** NLOD — free for any purpose. Good practice to credit Brønnøysundregistrene.
**Rate limit:** No documented limit, but pagination capped: `(page+1) * size` cannot exceed 10,000. For full dataset, use bulk download at `/enheter/lastned`.
**Personal data:** Roles endpoint returns names and birth dates — handle under GDPR.

## Search entities

```
GET /enheter?navn=<name>&size=20
```

**Parameters:** `navn`, `organisasjonsnummer` (exact), `organisasjonsform` (AS, ASA, ENK, NUF, etc.), `kommunenummer` (4-digit), `naeringskode` (NACE, e.g. `62.010`), `konkurs`/`underAvvikling` (boolean), `registrertIMvaregisteret` (boolean), `fraRegistreringsdatoEnhetsregisteret`/`tilRegistreringsdatoEnhetsregisteret` (YYYY-MM-DD), `size` (max 100), `page` (0-indexed), `sort` (e.g. `navn,asc`)

**Response fields:** `organisasjonsnummer`, `navn`, `organisasjonsform.kode/.beskrivelse`, `registreringsdatoEnhetsregisteret`, `stiftelsesdato`, `hjemmeside`, `naeringskode1.kode/.beskrivelse`, `antallAnsatte`, `forretningsadresse` (with adresse[], postnummer, poststed, kommunenummer, land), `konkurs`, `underAvvikling`, `registrertIMvaregisteret`

**Pagination:** Response includes `page: { totalElements, totalPages, number }`. Navigate with `?page=N`. Max `(page+1)*size` cannot exceed 10,000.

## Single entity
```
GET /enheter/{organisasjonsnummer}
```

## Sub-units (underenheter)
```
GET /underenheter?overordnetEnhet={orgnr}&size=20
GET /underenheter/{orgnr}
```

## Roles (board, CEO, auditor)
```
GET /enheter/{organisasjonsnummer}/roller
```

Returns `rollegrupper[]`, each with `type.kode/.beskrivelse` and `roller[]`.

**Role codes:** `DAGL` (CEO), `STYR` (board member), `LEDE` (chair), `NEST` (vice chair), `REVI` (auditor), `REGN` (accountant), `KONT` (contact person), `EIKM` (owner KS/ANS)

Each role has `person.navn.fornavn/.etternavn`, `person.fodselsdato`, `fratraadt` (boolean).

## Org form codes

| Code | Type |
|---|---|
| `AS` | Aksjeselskap |
| `ASA` | Allmennaksjeselskap |
| `ENK` | Enkeltpersonforetak |
| `NUF` | Norskregistrert utenlandsk foretak |
| `ANS`/`DA` | Ansvarlig selskap / delt ansvar |
| `STI` | Stiftelse |
| `FLI` | Forening/lag |
| `KF` | Kommunalt foretak |
| `KOMM`/`FYLK`/`STAT` | Kommune/fylke/statlig |

## Common recipes

```bash
# Look up by org number
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter/923609016" -H "Accept: application/json"

# Board members and CEO
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter/923609016/roller" -H "Accept: application/json"

# Recently registered AS companies
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?fraRegistreringsdatoEnhetsregisteret=2026-03-01&organisasjonsform=AS&size=20&sort=registreringsdatoEnhetsregisteret,desc" -H "Accept: application/json"

# All AS in a municipality
curl -s "https://data.brreg.no/enhetsregisteret/api/enheter?kommunenummer=4640&organisasjonsform=AS&size=100" -H "Accept: application/json"
```
