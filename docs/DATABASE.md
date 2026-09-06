# BenchFleet database design

## Status

**Database:** SQLite (`db.sqlite3`) via Django ORM
**Beslutning:** Accepteret til prototype og mindre single-organisation-installationer

## Beslutningsgrundlag

Prototypen starter med 2 enheder og forventes gradvist testet med 5, 20 og 100 enheder. Uploads er små, periodiske og i første omgang manuelle. Der er én backend instance, ingen realtimekrav, moderate relationelle data og et stærkt behov for enkel installation, backup og nulstilling.

Den primære risiko er samtidige writes, ikke datamængden. Inventory og benchmark ingestion skal derfor bruge korte atomiske transaktioner, og database-lock-fejl skal måles før infrastrukturen ændres.

## Kandidater

### SQLite

**Status:** PASS / valgt

- indbygget Django-support og ingen ekstern service
- matcher lav write-concurrency og lokal LAN-prototype
- én fil gør backup og udviklingsreset enkelt
- ORM-designet kan migreres til PostgreSQL

Begrænsninger: én writer ad gangen, ikke egnet til flere app-instances på delt filsystem og kræver koordineret backup.

### Supabase

**Status:** FAIL for prototype, genovervej ved managed databasebehov

Managed PostgreSQL tilfører netværk, credentials og ekstern drift uden et aktuelt samtidighedsbehov. Supabase Auth overlapper Django authentication; Storage, Realtime og Edge Functions løser ingen MVP-krav.

### Traditionel PostgreSQL

**Status:** FAIL for prototype, dokumenteret migrationsmål

PostgreSQL er relevant ved vedvarende `database is locked`-fejl, høj upload-concurrency, flere backend-instances eller større SaaS-drift. Ingen af disse krav er observeret endnu.

## Portabilitetsregler

- Al normal adgang sker gennem Django ORM og migrations.
- Brug portable Django-felter, constraints og indexes; ingen SQLite-specifik SQL.
- UUID, timestamps og numeriske værdier valideres på applikationsniveau og lagres i standard Django-felter.
- Transaktioner holdes korte; benchmarks køres aldrig inde i databasetransaktioner.
- `db.sqlite3` og backups må ikke commits.

## Entiteter

Felttyper og endelige nullability-regler fastlægges i Task 001/002, men nedenstående er den godkendte konceptuelle model.

### Device

Formål: permanent backend-identitet og senest kendte basisdata.

```text
id                  BigAutoField / primary key
uuid                UUIDField / public immutable identifier
hostname            CharField
manufacturer        CharField / blank allowed
model               CharField / blank allowed
serial_number       CharField / blank allowed
machine_guid        CharField / blank allowed
token_digest        CharField / never expose
first_seen          DateTimeField
last_seen           DateTimeField
created_at          DateTimeField
updated_at          DateTimeField
```

Constraints/indexes:

- `uuid` er unique.
- `token_digest` er unique.
- Identity-felter er ikke enkeltvis unique, da OEM-placeholderværdier og manglende data forekommer.
- Index på `hostname`, `serial_number`, `manufacturer`, `model` og `last_seen` vurderes mod dashboardets faktiske queries.

### HardwareSnapshot

Formål: historisk inventory på et bestemt indsamlingstidspunkt.

```text
id
device_id           ForeignKey(Device, PROTECT)
collected_at        DateTimeField
cpu_manufacturer    CharField / blank allowed
cpu_model           CharField
cpu_architecture    CharField
physical_cores      PositiveSmallIntegerField / nullable
logical_cores       PositiveSmallIntegerField / nullable
cpu_frequency_mhz   FloatField / nullable
cpu_flags           JSONField / default list
memory_bytes        PositiveBigIntegerField
memory_available_bytes PositiveBigIntegerField / nullable
os_edition          CharField / blank allowed
os_version          CharField
os_build            CharField / blank allowed
os_architecture     CharField / blank allowed
bios_version        CharField / blank allowed
created_at          DateTimeField
```

Constraints/indexes: index `(device_id, -collected_at)`; ikke-negative størrelser/core counts valideres, og available memory må ikke overstige total memory.

### StorageDevice

Formål: storage-enheder, der tilhører et inventory snapshot. Dette præciserer PRD'ens foreløbige `device`-relation, fordi snapshotrelationen bevarer historik korrekt.

```text
id
hardware_snapshot_id ForeignKey(HardwareSnapshot, CASCADE)
model                 CharField / blank allowed
capacity_bytes        PositiveBigIntegerField
free_bytes            PositiveBigIntegerField
filesystem            CharField / blank allowed
mount_point            CharField / blank allowed
created_at             DateTimeField
```

Constraints: `free_bytes <= capacity_bytes`; eventuel uniqueness på `(snapshot, mount_point)` efter collectoradfærd er verificeret.

### BenchmarkRun

Formål: metadata og lifecycle for én benchmarkkørsel.

```text
id
public_id             UUIDField / client-generated idempotency key
device_id             ForeignKey(Device, PROTECT)
benchmark_version     CharField
profile               CharField / choices: quick
started_at            DateTimeField
completed_at          DateTimeField / nullable
status                CharField / started, completed, failed
agent_version         CharField
cpu_usage_before_pct  FloatField / nullable
ram_usage_before_pct  FloatField / nullable
ac_power_before       BooleanField / nullable
error_code            CharField / blank allowed
created_at            DateTimeField
```

Constraints/indexes:

- `public_id` er globalt unique for idempotent upload.
- `completed_at >= started_at` når completed_at findes.
- Index `(device_id, -completed_at)` og `(status, created_at)` hvis admin-query kræver det.

### BenchmarkMetric

Formål: rå benchmarkmåling og dens afledte subsystem-score.

```text
id
benchmark_run_id      ForeignKey(BenchmarkRun, CASCADE)
metric                CharField
raw_value             FloatField
unit                  CharField
score                 FloatField / nullable
created_at            DateTimeField
```

Constraints:

- unique `(benchmark_run_id, metric)`.
- `raw_value` og `score` skal være finite og ikke-negative for MVP-metrics.
- Tilladte metric/unit-kombinationer håndhæves af ingestion/scoring og testes; databaseconstraint tilføjes kun hvis den forbliver portabel og enkel.

### DeviceScore

Formål: reproducerbart score-resultat knyttet til det run, der skabte det.

```text
id
device_id             ForeignKey(Device, PROTECT)
benchmark_run_id      OneToOneField(BenchmarkRun, CASCADE)
cpu_single_score      FloatField
cpu_multi_score       FloatField
memory_score          FloatField
disk_score            FloatField
overall_score         FloatField
score_version         PositiveIntegerField
created_at            DateTimeField
```

Constraints/indexes:

- præcis én score record pr. benchmark run
- alle scores er finite og ikke-negative
- index `(device_id, -created_at)`; index på `overall_score` vurderes til sortering

## Relationer

```text
Device
  ├── 1:N HardwareSnapshot
  │          └── 1:N StorageDevice
  ├── 1:N BenchmarkRun
  │          ├── 1:N BenchmarkMetric
  │          └── 1:0..1 DeviceScore
  └── 1:N DeviceScore
```

`PROTECT` på Device-relationer forhindrer utilsigtet tab af historik. Eksplicit administrativ sletning kan senere implementeres som en kontrolleret service eller retention policy.

## Query patterns og indexing

Første queries er:

- device-liste sorteret på hostname, latest overall score, CPU/RAM eller last seen
- søgning på hostname, serienummer, producent, model og CPU-model
- seneste inventory og score pr. device
- benchmarkhistorik pr. device
- fleet average/min/max og count

Kun de oplagte foreign-key-/unique-indexes oprettes først. Sammensatte eller tekstsøgningsindexes tilføjes efter Django query inspection og måling; spekulativ PostgreSQL full-text search undgås.

## Dataintegritet og transaktioner

- Registration er atomisk, så parallelle requests ikke bevidst opretter dubletter; identity matching skal have en dokumenteret canonical key eller konfliktstrategi.
- Hvert inventory-upload opretter snapshot + storage rows i én kort transaktion.
- Hvert benchmark-upload opretter run, metrics og score i én transaktion for completed runs.
- Fejlede lokale benchmarks kan gemmes som run uden metrics/score efter en særskilt valideret kontrakt.
- API'et afviser NaN, infinity, negative throughputværdier, ukendte units og manglende quick-metrics.

## Persondata og secrets

BenchFleet indsamler device-data, som kan blive indirekte personhenførbare via hostname/assignment. Adgang begrænses til device-tokenets egen upload og autentificerede administratorer. Tokenet gemmes kun som digest. Dokumenter, brugernes filer og aktivitetsdata indsamles ikke.

Retention, eksport, sletning og eventuel audit trail skal fastlægges før reel organisationsdrift. Logs må ikke indeholde tokens, MachineGuid eller komplette payloads uden eksplicit redaction-vurdering.

## Data lifecycle

### Oprettelse

Registration opretter Device; efterfølgende uploads opretter immutable snapshots/runs og opdaterer kun device metadata/`last_seen`.

### Ændring

Rå benchmarkmetrics ændres normalt ikke. En ny scoremodel skaber/reberegner versionsmærkede scoredata via en eksplicit migration/management command frem for tavs mutation.

### Arkivering og sletning

Ingen automatisk retention i prototypen. Device-sletning er ikke en normal agentoperation. Før produktion fastlægges retention og en kontrolleret cascade/anonymiseringsproces.

## Backup og restore

Prototypeplan:

1. Stop writes eller brug SQLite online backup API/Django-kompatibelt backupværktøj.
2. Kopiér `db.sqlite3` til en timestamped placering uden for repositoryet.
3. Verificér backup med SQLite integrity check.
4. Restore til en separat fil/instans, kør migrations/checks og inspicér device/run counts.

Backupfrekvens, kryptering og off-site-opbevaring fastlægges før rigtig drift. En restore-test er et production-exit-kriterium; en filkopi uden restore-test tæller ikke som verificeret backup.

## Migration strategy

Django migrations versionsstyres og testes. Eksisterende migrations omskrives normalt ikke. Skift til PostgreSQL udløses af målte problemer som vedvarende lock errors, flere app-instances eller dokumenteret write-load og indebærer databasekonfiguration, dataeksport/import, sequence/verifikation og regressionstest — ikke omskrivning af domænelogik.

## Åbne databasebeslutninger

- [ ] Fastlæg canonical identity matching og konfliktbehandling før Device constraints låses.
- [ ] Beslut om scores bedst gemmes som `FloatField` eller fixed precision efter scoreformlen er defineret.
- [ ] Fastlæg retention og autoriseret sletning før pilot med reelle organisationsdata.
- [ ] Mål dashboardqueries før ekstra indexes tilføjes.
