# BenchFleet architecture

## Status

**Technology status:** Besluttet for prototype

- Backend: Python 3.12+, Django 5+, Django REST Framework
- Frontend: Django templates, Bootstrap, HTMX efter konkret behov
- Agent: Python 3.12+ CLI til Windows
- Database: SQLite via Django ORM
- Authentication: per-device token til API; Django authentication til administration
- Hosting: lokal native Django-installation på lukket LAN i prototypen
- Background jobs/realtime: ingen

## Arkitekturprincipper

1. Bevis flowet agent → API → database → visning med to enheder før skalering.
2. Gem rå data før afledte værdier.
3. Versionsstyr benchmark og scoring uafhængigt.
4. Isolér platformsafhængig discovery bag små collectors med graceful degradation.
5. Brug Django ORM og portable felttyper; undgå SQLite-specifik SQL.
6. Hold views tynde, men introducer kun services/selectors når logikken reelt kræver det.
7. Ingen ny infrastruktur uden et målt behov.

## Teknologivurdering

### Valg A: Plain Django + DRF (valgt)

Fordele:

- matcher relationel datamodel, admin, authentication, REST API og server-renderet dashboard
- kort vej til fungerende prototype og få moving parts
- Django Admin giver milestone 1 et brugbart UI næsten gratis
- samme Python-sprog i agent, benchmark og backend

Ulemper:

- agentpakken og webapplikationen kræver tydelig adskillelse i monorepoet
- Django er større end et mikroframework, men de indbyggede funktioner bruges konkret

### Valg B: SaaS Pegasus som Django-fundament

Fordele:

- kan accelerere et klassisk SaaS-produkt med færdig authentication, teams, UI, subscriptions og deployment-integrationer
- bygger på Django og kan derfor principielt rumme domænet

Ulemper:

- BenchFleet-prototypen kræver ikke teams, subscriptions, social login, email flows eller multi-tenancy
- generated SaaS-overflade og frontend/build-valg øger omfanget før kernehypotesen er bevist
- kan gøre en enkel LAN-prototype sværere at forstå og vedligeholde

**Vurdering:** Pegasus er vurderet som foretrukket kandidat i henhold til projektmetoden, men fravælges til prototypen, fordi dets primære accelerators ligger uden for MVP. Vurderingen bør genåbnes før en SaaS/multi-organisation-version. Aktuel Pegasus-dokumentation og generator kunne ikke verificeres i dette miljø på grund af blokeret ekstern netværksadgang; ingen versionsspecifikke antagelser er derfor lagt til grund.

### Valg C: FastAPI + separat frontend/admin

Fordele:

- let API-fokus og stærk typebaseret request-validering

Ulemper:

- kræver separate valg eller implementering for admin, webauthentication, ORM/migrations og dashboard
- længere vej til et komplet administrativt prototypeflow end Django

### Valg D: Django + React + PostgreSQL/container stack

Fordele:

- relevant ved kompleks klienttilstand og høj samtidighed

Ulemper:

- dobbelt frontend/backend-tooling og unødvendig driftskompleksitet
- løser ikke et observeret prototypeproblem

## Valgt løsning

Plain Django + DRF, SQLite og server-renderede templates vælges. Windows-agenten er en separat installérbar Python-pakke i samme repository. Dette følger produktets eksplicitte prioritet om hurtigst mulig ende-til-ende-værdi og er reversibelt: agent/API-kontrakten er versioneret, og ORM-modellerne holdes portable til PostgreSQL.

## Udviklingsmiljø

- OS: backend kan udvikles på Windows, macOS eller Linux; agentens Windows collectors verificeres på Windows
- Runtime: Python 3.12+
- Package management: `venv` + `pip` + `requirements.txt` for den dokumenterede prototypevej; agenten får `pyproject.toml`
- Database: lokal `db.sqlite3`
- Frontend tooling: ingen Node-toolchain i første omgang; Bootstrap kan leveres som statisk asset/CDN efter sikkerhedsvurdering
- Docker: nej

Docker løser ikke et aktuelt problem: Django og SQLite kræver ingen eksterne services. Native setup giver færre trin og er samtidig nødvendigt for realistisk Windows-agenttest.

## Systemoversigt

```text
Windows PC
┌─────────────────────────────────────────────┐
│ BenchFleet agent CLI                        │
│ config → identity/inventory → benchmarks    │
│              → local queue → HTTP client    │
└──────────────────────┬──────────────────────┘
                       │ HTTP(S) / JSON / token
                       ▼
Django
┌─────────────────────────────────────────────┐
│ DRF API → validation → scoring              │
│ Django Admin + templates/HTMX dashboard     │
└──────────────────────┬──────────────────────┘
                       │ Django ORM
                       ▼
                 SQLite db.sqlite3
```

## Repository- og komponentstruktur

```text
manage.py
config/                 Django settings and root URLs
devices/                identity, inventory models and admin
benchmarks/             runs, metrics and benchmark ingestion
scoring/                pure deterministic scoring rules
api/                    v1 serializers, authentication and views
dashboard/              server-rendered fleet views
agent/
  pyproject.toml
  benchfleet_agent/
    collectors.py
    benchmark.py
    api.py
    config.py
    queue.py
    cli.py
```

| Komponent | Ansvar | Vigtige afhængigheder |
|---|---|---|
| Agent config/CLI | kommandoer, lokal konfiguration og orkestrering | stdlib, platform services |
| Collectors | identity, hardware, OS og conditions | `psutil`, `py-cpuinfo`, valgfri WMI/NVML |
| Benchmark engine | versionsstyrede CPU/RAM/disk workloads | stdlib, multiprocessing, psutil |
| Agent API/queue | payloads, retries og persistent offlinekø | `requests`, filesystem |
| Devices | device identity og hardware snapshots | Django ORM |
| Benchmarks | runs og rå metrics | Devices, Django ORM |
| Scoring | ren normalisering og vægtning | benchmark metrics |
| API v1 | validation, device auth og atomisk ingestion | DRF, domain apps |
| Dashboard/Admin | queries og menneskelig visning | Django auth, domain apps |

## Centrale flows

### Registration

1. Agent indsamler tilgængelige stabile identifikatorer.
2. `POST /api/v1/devices/register/` validerer payload og matcher/opretter atomisk.
3. Serveren genererer UUID samt kryptografisk token; kun en hash/digest bør gemmes.
4. Token returneres én gang og gemmes lokalt af agenten.

Registration matching og enrollment-beskyttelse er åbne designbeslutninger og skal låses før endpointet implementeres.

### Inventory

Agent sender et samlet snapshot. API'et validerer UUID/token ownership og opretter snapshot og storage records i én transaktion. Seneste device metadata opdateres uden at overskrive historikken.

### Benchmark

Agent sender run metadata og rå metrics. API'et validerer metric-navne, finite numeriske værdier, units og benchmarkversion. Et komplet validt run gemmes atomisk, hvorefter den rene scoringfunktion beregner subsystem- og overall-score og opretter `DeviceScore`.

### Offline queue

Færdige payloads skrives atomisk til en lokal queue-fil før uploadforsøg. Ved success slettes elementet; ved retrybar fejl bevares det. Køelementer har en stabil client-generated idempotency key, så tvetydige netværksfejl ikke skaber dubletter.

## API

**Stil:** REST/JSON
**Base:** `/api/v1/`
**Klient:** BenchFleet-agenten

Første endpoints:

```text
POST /api/v1/devices/register/
POST /api/v1/devices/{uuid}/inventory/
POST /api/v1/devices/{uuid}/benchmarks/
```

API-kontrakten dokumenteres med serializer-tests. Ikke-understøttede versioner og metrics afvises eksplicit. Uploads bør acceptere en idempotency key. HTTP timeouts sættes i agenten; kun netværksfejl, 429 og relevante 5xx-fejl retries automatisk med bounded backoff. 4xx-valideringsfejl logges uden secrets og beholdes/markeres til diagnose frem for uendeligt retry.

## Authentication og authorization

- Django sessions og standard permissions bruges til Admin/dashboard.
- Hvert device får et tilfældigt token, som kun autoriserer det UUID tokenet tilhører.
- Token sammenlignes sikkert mod en lagret digest og roteres senere via et eksplicit flow.
- Object ownership håndhæves i API-view/authentication-laget og testes mod cross-device upload.
- Tokens filtreres fra logs og fejlbeskeder.
- Prototype-HTTP er kun tilladt på et betroet, lukket LAN; ekstern eksponering kræver HTTPS, secure settings og deployment-review.

## Benchmark- og scoringsgrænser

Benchmark engine er ansvarlig for workload og rå resultater, ikke organisationsspecifik rangering. Scoring tager validerede rå metrics og en versioneret konfiguration som input og returnerer deterministiske scores. Baselines og vægte ligger centralt og må ikke hardcodes i serializers/templates. Ændret algoritme med ikke-sammenligneligt output kræver ny benchmark- eller scoreversion.

## Integrationer og valgfrie dependencies

WMI, pywin32 og NVIDIA NVML importeres ikke som ubetingede krav til alle platforme. Hver collector returnerer enten normaliserede data eller en struktureret unavailable/error-status. Fejl må ikke indeholde token eller følsomme systemdata. Eksterne cloudintegrationer findes ikke i MVP.

## Background jobs og realtime

Ingen Redis, Celery, scheduler eller websocket introduceres. Agenten køres manuelt. Windows Task Scheduler vurderes efter det manuelle flow fungerer. Serverberegning sker synkront, fordi uploads er små og scoring billig.

## Deployment

Prototype:

```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver 0.0.0.0:8000
```

Dette er alene til et lukket LAN. Før produktion kræves mindst rigtig WSGI/ASGI-server, `DEBUG=False`, HTTPS, secret management, allowed-host-konfiguration, backup/restore-test, statiske assets, logging/monitorering og rollbackprocedure.

## Arkitekturbeslutninger

### ADR 001: Plain Django frem for Pegasus i prototypen

**Status:** Accepted for prototype

**Valg:** Start med et minimalt Django-projekt. Genåbn Pegasus-vurderingen ved multi-organisation/SaaS-scope.

**Konsekvens:** Hurtigere og mindre prototype, men SaaS-funktioner kan senere kræve integration eller migrering.

### ADR 002: SQLite som standard

**Status:** Accepted

**Valg:** SQLite via Django ORM indtil målinger viser write-concurrency eller multi-instance-behov.

**Konsekvens:** Minimal drift; én backend instance og omhyggelige korte transaktioner.

### ADR 003: Monorepo med separat agentpakke

**Status:** Accepted

**Valg:** Web og agent deler repository, men agenten har eget `pyproject.toml` og må ikke importere Django-apps.

**Konsekvens:** Hurtig koordination og fælles kontrakttests; senere repo-split er mulig.

### ADR 004: Ingen container eller job queue

**Status:** Accepted

**Valg:** Native runtime og manuelle agentkørsler i MVP.

**Konsekvens:** Enkel start; scheduling og robust produktionsdrift er bevidst udskudt.
