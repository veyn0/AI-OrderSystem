# ADR-005: Shared Database + Row-Level Security für Multi-Tenancy

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform soll mehrere Restaurants (Tenants) bedienen. Mandantentrennung ist sicherheitskritisch — Datenleck zwischen Tenants ist KO-Kriterium. Wachstumshorizont: 50 Tenants.

Drei Standard-Modelle für Multi-Tenancy:

1. **Shared Database, Shared Schema, Tenant-ID-Spalte** (mit oder ohne RLS).
2. **Shared Database, Schema-pro-Tenant** (jedes Tenant eigenes DB-Schema).
3. **Database-pro-Tenant** (jedes Tenant eigene DB-Instanz).

## Entscheidung

**Shared Database, Shared Schema, Tenant-ID-Spalte + PostgreSQL Row-Level Security (RLS).**

Jede tenant-bezogene Tabelle hat `tenant_id UUID NOT NULL` mit RLS-Policy, die Filter `tenant_id = current_setting('app.tenant_id')::uuid` erzwingt.

Defense-in-Depth: zusätzliche Tenant-Filter im Repository-Layer und Permission-Prüfung im Service-Layer.

## Alternativen

### Schema-pro-Tenant
- Vorteil: noch stärkere Trennung, einfaches Daten-Backup pro Tenant.
- Nachteile:
  - 50 Tenants = 50 Schemas, jedes Schema-Migration muss auf allen Schemas laufen → Operations-Aufwand.
  - Schema-Dispatching im Hibernate komplex (`MultiTenantConnectionProvider` mit Schema-Switch pro Request).
  - JOIN über Tenants (z.B. Plattform-Reporting) wird unmöglich oder sehr komplex.
  - Performance-Overhead durch Schema-Wechsel pro Connection.

Für unseren Wachstumshorizont (50 Tenants) Overkill.

### Database-pro-Tenant
- Vorteil: stärkste Isolation. Bei einem kompromittierten Tenant kein DB-Zugriff auf andere.
- Nachteile:
  - Operations-Komplexität: 50 PostgreSQL-Instanzen oder 50 Datenbanken.
  - Connection-Pooling pro Tenant.
  - Resource-Overhead massiv.
  - Cross-Tenant-Reporting (Plattform-weite Statistiken) über Federation = komplex.
  - Migration auf 50 DBs.

Vermutlich relevant ab 1000+ Tenants oder Premium-Tier mit dezidierten Datenbanken.

### Shared Schema ohne RLS
- Verworfen. Tenant-Filter nur im Application-Code = einzige Schicht. Ein Bug → Datenleck.
- RLS gibt zusätzliche unabhängige Schicht.

### Shared Schema mit RLS aber ohne App-Layer-Check
- Verworfen. Wenn RLS umgangen wird (z.B. SQL-Injection mit hochprivilegiertem User, BYPASSRLS-Pool), ist alles offen. Defense-in-Depth nötig.

## Konsequenzen

### Positiv
- **RLS** ist eine harte DB-seitige Garantie — selbst bei Anwendungs-Bugs ist Cross-Tenant-Read blockiert.
- Eine DB → ein Backup, eine Migration, eine Connection-Pool.
- Cross-Tenant-Reporting (für Plattform-Admins) möglich über separaten BYPASSRLS-Pool.
- Shared-Schema ist effizienter (Index-Wiederverwendung über Tenants).
- 50 Tenants in einer DB unproblematisch.

### Negativ
- **Sicherheits-kritisches Detail**: jede neue Tabelle muss explizit RLS aktivieren. Wir haben CI-Check, aber Disziplin nötig.
- **Setting der Session-Variable** `app.tenant_id` muss bei jedem Request korrekt erfolgen. Filter-Layer + Connection-Interceptor sorgen dafür.
- **Plattform-Pool** mit `BYPASSRLS` ist mächtig — Code, der ihn nutzt, muss durch Code-Review.
- Migration zu DB-pro-Tenant für Premium-Kunden später möglich, aber nicht trivial — Hibernate `MultiTenantConnectionProvider` würde dann genutzt.

### Neutral
- Sicherheits-Tests müssen auf Cross-Tenant-Versuche prüfen — das ist Pflicht-Suite.

## Querverweise

- [`../architecture/tenancy.md`](../architecture/tenancy.md)
- [`../architecture/data-model.md`](../architecture/data-model.md)
- [ADR-004](004-postgresql.md)
