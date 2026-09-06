# AGENTS.md

Dette repository indeholder **BenchFleet**, en planlagt platform til inventory,
benchmarking og deterministisk vurdering af Windows-computere. Repositoryet er
ikke længere et generisk starterkit. Produktkoden er endnu ikke scaffoldet; den
aktuelle implementeringsstatus står i `docs/TASKS.md`.

## Før du arbejder

Læs altid, i denne rækkefølge:

1. `CLAUDE.md`
2. `docs/PRD.md`
3. `docs/ARCHITECTURE.md`
4. `docs/DATABASE.md`
5. `docs/ROADMAP.md`
6. `docs/TASKS.md`

`CLAUDE.md` er den permanente metodefil og må ikke erstattes eller regenereres.
Stackspecifikke kommandoer og arkitektur hører kun hjemme i dens afsluttende
sektioner **Kommandoer** og **Arkitektur i denne kodebase**. Dokumenterne i
`docs/` er source of truth. Hvis dokumentation, tests og kode er i konflikt, skal
konflikten identificeres eksplicit; gæt ikke, og omskriv ikke dokumentationen for
at skjule afvigelsen.

## Projektets faste rammer

- Prototype: Python 3.12+, Django 5+, Django REST Framework og SQLite.
- UI: Django templates og Bootstrap; HTMX kun hvor det konkret reducerer
  kompleksitet.
- Agent: separat installérbar Python-pakke til Windows i samme monorepo.
- Integration: agenten må ikke importere Django-apps; brug kun det versionerede
  JSON API under `/api/v1/`.
- Drift: native udvikling og én lokal backend-instance på lukket LAN.
- Ingen Docker, PostgreSQL, Redis, Celery, React, realtime eller SaaS-features
  uden et dokumenteret, observeret behov og opdateret arkitekturbeslutning.
- SaaS Pegasus er vurderet og fravalgt til prototypen. Genåbn kun vurderingen,
  hvis scope ændres mod SaaS/multi-organisation.

Disse valg er vedtaget i `docs/ARCHITECTURE.md` og `docs/DATABASE.md`. Ændr dem
ikke som en sideeffekt af en almindelig task.

## Domæneinvarianter

- Hostname er aldrig permanent device-identitet.
- Device-token tilhører ét UUID, lagres kun som digest i backend og må aldrig
  logges eller commits.
- Cross-device upload skal afvises server-side og dækkes af tests.
- Rå benchmarkværdier og units gemmes altid før eller sammen med afledte scores.
- Benchmarkversion og scoreversion er uafhængige.
- Ufuldstændige eller fejlede benchmarkruns må ikke publicere en komplet score.
- Inventory-snapshots og benchmarkruns er historik; overskriv ikke rå historiske
  data med seneste tilstand.
- Ukendte units, NaN, infinity, negative throughputværdier og inkonsistente
  payloads skal afvises.
- Valgfrie Windows-, WMI- og NVIDIA-collectors skal degradere kontrolleret og må
  ikke blokere minimumsflowet.
- BenchFleet indsamler aldrig dokumenter, browserhistorik, mail, brugerfiler,
  tastetryk eller screenshots.

## Arbejdsgang

Ved normal udvikling skal `prompts/NEXT_TASK.md` følges. Ved bugs bruges
`prompts/FIX_BUG.md`, og ved helhedsreview bruges `prompts/REVIEW_PROJECT.md`.
`prompts/` er source of truth; `.claude/commands/` er kun tynde wrappers.

1. Kør `git status` og undersøg relevante filer, patterns og eksisterende tests.
2. Vælg næste ikke-blokerede `TODO` i `docs/TASKS.md`, og sæt højst én task til
   `IN PROGRESS`.
3. Bekræft taskens acceptkriterier, afhængigheder, risici og målbare output før
   implementering.
4. Implementér den mindste robuste løsning. Opret ikke parallel funktionalitet,
   spekulative abstraktioner eller ny infrastruktur.
5. Test mindst happy path, invalid input, permissions, edge cases og relevante
   business rules. En bugfix skal så vidt muligt have en regressionstest.
6. Inspicér faktisk output. Ved UI-ændringer inspiceres siden i en browser; ved
   agent/collector-ændringer kræves realistisk Windows-output, når tasken siger
   det.
7. Kør relevante checks, gennemgå `git diff` og fjern debug/dead code.
8. Opdatér PRD, arkitektur, database, roadmap eller tasks, hvis beslutninger,
   kontrakter eller scope ændres.
9. Markér først en task `DONE`, når alle acceptkriterier og Definition of Done i
   `CLAUDE.md` er opfyldt. Hvis hardware- eller miljøverifikation mangler, behold
   en sandfærdig status og dokumentér begrænsningen.

Nye nødvendige tasks må registreres under **Opdagede tasks** i `docs/TASKS.md`,
men udvider ikke automatisk det aktive scope.

## Planlagte kommandoer

Produktkoden findes endnu ikke. Følgende kommandoer er en kontrakt for Task 001,
ikke aktuelt fungerende setup-instruktioner:

```bash
python -m venv .venv
python -m pip install -r requirements.txt
python manage.py migrate
python manage.py check
pytest
python manage.py runserver 0.0.0.0:8000
```

Når scaffoldet ændrer kommandoerne, opdatér både `README.md` og de stackspecifikke
sektioner i `CLAUDE.md`. Tilføj præcise agentkommandoer her, når Task 006 har
etableret dem. Opfind ikke midlertidige kommandoer i dokumentationen.

## Kode- og arkitekturregler

- Planlagte Django-apps er `devices`, `benchmarks`, `scoring`, `api` og
  `dashboard`; tilføj kun andre ved konkret domænebehov.
- Hold views tynde. Brug services til komplekse state changes og selectors til
  komplekse reads, men kun når de faktisk forbedrer strukturen.
- Håndhæv validering, authentication, authorization og object ownership på
  serveren; UI-kontrol er aldrig en sikkerhedsgrænse.
- Brug Django ORM, portable felter/constraints og korte transaktioner. Ingen
  SQLite-specifik SQL.
- Undgå N+1 queries, og tilføj indexes efter observerede query patterns frem for
  spekulation.
- Hold scoring som ren, deterministisk og testbar domænelogik; hardcod ikke
  baselines eller vægte i serializers eller templates.
- Hold platformsspecifik discovery bag små collectors med struktureret graceful
  degradation.
- Imports må ikke pakkes ind i brede `try/except`-blokke. Valgfrie dependencies
  isoleres ved tydelige adapter-/collector-grænser.
- Secrets, lokale agentdata, databaser og genererede artifacts må ikke commits.

## Definition of done for en ændring

En ændring er ikke færdig alene fordi koden kører. Bekræft:

- taskens acceptkriterier
- relevante nye og eksisterende tests
- migrations/schema og frisk database, når relevant
- permissions, secrets, logging og fejlscenarier
- faktisk output og platformskrav, når relevant
- opdateret dokumentation
- fokuseret og gennemgået diff uden debugkode
- største resterende risiko og næste relevante task

Afslut rapporten kort med **Implementeret**, **Verificeret**, **Største
kritikpunkt**, **Ændrede filer** og **Næste task**.
