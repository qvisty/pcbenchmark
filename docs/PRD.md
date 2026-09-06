# BenchFleet product requirements document

## Produkt

**Navn:** BenchFleet
**Version:** 0.1
**Status:** Godkendt til prototypeplanlægning

BenchFleet er en platform til automatisk registrering, benchmarking og vurdering af Windows-computere. Produktet kombinerer hardware inventory, reproducerbare benchmarks og deterministisk scoring, så en organisation både kan se, hvad en computer indeholder, og hvordan den faktisk performer.

Udviklingsprincippet er:

```text
Make it work
↓
Make it reliable
↓
Make it scalable
```

Korteste robuste vej til en fungerende ende-til-ende-prototype prioriteres over infrastruktur og visuel polish.

## Problem

IT-administratorer og teknikere har typisk inventory-data som hostname, model, CPU, RAM og Windows-version, men mangler et konsistent mål for den oplevede maskinydelse. Ens modeller kan performe forskelligt på grund af diskproblemer, termisk throttling, belastning, utilstrækkelig RAM eller hardwarefejl. Det gør fejlsøgning og prioritering af reparation eller udskiftning subjektiv.

BenchFleet skal besvare:

1. Hvilken hardware har computeren?
2. Hvor hurtigt fungerer den under en ensartet test?
3. Hvilket subsystem er den sandsynlige flaskehals?
4. Hvordan performer den mod andre computere?
5. Hvilke computere bør undersøges først?

## Målsætning

Prototypeproduktet skal gøre det muligt at:

1. identificere en Windows-computer stabilt uden at bruge hostname som permanent ID
2. registrere CPU, RAM, disk og Windows-information
3. køre en kort, versionsstyret CPU-, RAM- og diskbenchmark
4. sende inventory og rå benchmarkresultater til et Django REST API
5. gemme data og historik i SQLite
6. beregne versionsstyrede subsystem- og overall-scores
7. vise, sortere og sammenligne mindst to computere

## Ikke mål

Prototypeversionen er ikke:

- MDM, antivirus, patch management eller endpoint security
- Intune-erstatning, fjernstyring eller software deployment
- multi-tenant SaaS
- en erstatning for Geekbench eller Cinebench
- et produktionsklart interneteksponeret system
- automatisk scheduling, GPU-benchmarking, health score eller replacement score
- PostgreSQL-, Redis-, Celery-, React-, Kubernetes- eller Docker-baseret uden et observeret behov

## Brugere

### IT-administrator

Skal kunne se alle computere, søge og sortere efter performance, identificere dårlige enheder og sammenligne hardware og benchmarkresultater.

### Tekniker

Skal kunne åbne en enhed, se rå målinger og subsystem-scores og forstå, hvilken komponent der sandsynligvis skaber forskellen.

### Ledelse (senere)

Skal senere kunne bruge aggregerede data til udskiftningsplanlægning og hardwarebudgetter. Det er ikke et MVP-workflow.

## Centrale brugerrejser

### Journey 1: Registrér første computer

1. Teknikeren konfigurerer agentens serveradresse.
2. Agenten indsamler stabile identitetsoplysninger og kalder registrerings-endpointet.
3. Backend opretter eller genkender enheden og returnerer `device_uuid` og et device-token.
4. Agenten gemmer konfigurationen lokalt uden at logge tokenet.

Acceptkriterier:

- [ ] Samme stabile maskinidentitet giver ikke dubletter ved gentagen registrering.
- [ ] Hostname kan ændres uden at ændre enhedens permanente identitet.
- [ ] Manglende obligatoriske felter returnerer en valideringsfejl uden delvise data.
- [ ] Tokenet lagres sikkert nok til en lokal prototype og vises kun ved udstedelse.

### Journey 2: Benchmark og upload

1. Teknikeren kører `benchfleet-agent benchmark`.
2. Agenten registrerer benchmarkbetingelser og inventory.
3. Agenten kører quick-profilets CPU single, CPU multi, memory copy samt sekventiel disk write/read.
4. Agenten uploader inventory og benchmarkpayload.
5. Ved netværksfejl køes payloaden lokalt og kan uploades igen.

Acceptkriterier:

- [ ] Quick benchmark måler alle fem påkrævede metrics og tager normalt 10-30 sekunder.
- [ ] CPU multi bruger multiprocessing, ikke threads.
- [ ] Midlertidige diskfiler slettes, også når benchmark fejler.
- [ ] API'et gemmer rå værdier, units, benchmarkversion og agentversion.
- [ ] Gentaget upload af samme køelement skaber ikke utilsigtede dubletter.

### Journey 3: Sammenlign computere

1. Administratoren logger ind i Django.
2. Dashboardet viser fleet-nøgletal og en sortérbar enhedsliste.
3. Administratoren åbner to enheder og ser hardware, seneste scores og historik.
4. Administratoren kan identificere hurtigste enhed og det subsystem, der forklarer forskellen.

Acceptkriterier:

- [ ] Oversigten viser devices, benchmarked devices, gennemsnit, laveste og højeste score.
- [ ] Listen kan søge på hostname, serienummer, producent, model og CPU.
- [ ] Listen kan sorteres på hostname, overall score, CPU, RAM og last seen.
- [ ] Detailvisningen viser både rå værdier og subsystem-/overall-scores.

## Funktionelle krav

### Must have

#### F1. Device identity og registration

Agenten forsøger at indsamle BIOS-serienummer, Windows MachineGuid, motherboard-serienummer, producent, model og hostname. Backend udsteder UUID og et per-device token. Den præcise identity matching-regel fastlægges og testes i Task 002; hostname må aldrig være eneste permanente nøgle.

#### F2. Hardware- og OS-inventory

Minimum: CPU-model/arkitektur/core counts, samlet og tilgængelig RAM, primær disks model/kapacitet/fri plads/filsystem samt Windows edition/version/build/arkitektur, hostname, producent, model og BIOS-version. Valgfri GPU-, WMI- og NVIDIA-data må fejle uden at stoppe agenten.

#### F3. Quick benchmark

Profilen producerer rå værdier for `cpu_single`, `cpu_multi`, `memory_copy_mb_s`, `disk_read_mb_s` og `disk_write_mb_s`. Benchmarkalgoritmen versionsstyres, og startbetingelserne CPU usage, RAM usage og AC power registreres når tilgængelige.

#### F4. API og authentication

Versioneret JSON API under `/api/v1/` tilbyder registration, inventory-upload og benchmark-upload. Device endpoints kræver korrekt per-device token og må ikke tillade et device at skrive data for et andet UUID. Administrative sider bruger Djangos authentication.

#### F5. Persistens og historik

Device, hardware snapshots, storage devices, benchmark runs, metrics og scores gemmes via Django ORM i SQLite. Alle benchmark runs bevares; rå data må aldrig erstattes af kun en score.

#### F6. Deterministisk scoring

Scoring producerer scores for CPU single, CPU multi, RAM og disk samt et overall-resultat. Første vægte er centralt defineret: CPU single 30 %, CPU multi 25 %, RAM 15 % og disk 30 %. `score_version = 1` gemmes med resultatet.

#### F7. Agent CLI og offline queue

CLI'en tilbyder `register`, `status`, `inventory`, `benchmark`, `upload` og `diagnose`. Mislykkede uploads gemmes i en persistent lokal kø og forsøges igen uden tab. Logs bruger INFO/WARNING/ERROR og må ikke indeholde tokens.

#### F8. Administration og dashboard

Alle kernemodeller registreres i Django Admin. MVP-dashboardet viser fleet-nøgletal, søgbar/sortérbar device-liste, device detail og benchmarkhistorik via Django templates, Bootstrap og kun HTMX hvor det reducerer kompleksitet.

### Should have efter første produktbevis

- sammenligning med fleet average og samme model
- simpel benchmarkgraf og outlier-detektion
- mere detaljeret GPU-, storage- og batteriinventory
- XLSX-eksport

### Could have senere

- health score, alerts og degradation detection
- replacement score, device groups og locations
- central agentkonfiguration og Windows Task Scheduler
- multi-organisation, RBAC, agent auto-update og forecasting

## Forretningsregler

1. Rå benchmarkmålinger og unit gemmes altid sammen med afledte scores.
2. Scoring og benchmarkalgoritmer har uafhængige versioner.
3. Overall score beregnes kun af kompatible, komplette metrics efter den aktive scoreversion.
4. En benchmarkfejl må registreres som fejlet run og må ikke publicere en misvisende komplet score.
5. Kun det token, der tilhører et UUID, må uploade data for enheden.
6. Valgfrie collectors må ikke blokere minimumsinventory.
7. BenchFleet indsamler aldrig dokumenter, browserhistorik, mail, brugerfiler, tastetryk eller screenshots.
8. HTTP er kun accepteret på et lukket prototype-LAN; HTTPS er obligatorisk før ekstern eksponering.

## Roller og rettigheder

| Aktør | Læse | Oprette/uploade | Redigere | Slette | Administration |
|---|---:|---:|---:|---:|---:|
| Device agent | Egen API-identitet | Kun egne data | Nej | Nej | Nej |
| Django staff/admin | Ja | Ja | Ja | Efter policy | Ja |
| Uautentificeret | Nej | Kun registration | Nej | Nej | Nej |

Registration-endpointets beskyttelse mod uønsket enrollment på et ikke-isoleret netværk er et åbent sikkerhedsspørgsmål.

## Ikke-funktionelle krav

### Benchmark

- Quick profile tager normalt 10-30 sekunder på målgruppen.
- Tre runs under sammenlignelige forhold bør variere mindre end ±10 %; afvigelser og målemetode dokumenteres.
- Diskbenchmark bruger en begrænset midlertidig fil (mål: 256 MB) og rydder op robust.

### Robusthed

- Agenten fortsætter ved manglende valgfri Windows/NVIDIA-afhængighed.
- Netværksfejl taber ikke færdige payloads.
- API-validering er atomisk for hvert upload.

### UX

- Primær dashboardoplevelse optimeres til desktop, men basale sider er responsive.
- Sortering, tomme tilstande og forståelige fejlmeddelelser er påkrævet.
- Farve må ikke være eneste signal for score eller status.

### Privatliv og logging

- Kun hardware-, OS-, benchmark- og nødvendige device identity-data indsamles.
- Tokens og andre secrets må aldrig logges eller commits.
- Retention og sikker sletning fastlægges før produktion; prototypen bevarer benchmarkhistorik.

## Success metrics og Definition of Done for prototype

Prototypens centrale hypotese er bevist når:

1. To Windows-computere kan registreres, benchmarkes og uploades automatisk.
2. Begge resultater gemmes og kan ses i Django med rå metrics og scores.
3. En administrator kan afgøre, hvilken computer er hurtigst, og hvilket subsystem skaber forskellen.
4. Kritisk business logic (registration, API ownership, scoring, payloadvalidering og offline queue) har automatiske tests.
5. Tre sammenlignelige benchmarkruns på en referencecomputer evalueres mod ±10 %-målet.

## Åbne spørgsmål

- Hvilken kombination og prioritet af hardwareidentifikatorer skal afgøre, om registration genkender en eksisterende enhed?
- Skal første registration være åben på det lukkede LAN, bruge et fælles enrollment-token eller kun tillades af en administrator?
- Hvilke konkrete baselineværdier og normaliseringsformel skal `score_version = 1` bruge?
- Skal referenceplatformen til benchmarkreproducerbarhed være en bestemt fysisk Windows-computer eller flere hardwareklasser?
- Hvor længe skal device- og benchmarkhistorik opbevares, og hvem må slette en enhed?
