# ADR-004: PostgreSQL als Datenbank

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform braucht eine Datenbank, die:

- relationale Modellierung gut unterstützt (Aggregate mit klaren Beziehungen).
- starke Mandantentrennung erlaubt.
- ACID-Transaktionen über mehrere Tabellen.
- semi-strukturierte Daten ermöglicht (JSONB für Audit-Payloads, Acceptance-Rules).
- ausgereift, frei und langfristig stabil ist.
- GoBD-konforme Aufbewahrung erlaubt.

## Entscheidung

**PostgreSQL 16+** als einzige relationale Datenbank für die Plattform.

Im Demo-Profil eingebettet via `io.zonky.test:embedded-postgres` (eine echte PostgreSQL-Instanz, kein H2-In-Memory-Surrogat).

## Alternativen

### MariaDB / MySQL
Verworfen.
- **Row-Level Security** ist in MySQL/MariaDB nicht nativ verfügbar (Workarounds via Views sind fehleranfällig).
- JSON-Unterstützung ist da, aber JSONB von PostgreSQL ist ausgereifter (Indexierung).
- PostgreSQL hat insgesamt das fortgeschrittenere Feature-Set für komplexe Anwendungen.

### MongoDB / NoSQL
Verworfen.
- Unsere Domäne ist stark relational (Order → Items, Tenant → Locations → Tables).
- ACID-Transaktionen kritisch für Bestell-/Zahlungs-Logik.
- GoBD-Aufbewahrung verlangt strukturierte Daten.

### SQLite
Verworfen für Prod.
- Kein Multi-User-Concurrent-Write.
- Keine RLS.
- Skaliert nicht über einen Single-Process hinaus.

Für Einzelkomponenten-Tests ggf. eingesetzt.

### CockroachDB / YugabyteDB
Postgres-kompatible verteilte DBs. Verworfen für MVP — Overkill, kein Bedarf an Multi-Region.

### Cloud-DB-as-a-Service (AWS RDS, GCP Cloud SQL)
Verworfen für MVP. Standalone-Anforderung verbietet das.

## Konsequenzen

### Positiv
- **Row-Level Security (RLS)** als Eckpfeiler der Mandantentrennung.
- **JSONB** mit GIN-Indizes für flexible Spalten (Audit-Payload, Acceptance-Rules).
- Generated Columns, Window Functions, CTEs — alles, was Reporting-Queries braucht.
- Reife Backup-Tools (`pg_basebackup`, WAL-Streaming).
- Open-Source und PostgreSQL-Lizenz (sehr permissiv).
- Spring-Integration ausgezeichnet (auto-config, Testcontainers).
- Embedded-Variante für Demo-Profil verfügbar.

### Negativ
- Keine native Multi-Datenbank-Sharding (für Phase 4+ relevant).
- Operations-Komplexität höher als SQLite.
- Voll-Backups bei großen DBs zeit-intensiv (mit Streaming-Backups gemildert).

### Neutral
- PostgreSQL Major-Versionen erfordern Migration alle paar Jahre, aber das ist gut dokumentiert.

## Querverweise

- [`../architecture/data-model.md`](../architecture/data-model.md)
- [`../architecture/tenancy.md`](../architecture/tenancy.md)
- [`../architecture/standalone-mode.md`](../architecture/standalone-mode.md)
- [ADR-005](005-multi-tenancy.md), [ADR-006](006-standalone-mode.md), [ADR-010](010-jpa-jooq.md)
