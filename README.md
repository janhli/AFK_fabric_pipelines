# SSB Statistikkbank – datapipeline

Henter, holder oversikt over og standardiserer data fra SSB Statistikkbanken i Microsoft Fabric. Bygget som en kjede av notebooks i `statbank_staging`-lakehouset, orkestrert av en Fabric Pipeline.

## Navnekonvensjon

Alle tabellreferanser i koden er konsekvent **3-delte**: `statbank_staging.<schema>.<tabell>` (f.eks. `statbank_staging.pipeline.ssb_config`). Dette gjør at ingen notebook er avhengig av at riktig lakehouse "tilfeldigvis" er satt som standard i notebook-konteksten.

De fire schemaene (opprettet av `00`):

| Schema | Inneholder |
|---|---|
| `pipeline` | Driftstabeller: `ssb_config`, `ssb_metadata`, `ssb_load_queue*`, `ssb_update_log` |
| `kodeverk` | Kodeverk/oppslagstabeller, f.eks. kommunekorrespondanse |
| `ssb` | De ferdige, standardiserte SSB-statistikktabellene (Silver) |
| `fhi` | Reservert for fremtidig bruk |

## Notebooks – oversikt

| # | Notebook | Kjøres | Rolle |
|---|---|---|---|
| 1 | `00_ssb_build_schemas_and_config` | Én gang, ved oppsett | Oppretter schemaene og en tom `ssb_config` |
| 2 | `01_ssb_metadata_oppsett` | Ved oppsett / ad hoc | Bygger `ssb_metadata` – en rik datakatalog. Kan også kjøres som bibliotek fra `02` |
| 3 | `02_ssb_config_admin` | Ad hoc | Legg til / oppdater / slett tabeller i `ssb_config` |
| 4 | `03_ssb_oppdateringsdetector` | Periodisk (Fabric Pipeline) | Sjekker SSB for oppdateringer, fyller lastekøen |
| 5 | `04_ssb_ingest_landing` | Periodisk (Fabric Pipeline, etter 03) | Henter rådata fra SSB til Landing (Bronze) |
| 6 | `05_ssb_standardize` | Periodisk (Fabric Pipeline, etter 04) | Standardiserer til Silver-tabeller |
| – | `03_ssb_oppdateringsdetector_debugger` | Manuelt, ved feilsøking | Tømmer lastekøen manuelt |
| – | `05_ssb_standardize_debugger` | Manuelt, ved feilsøking | Kjører 05 mot én valgt tabell, uavhengig av køen |

## Den daglige pipelinen

```
03_ssb_oppdateringsdetector   – sjekker hva som er nytt hos SSB
        ↓  fyller ssb_load_queue / ssb_load_queue_critical
04_ssb_ingest_landing         – henter rådata (JSON) til Landing layer
        ↓  Files/landing/ssb/statbank/<tabellnr>/snapshot_date=.../period=...
05_ssb_standardize            – parser, beriker og skriver til Silver
        ↓
   statbank_staging.ssb.statbank_<tabellnr>
```

Denne kjeden kjøres periodisk (f.eks. daglig) fra en Fabric Pipeline. `00`, `01` og `02` er derimot manuelle verktøy du bruker ved oppsett eller når du vil legge til/endre hvilke tabeller som følges – de er ikke del av den automatiske kjøringen.

## Første gangs oppsett

1. Kjør **`00_ssb_build_schemas_and_config`** – oppretter schemaene og en tom `ssb_config`.
2. Kjør **`02_ssb_config_admin`** – fyll inn `table_list` med tabellnumrene du vil følge, kjør Run All. Dette fyller `ssb_config` og bygger samtidig `ssb_metadata`.
3. *(Valgfritt)* Kjør **`01_ssb_metadata_oppsett`** manuelt hvis du vil oppdatere/utvide katalogen for flere tabeller enn det du la inn i steg 2.
4. Sett opp en Fabric Pipeline som kjører **`03` → `04` → `05`** i rekkefølge, periodisk.
5. Første gang `03` kjører vil alle tabellene ha `reason = first_load`, siden ingen er lastet ned ennå.

## Avhengigheter

- Microsoft Fabric Lakehouse (`statbank_staging`) med Delta- og schema-støtte
- SSB PXWeb v2 API (`https://data.ssb.no/api/pxwebapi/v2`)
- Python-pakker: `requests`, `pyspark`, `delta`, `pandas`

---

## 00_ssb_build_schemas_and_config

**Kjøres:** Kun én gang, ved aller første oppsett.

Oppretter de fire schemaene nevnt over, og en tom `ssb_config`-tabell med riktig struktur. Trygg å kjøre flere ganger (`CREATE SCHEMA IF NOT EXISTS`), men kjør den ikke på nytt hvis `ssb_config` allerede har data du vil beholde – den overskriver tabellen.

---

## 01_ssb_metadata_oppsett

**Kjøres:** Ved oppsett, og ad hoc når du vil oppdatere/utvide metadata-katalogen.

Henter rik, maskinlesbar metadata fra SSB PXWeb v2 API – tittel, dimensjoner, måleenhet, oppdateringsfrekvens, KLASS-klassifikasjonskoder, kontaktperson og hele den rå API-responsen. Dette er en frittstående **datakatalog**, ment for oppdagelse og dokumentasjon – ingen andre notebooks er avhengige av at den finnes eller er oppdatert for å fungere.

Kan kjøres på to måter:
- **Frittstående** (`LIBRARY_MODE = False`): henter og skriver for alle tabellnumre i `TABELLNR_LIST_MANUAL` (eller en pipeline-parameter/miljøvariabel).
- **Som bibliotek** (`%run 01_ssb_metadata_oppsett {"LIBRARY_MODE": true}`): laster kun funksjonene, henter/skriver ingenting selv. Dette er slik `02_ssb_config_admin` bruker den – for å slippe å duplisere hente-logikken, og for å bygge opp `ssb_metadata` automatisk hver gang en tabell legges til i `ssb_config`.

**Output:** `statbank_staging.pipeline.ssb_metadata` – én rad per tabell. Skrives med MERGE (`upsert_metadata_row`), så den kan oppdateres for én og én tabell uten å slette resten av katalogen.

---

## 02_ssb_config_admin

**Kjøres:** Ad hoc ved behov.

Administrasjonsverktøyet for hele pipelinen – dette er stedet du går for å bestemme *hvilke* SSB-tabeller som skal følges. Elleve nummererte steg (se markdown-headeren i selve notebooken), styrt av parametrene i parameter-cellen: sett parameteren for steget du vil kjøre, la resten stå som `None`/tom liste, og kjør Run All.

Henter automatisk fersk metadata fra SSB (via `01` i bibliotekmodus) når du legger til eller resetter en tabell – både for å fylle `ssb_config`-feltene (kategori, frekvens → lookback) og for å bygge/oppdatere raden i `ssb_metadata`-katalogen samtidig.

**Output:** `statbank_staging.pipeline.ssb_config` – driftstabellen resten av pipelinen leser fra:

| Kolonne | Beskrivelse |
|---|---|
| `table_id` | SSB tabellnummer |
| `table_name` | Tabellnavn fra SSB API |
| `frequency` | Oppdateringsfrekvens (årlig/kvartalsvis/månedlig/ukentlig/daglig) |
| `category` | Tematisk kategori fra SSB |
| `lookback_periods` | Antall perioder som refreshes ved hver oppdatering |
| `priority` | `CRITICAL` eller `NORMAL` |
| `last_downloaded_timestamp` | Når `04` sist hentet rådata til Landing |
| `last_loaded_timestamp` | Når `05` sist skrev ferdig standardisert data til Silver |

---

## 03_ssb_oppdateringsdetector

**Kjøres:** Periodisk fra Fabric Pipeline (f.eks. daglig) – første steg i den automatiske kjeden.

Leser `ssb_config`, spør SSB sitt API om hvilke tabeller som er publisert/oppdatert nylig, og sammenligner mot `last_downloaded_timestamp` for hver tabell. Tabeller havner i lastekøen av to grunner:
- **first_load** – tabellen er aldri hentet ned før
- **updated** – SSB har publisert noe nyere enn det vi sist hentet

Hvor mange dager bakover det spørres om hos SSB beregnes dynamisk ut fra når pipelinen sist kjørte vellykket (med 1 dags buffer), slik at ingenting glipper mellom kjøringer selv om en kjøring skulle feile eller bli forsinket.

**Output:**

| Tabell | Innhold |
|---|---|
| `ssb_load_queue` | Alle tabeller som trenger oppdatering (CRITICAL + NORMAL) |
| `ssb_load_queue_critical` | Kun tabeller med `priority = CRITICAL` |
| `ssb_update_log` | Én logg-rad per kjøring, brukes til å beregne søkevindu neste gang |

Hver kjøring merkes med et felles `check_timestamp`, som følger med tabellene videre gjennom kjeden (inn i `04` sitt manifest og `05` sin sporing) – nyttig for å se hvilken `03`-kjøring som faktisk ligger bak et gitt sett med data.

*(`03_ssb_oppdateringsdetector_debugger` er en liten, frossen wrapper-notebook som manuelt tømmer `ssb_load_queue` – kun til bruk under feilsøking, ikke del av normal drift.)*

---

## 04_ssb_ingest_landing

**Kjøres:** Fra Fabric Pipeline, rett etter `03`. Kan også kjøres manuelt – leser da fra gjeldende `ssb_load_queue`.

### Hva den gjør, steg for steg

**Steg 1 – Les køen.** Starter med å lese `ssb_load_queue` – tabellen `03` nettopp har fylt. For hver rad vet den hvilket tabellnummer, og hvor mange perioder som skal refreshes (`lookback_periods`).

**Steg 2 – Hent tabell-info fra SSB.** Spør SSB om grunnleggende info, blant annet en `updated`-timestamp for når SSB sist publiserte nye tall for akkurat denne tabellen.

**Steg 3 – Change detection.** En ekstra sikkerhetssjekk: sammenligner SSBs `updated`-timestamp mot forrige nedlastede snapshot (via manifestet). Har ingenting endret seg, hoppes tabellen over.

**Steg 4 – Hent metadata.** Spør SSB om tabellens struktur – dimensjoner og tilgjengelige tidskoder. `get_time_dimension` finner navnet på tidsdimensjonen uansett hva SSB har kalt den, siden dette ikke er konsekvent på tvers av tabeller.

**Steg 5 – Last ned periode for periode.** SSB har en grense på hvor mye data man kan hente i én spørring, så dataene hentes én periode (år/kvartal/måned) om gangen og lagres som egne JSON-filer. Historiske perioder utenfor `lookback_periods` beholdes uendret fra forrige snapshot – kun de siste periodene hentes på nytt, siden SSB av og til reviderer ferske tall.

**Steg 6 – Manifest og _SUCCESS.** Etter at alle perioder er lagret skrives en `manifest.json` med metadata om kjøringen (antall observasjoner, SSBs `updated`-timestamp, og hvilken `03`-kjørings `check_timestamp` som utløste nedlastingen). `_SUCCESS`-filen er kvittering på at snapshotet er komplett.

**Steg 7 – Oppdater ssb_config.** Etter vellykket nedlasting kalles `mark_table_as_downloaded()`, som skriver dagens tidspunkt til **`last_downloaded_timestamp`** i `ssb_config`. Dette er det `03` leser neste gang den kjører, for å vite om tabellen er hentet siden sist.

### Mappestrukturen som bygges opp

```
Files/
└── landing/ssb/statbank/
    └── 07459/
        └── snapshot_date=2026-04-17/
            ├── manifest.json
            ├── _SUCCESS
            ├── period=1986/
            │   └── ssb_07459_1986.json
            └── ...
```

Hver dag notebooken finner nye data opprettes en ny `snapshot_date=`-mappe. Historikk bevares.

---

## 05_ssb_standardize

**Kjøres:** Fra Fabric Pipeline, rett etter `04` – siste steg i den automatiske kjeden.

### Hva den gjør

Leser JSON-filene `04` la igjen i Landing, og bygger de ferdige statistikktabellene:

1. **Finn nyeste snapshot** for hver tabell i køen, og les alle periode-filene i den.
2. **Flat ut JSON-Stat2-formatet** – SSB leverer alle tall som én lang liste, uten å si eksplisitt hvilken kombinasjon av dimensjoner (region, kjønn, alder, år, ...) hvert tall tilhører. `parse_jsonstat2_to_dataframe` regner dette ut og bygger en tabell med én kolonne per dimensjon (`{dimensjon}_code` / `{dimensjon}_label`). For uvanlig store/brede tabeller distribueres denne utregningen over klyngen i stedet for å kjøres ett sted, for å unngå at det tar for lang tid eller går tom for minne.
3. **Beriker med kommunekorrespondanse** – oversetter historiske kommunekoder til dagens gjeldende kommunenummer, og merker av hvilke rader som gjelder Akershus.
4. **Kjører datakvalitetssjekker** – teller rader, nullverdier, duplikater og årsspenn, og varsler (uten å stoppe) hvis noe ser mistenkelig ut.
5. **Skriver til Silver** (`statbank_staging.ssb.statbank_<tabellnr>`) – full overwrite ved første last, ellers en MERGE på kun de siste periodene, for å unngå å skrive om hele historikken hver gang.
6. **Merker Silver-tabellen med lesbar metadata** – tabellbeskrivelse (tittel fra `ssb_config`), kategori, oppdateringsfrekvens, tidspunkt for siste standardisering og en lenke tilbake til SSB, satt som `COMMENT`/`TBLPROPERTIES` på tabellen. Gjør at noen som blar i lakehouset eller SQL-endepunktet ser hva tabellen inneholder uten å slå opp andre steder.
7. **Kjører en tabellspesifikk raffineringsnotebook** (`ssb_refine_<tabellnr>`) hvis en slik finnes, for ekstra tilpasning av akkurat den tabellen.
8. **Oppdaterer `ssb_config`** – skriver dagens tidspunkt til **`last_loaded_timestamp`** for alle tabeller som ble ferdig behandlet.

Underveis sammenlignes også `check_timestamp` fra `04` sitt manifest mot køens nåværende innhold – dette varsler (uten å stoppe kjøringen) hvis `03` skulle ha kjørt på nytt og overskrevet køen mens `04`/`05` fortsatt jobbet med et eldre snapshot.

*(`05_ssb_standardize_debugger` er en tynn wrapper: setter et par debug-parametre – blant annet `QUEUE_TABLE` satt til `ssb_config` i stedet for den ekte køen, og `DEBUG_TABLE_ID` for å teste én bestemt tabell – og kaller så den ekte `05_ssb_standardize` via `notebook.run()`. Slik kan man teste én tabell uten å røre eller være avhengig av den faktiske pipeline-køen.)*

---

## Tabelloversikt

| Tabell | Schema | Skrives av | Leses av |
|---|---|---|---|
| `ssb_config` | `pipeline` | `00` (oppretter), `02` (administrerer) | `03`, `04`, `05` |
| `ssb_metadata` | `pipeline` | `01` (også kalt fra `02`) | – (frittstående katalog, ingen notebooks leser fra den i dag) |
| `ssb_load_queue` / `ssb_load_queue_critical` | `pipeline` | `03` | `04`, `05` |
| `ssb_update_log` | `pipeline` | `03` | `03` (leser sin egen historikk for å beregne søkevindu) |
| `statbank_staging.ssb.statbank_<tabellnr>` | `ssb` | `05` | Rapportering / Power BI |

## De to tidsstemplene i ssb_config

Lett å forveksle, men de betyr forskjellige ting:

- **`last_downloaded_timestamp`** – satt av **`04`** når rådata er hentet ned til Landing.
- **`last_loaded_timestamp`** – satt av **`05`** når data er ferdig standardisert til Silver.

`03` leser kun `last_downloaded_timestamp` (for å avgjøre om noe trenger en ny nedlasting), mens `05` leser `last_loaded_timestamp` (for å avgjøre om en tabell skal skrives på nytt fra bunnen eller kun oppdateres med MERGE). Sirkelen lukkes slik:

```
03 fyller køen  →  04 henter og merker last_downloaded_timestamp
                →  05 standardiserer og merker last_loaded_timestamp
                →  03 ser dette neste gang den kjører
```
