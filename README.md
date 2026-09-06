# BenchFleet

BenchFleet er en planlagt platform til automatisk registrering, benchmarking og
vurdering af Windows-computere. En lokal Python-agent indsamler hardware- og
OS-inventory, udfører reproducerbare CPU-, RAM- og diskbenchmarks og sender de rå
resultater til en Django-backend. Her gemmes historikken, deterministiske scores
beregnes, og IT-administratorer kan finde og sammenligne langsomme enheder.

> **Projektstatus:** Dokumentation og prototypeplan er godkendt, men produktkoden
> er endnu ikke scaffoldet. Den aktive opgave er Task 001 i
> [`docs/TASKS.md`](docs/TASKS.md). Kommandoerne nedenfor er derfor den planlagte
> udviklingskontrakt og virker først, når Task 001 er implementeret.

## Hvorfor BenchFleet?

Traditionelt hardware-inventory fortæller, hvad en computer indeholder, men ikke
hvordan den faktisk performer. BenchFleet skal gøre forskellen målbar og hjælpe
en tekniker med at besvare:

1. Hvilken hardware og Windows-version har computeren?
2. Hvordan performer den under en ensartet, versionsstyret test?
3. Er CPU, RAM eller disk den sandsynlige flaskehals?
4. Hvordan klarer den sig mod andre computere i flåden?
5. Hvilke enheder bør undersøges først?

## Prototypeomfang

Den første komplette prototype skal:

- identificere en Windows-computer stabilt uden hostname som permanent nøgle
- indsamle CPU-, RAM-, disk- og Windows-data
- måle CPU single/multi, memory copy samt sekventiel disk read/write
- uploade gennem et versioneret og device-autoriseret JSON API
- bevare rå metrics og historik i SQLite
- beregne versionsstyrede subsystem- og overall-scores
- vise, sortere og sammenligne mindst to enheder
- bevare færdige uploads i en lokal kø ved netværksfejl

BenchFleet er i prototypefasen **ikke** MDM, endpoint security, software
deployment, multi-tenant SaaS eller en erstatning for specialiserede benchmark-
produkter. Se den fulde afgrænsning og acceptkriterier i
[`docs/PRD.md`](docs/PRD.md).

## Valgt arkitektur

```text
Windows PC
┌──────────────────────────────────────────┐
│ BenchFleet agent (Python CLI)            │
│ identity → inventory → benchmark → queue │
└───────────────────┬──────────────────────┘
                    │ /api/v1/ JSON + device-token
                    ▼
┌──────────────────────────────────────────┐
│ Django + Django REST Framework           │
│ API → validering → scoring → Admin/UI    │
└───────────────────┬──────────────────────┘
                    │ Django ORM
                    ▼
                  SQLite
```

| Område | Prototypevalg |
|---|---|
| Backend | Python 3.12+, Django 5+, Django REST Framework |
| Agent | Separat installérbar Python 3.12+ CLI til Windows |
| Frontend | Django templates og Bootstrap; HTMX kun ved konkret behov |
| Database | SQLite via Django ORM |
| Adgang | Per-device token til API; Django authentication til administration |
| Drift | Native installation på et lukket LAN |

SaaS Pegasus er vurderet og bevidst fravalgt til prototypen, fordi teams,
subscriptions og multi-tenancy ikke indgår i MVP. Docker, PostgreSQL, Redis,
Celery og React tilføjes heller ikke uden et observeret behov. Begrundelser og
genovervejelseskriterier findes i [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)
og [`docs/DATABASE.md`](docs/DATABASE.md).

## Roadmap

Udviklingen følger **make it work → make it reliable → make it scalable**:

1. **Fundament (aktuel):** implementeringsklar dokumentation og små tasks.
2. **Agent → API → SQLite → Admin:** første registrering og inventory fra en
   fysisk Windows-computer.
3. **CPU-produktbevis:** to computere benchmarkes, scores og sammenlignes.
4. **Komplet quick benchmark:** RAM/disk, offline queue og robust upload.
5. **MVP-dashboard:** fleet summary, søgning, sortering og device detail.
6. **Stabilisering/pilot**, derefter skalering alene efter målte behov.

Den autoritative faseplan findes i [`docs/ROADMAP.md`](docs/ROADMAP.md), mens
[`docs/TASKS.md`](docs/TASKS.md) altid indeholder den aktuelle arbejdsplan.

## Planlagt lokal udvikling

Task 001 skal etablere en reproducerbar native installation uden Docker. Når den
er færdig, er det forventede backend-flow:

```bash
python -m venv .venv

# Linux/macOS
source .venv/bin/activate

# Windows PowerShell
# .venv\Scripts\Activate.ps1

python -m pip install -r requirements.txt
python manage.py migrate
python manage.py check
pytest
python manage.py runserver 0.0.0.0:8000
```

Opret en administrator med `python manage.py createsuperuser`, når Django-
scaffoldet findes. `runserver` er kun accepteret til lokal udvikling og en
betroet prototype på lukket LAN; ekstern eksponering kræver produktionsserver,
HTTPS, sikre settings og et deployment-review.

Agentens installation og Windows-kommandoer dokumenteres, når agentpakken bliver
oprettet i Task 006. Påkrævede CLI-kommandoer bliver `register`, `status`,
`inventory`, `benchmark`, `upload` og `diagnose`.

## Repositorystruktur

```text
.
├── AGENTS.md              # arbejdsinstruktioner til AI-agenter
├── CLAUDE.md              # permanent udviklingsmetode og projektkommandoer
├── README.md
├── docs/
│   ├── PRD.md             # produktkrav og Definition of Done
│   ├── ARCHITECTURE.md    # stack, komponenter, flows og ADR'er
│   ├── DATABASE.md        # databasevalg, model og dataintegritet
│   ├── ROADMAP.md         # produktfaser og exit criteria
│   └── TASKS.md           # aktiv, prioriteret arbejdsplan
├── prompts/               # fælles workflows til alle AI-agenter
└── .claude/commands/      # tynde Claude Code-wrappers om prompts/
```

Den planlagte produktkode får Django-apps til `devices`, `benchmarks`, `scoring`,
`api` og `dashboard` samt en isoleret pakke under `agent/`. Agenten må ikke
importere Django-kode; komponenterne integrerer kun gennem `/api/v1/`.

## Arbejd på projektet

Repositoryet er optimeret til små, verificerbare ændringer med både mennesker og
AI-agenter:

1. Læs [`AGENTS.md`](AGENTS.md), [`CLAUDE.md`](CLAUDE.md) og source-of-truth-
   dokumenterne i `docs/`.
2. Vælg næste ikke-blokerede `TODO` i `docs/TASKS.md`; arbejd normalt kun på én
   task ad gangen.
3. Undersøg eksisterende patterns og afklar dokumentationskonflikter før kode.
4. Implementér det mindste robuste scope og test happy path, ugyldigt input,
   permissions, edge cases og relevante business rules.
5. Inspicér faktisk output, gennemgå diffen og opdatér dokumentationen, hvis en
   beslutning eller kontrakt ændres.
6. Markér først en task `DONE`, når dens acceptkriterier og projektets Definition
   of Done er opfyldt.

Brug [`prompts/NEXT_TASK.md`](prompts/NEXT_TASK.md) til normal udvikling,
[`prompts/FIX_BUG.md`](prompts/FIX_BUG.md) til fejl og
[`prompts/REVIEW_PROJECT.md`](prompts/REVIEW_PROJECT.md) til et helhedsreview. I
Claude Code findes de som `/next-task`, `/fix-bug` og `/review-project`.

## Source of truth

Ved uoverensstemmelser skal konflikten beskrives eksplicit — der må ikke gættes:

- `docs/PRD.md` ejer produktkrav, brugere, scope og succeskriterier.
- `docs/ARCHITECTURE.md` ejer stack, komponentgrænser og driftsvalg.
- `docs/DATABASE.md` ejer persistens, relationer og integritetsregler.
- `docs/ROADMAP.md` ejer faser og milestones.
- `docs/TASKS.md` ejer aktiv status, rækkefølge og task-acceptkriterier.
- `CLAUDE.md` ejer den permanente udviklingsmetode.

## Sikkerhed og data

- Device-token må aldrig logges eller commits og lagres kun som digest i
  backend.
- Et device må kun uploade til sit eget UUID; cross-device adgang skal testes.
- MachineGuid og komplette payloads må ikke ukritisk skrives til logs.
- BenchFleet indsamler ikke dokumenter, mail, browserhistorik, brugerfiler,
  tastetryk eller screenshots.
- HTTP accepteres kun på et betroet, lukket prototype-LAN; brug HTTPS før enhver
  ekstern eksponering.

## Licens

Der er endnu ikke tilføjet en licensfil. Projektet bør derfor betragtes som
proprietært, indtil ejeren vælger og tilføjer en licens.
