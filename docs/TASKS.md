# BenchFleet tasks

## Arbejdsregler

Tilladte statusser er `TODO`, `IN PROGRESS`, `BLOCKED`, `REVIEW` og `DONE`. Kun én task er normalt `IN PROGRESS`. En task er først DONE når acceptkriterierne, relevante tests og faktisk output er verificeret. Afsluttede milestone-tasks flyttes til `docs/TASKS_ARCHIVE.md`.

## Aktuel milestone

**Fase 1: Agent → API → SQLite → Admin**

Mål: En fysisk Windows-computer kan registreres med stabil identitet, uploade et minimumsinventory gennem et autoriseret API og ses i Django Admin.

## Task 001: Scaffold Django/DRF og native udviklingsmiljø

**Status:** TODO

### Formål

Skab den mindste kørbare projektstruktur, som resten af milestone 1 kan bygges og testes på.

### Afhængigheder

Ingen.

### Forventede områder

```text
manage.py
config/
devices/
api/
requirements.txt
pytest.ini eller pyproject testconfig
.gitignore
README.md
CLAUDE.md (kun stackspecifikke kommandoer)
```

### Acceptkriterier

- [ ] Python 3.12+, Django 5+, DRF, pytest og pytest-django er pinned på et reproducerbart niveau.
- [ ] Projektet bruger SQLite `db.sqlite3` og kræver ingen ekstern service eller Docker.
- [ ] `python manage.py migrate`, `python manage.py check` og en smoke test virker fra frisk venv.
- [ ] `db.sqlite3`, virtualenv, secrets og lokale agentdata ignoreres af Git.
- [ ] Setup-, migrate-, test- og runserver-kommandoer er dokumenteret.

### Verifikation

- [ ] frisk native installation
- [ ] migrations på tom database
- [ ] Django system check
- [ ] pytest smoke test
- [ ] faktisk admin-login-side inspiceret

## Task 002: Lås device identity og enrollment-kontrakt

**Status:** TODO

### Formål

Fjern de to sikkerheds- og dataintegritetsuklarheder, der blokerer registration.

### Afhængigheder

- Task 001

### Acceptkriterier

- [ ] Prioritet og normalisering for BIOS serial, MachineGuid og motherboard serial er dokumenteret med placeholder/blank cases.
- [ ] Konfliktadfærd ved modsatrettede identifikatorer er dokumenteret; hostname er aldrig permanent nøgle.
- [ ] Enrollment vælges eksplicit mellem lukket-LAN åben registration og fælles bootstrap-token.
- [ ] Token generation, digest storage, one-time response og rotation/re-registration-adfærd er specificeret.
- [ ] Beslutningen er afspejlet i PRD, arkitektur og databaseconstraints før modelmigrationen oprettes.

### Verifikation

- [ ] eksempeltabel med first registration, repeat, hostname change, missing identifiers og conflict
- [ ] security review af enrollment og cross-device ownership

## Task 003: Implementér Device og HardwareSnapshot

**Status:** TODO

### Formål

Implementér den minimale portable datamodel og Admin-visning til device og CPU/OS-inventory.

### Afhængigheder

- Task 002

### Acceptkriterier

- [ ] Device og HardwareSnapshot følger `docs/DATABASE.md` samt Task 002-beslutningen.
- [ ] Constraints beskytter UUID/token uniqueness og gyldige inventoryværdier.
- [ ] Delete behaviour bevarer historik mod utilsigtet devicesletning.
- [ ] Begge modeller er registreret med nyttige list/search/filter-felter i Django Admin.
- [ ] Migrations virker på en frisk SQLite-database og model-/constrainttests består.

### Verifikation

- [ ] model happy paths og invalid values
- [ ] uniqueness/concurrency-relevant cases
- [ ] migrations fra tom database
- [ ] faktisk Admin-output inspiceret

## Task 004: Implementér registration API

**Status:** TODO

### Formål

Opret eller genkend en enhed atomisk og udsted per-device credentials efter den godkendte kontrakt.

### Afhængigheder

- Task 003

### Acceptkriterier

- [ ] `POST /api/v1/devices/register/` validerer og normaliserer input.
- [ ] Første registration returnerer UUID og token; token lagres kun som digest og logges ikke.
- [ ] Gentagen registration følger den dokumenterede idempotens-/tokenregel uden device-dublet.
- [ ] Manglende/placeholder-identitet og konfliktcases giver dokumenterede 4xx-svar.
- [ ] Enrollment-beskyttelsen fra Task 002 håndhæves.

### Verifikation

- [ ] happy path og repeat registration
- [ ] invalid/missing/conflicting input
- [ ] enrollment permission
- [ ] database rollback ved fejl
- [ ] response/log inspection for token leakage

## Task 005: Implementér device authentication og inventory API

**Status:** TODO

### Formål

Modtag CPU/OS-inventory sikkert og atomisk for det autentificerede device.

### Afhængigheder

- Task 004

### Acceptkriterier

- [ ] `POST /api/v1/devices/{uuid}/inventory/` kræver korrekt device-token.
- [ ] Gyldig payload opretter ét snapshot og opdaterer device metadata/last seen atomisk.
- [ ] Forkert, manglende og et andet devices token afvises uden writes.
- [ ] Ukendte/negative/inkonsistente værdier giver klare 4xx-fejl.
- [ ] Token og MachineGuid redacteres fra logs og fejloutput.

### Verifikation

- [ ] happy path
- [ ] invalid payload og edge values
- [ ] cross-device/permission tests
- [ ] transaction rollback
- [ ] serializer/API contract tests

## Task 006: Scaffold den installérbare Windows-agent

**Status:** TODO

### Formål

Opret en isoleret Python-pakke med CLI, konfiguration og logging uden afhængighed til Django-koden.

### Afhængigheder

- Task 004 (registration contract)

### Acceptkriterier

- [ ] `agent/pyproject.toml` definerer Python 3.12+, runtime dependencies og `benchfleet-agent` entry point.
- [ ] CLI tilbyder mindst `register`, `status`, `inventory`, `benchmark`, `upload` og `diagnose`; ikke-implementerede kommandoer fejler tydeligt.
- [ ] Config validerer server URL og gemmer UUID/token uden at logge tokenet.
- [ ] Windows default paths og en testoverride til temporary directory er defineret.
- [ ] Agentens unit tests kan køre på udviklingsplatformen uden WMI/NVIDIA.

### Verifikation

- [ ] installér package i frisk venv
- [ ] CLI help/status og invalid config
- [ ] log inspection for secrets
- [ ] tests med isoleret filesystem

## Task 007: Implementér identity samt minimal CPU/OS-collector

**Status:** TODO

### Formål

Indsaml registration- og minimumsinventory-data på Windows med graceful degradation.

### Afhængigheder

- Task 002
- Task 006

### Acceptkriterier

- [ ] Collector forsøger de godkendte identitetskilder og normaliserer placeholderværdier.
- [ ] CPU model, arkitektur, physical/logical cores samt Windows version/build indsamles.
- [ ] Valgfri WMI/pywin32/py-cpuinfo-mangel giver struktureret unavailable-status, ikke crash.
- [ ] Hardware parsing har fixtures/tests for realistiske og manglende Windows-resultater.
- [ ] Ingen private brugerdata indsamles.

### Verifikation

- [ ] parser happy/invalid/empty cases
- [ ] mocked dependency failures
- [ ] faktisk collectoroutput på fysisk Windows-maskine
- [ ] privacy/output review

## Task 008: Implementér agent registration og inventory upload

**Status:** TODO

### Formål

Forbind agentens CLI til API'et og fuldfør milestone 1 ende til ende.

### Afhængigheder

- Task 005
- Task 007

### Acceptkriterier

- [ ] `register` sender identity, gemmer UUID/token og håndterer serverfejl uden secret leakage.
- [ ] `inventory` indsamler og uploader minimumspayload med timeouts.
- [ ] HTTP client skelner mellem retrybare netværk/5xx-fejl og permanente 4xx-fejl.
- [ ] Integrationstest bruger en rigtig Django testserver eller dokumenteret contract fixture.
- [ ] En fysisk Windows-computer kan registrere, uploade og ses korrekt i Admin.

### Verifikation

- [ ] mocked HTTP happy/timeout/4xx/5xx
- [ ] API integrationstest
- [ ] faktisk Windows → Django → SQLite → Admin output inspiceret
- [ ] gentaget registration/upload kontrolleret for dubletter

## Milestone exit review

Når Task 001-008 er DONE:

- [ ] Kør samlet test suite og Django checks.
- [ ] Kør flowet mod en frisk database.
- [ ] Inspicér faktisk Windows-output og Admin-data.
- [ ] Sammenlign resultatet mod Fase 1 exit criteria i roadmap.
- [ ] Dokumentér største resterende risiko og flyt DONE tasks til `docs/TASKS_ARCHIVE.md`.
- [ ] Aktivér Fase 2-tasks (CPU benchmark, benchmarkmodeller, score v1 og to-device product proof).

## Opdagede tasks

Nye nødvendige tasks registreres her uden automatisk scopeudvidelse.

- Ingen endnu.

## Teknisk gæld

- Ingen produktkode findes endnu; teknisk gæld registreres først når den er konkret observeret.

## Bugs

- Ingen registrerede bugs.
