# BenchFleet project roadmap

## Princip

Roadmappet følger `make it work → make it reliable → make it scalable`. Hver fase skal levere et observerbart produktinkrement. Detaljerede aktive opgaver findes i `docs/TASKS.md`; funktioner fra senere faser implementeres ikke automatisk.

## Fase 0: Implementeringsklart fundament (aktuel)

Mål:

- låse prototypeproblem, MVP, ikke-mål og Definition of Done
- dokumentere Django/DRF, SQLite, native development og fravalg af Pegasus/Docker
- beskrive databasegrænser, API-sikkerhed og benchmark/scoring-versionering
- opdele første produktbevis i små verificerbare tasks

Exit criteria:

- source-of-truth-dokumenterne er konsistente og placeholders er fjernet
- åbne beslutninger, der blokerer registration/scoring, står eksplicit i tasks
- ingen produktkode er implementeret

## Fase 1: Agent → API → SQLite → Admin

Mål: en Windows-agent kan identificere og registrere en PC, uploade CPU-baseret inventory, og data kan ses i Django Admin.

Leverancer:

1. native Django/DRF-projekt med SQLite og testsetup
2. Device identity-regel, Device/HardwareSnapshot-model og admin
3. versioneret registration- og inventory-API med per-device auth
4. minimal installérbar agent med config, identity og CPU discovery
5. upload fra en fysisk Windows-referencecomputer

Exit criteria:

- registration er idempotent efter den godkendte identity-regel
- cross-device uploads afvises
- samme device kan registrere og uploade inventory til en frisk database
- faktisk data er inspiceret i Django Admin
- kritiske paths har automatiske tests

## Fase 2: Første performanceproduktbevis

Mål: to computere kan automatisk måles, uploades, scores og sammenlignes på CPU.

Leverancer:

- versionsstyret CPU single/multi quick benchmark
- BenchmarkRun/Metric og DeviceScore
- scoremodel v1 med godkendte baselines
- CLI-flow `benchmark` og upload
- Admin-visning af rå metrics og scores

Exit criteria:

```text
PC A → CPU benchmark → API → raw metric + score
PC B → CPU benchmark → API → raw metric + score
```

- begge enheder er automatisk målt, sendt, gemt og vist
- man kan identificere hurtigste CPU
- tre runs på referencehardware er evalueret mod ±10 %-målet

## Fase 3: Komplet quick benchmark og robust upload

Mål: udvide produktbeviset til RAM og disk og gøre agentflowet robust over for forventede fejl.

Leverancer:

- RAM memory-copy benchmark
- sekventiel disk write/read med sikker cleanup
- fuld vægtet overall score
- RAM/disk/Windows/storage inventory
- offline queue, idempotency og diagnose/status/upload-kommandoer
- graceful degradation for valgfri collectors

Exit criteria:

- alle fem raw metrics og fire subsystem-scores gemmes
- netværksfejl taber ikke payloads eller skaber dubletter ved retry
- disktempfil slettes ved success og fejl
- quick-profilet tager normalt 10-30 sekunder på referencehardware

## Fase 4: MVP-dashboard

Mål: IT-administratorer og teknikere kan forstå og sammenligne mindst to enheder uden at bruge Admin.

Leverancer:

- fleet summary med counts, average, lowest og highest
- søgbar og sortérbar device-liste
- device detail med hardware, rå metrics, scores og benchmarkhistorik
- simpel fleet average/difference

Exit criteria:

- alle MVP-krav i `docs/PRD.md` er opfyldt
- en bruger kan afgøre hvilken af to computere er hurtigst og hvorfor
- tomme, fejl- og loadingtilstande er forståelige
- permissions, N+1 queries og basal accessibility er verificeret

## Fase 5: Stabilisering og pilot

Fokus:

- reproducerbarhed på flere hardwareklasser
- outlier detection for samme model
- security review, token rotation og enrollment-beskyttelse
- backup/restore, logging og dokumenteret pilotdrift
- batteri- og udvidet hardware health-data
- XLSX-eksport

Exit criteria defineres før fasen startes ud fra erfaring fra rigtige enheder.

## Fase 6: Skalering efter observeret behov

Kun når målinger eller kundekrav kræver det vurderes:

- PostgreSQL
- rigtig WSGI/ASGI-deployment, HTTPS, monitoring og CI/CD
- central konfiguration og Windows Task Scheduler
- multi-organisation og rollebaseret adgang
- device groups, locations, replacement score og forecasting
- agent auto-update og integrationer

SQLite udskiftes ikke alene på grund af antal devices; triggeren er dokumenteret concurrency, lock-fejl, flere instances eller operationelle krav.

## Senere muligheder

- battery health og health score
- degradation alerts og avanceret peer comparison
- Teams/Power Automate/Intune/Graph integration
- AI-genererede forklaringer oven på deterministiske data

AI må ikke blive en afhængighed for benchmark eller scoring.
