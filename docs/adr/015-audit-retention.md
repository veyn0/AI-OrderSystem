# ADR-015: Audit-Retention 90 Tage / 10 Jahre

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Audit-Logs sind zentral für:
- Sicherheits-Diagnose (Login-Versuche, Anomalien).
- DSGVO-Auskunft.
- GoBD-Aufbewahrungspflicht (10 Jahre für steuerrelevante Daten).
- Streit-Klärung (was hat wer wann gemacht).

GoBD verlangt 10 Jahre Aufbewahrung finanzieller Daten. DSGVO verlangt jedoch Datenminimierung — überflüssig lange Aufbewahrung ist problematisch.

## Entscheidung

**Zwei separate Audit-Tabellen** mit unterschiedlicher Retention:

- **`audit_log`**: 90 Tage Aufbewahrung. Inhalt: Auth-Events, Permission-Änderungen, Speisekarten-Änderungen, Geräte-Lifecycle, Tenant-Lifecycle, etc. — alles, was nicht direkt steuerrelevant ist.
- **`financial_audit_log`**: 10 Jahre Aufbewahrung. Inhalt: Order-Lifecycle-Transitionen, Stornos, Refunds, Steuer-Overrides, Zahlungs-Vorgänge, TSE-Signaturen, Annahme-Entscheidungen.

Beide Tabellen sind **append-only** — UPDATE und DELETE auf Anwendungsebene blockiert via DB-Trigger. Retention-Pruning läuft als separater DB-User mit speziellem Recht.

## Alternativen

### Eine Tabelle, 10 Jahre Aufbewahrung für alles
Verworfen.
- DSGVO-Datenminimierung verletzt.
- Riesige Tabelle, langsame Queries.
- Login-Events von vor 5 Jahren sind nutzlos.

### Eine Tabelle, 90 Tage Aufbewahrung für alles
Verworfen.
- GoBD-Verstoß.
- Steuerrelevante Daten gehen verloren.

### Zwei Tabellen, gleiche Retention 5 Jahre
Verworfen.
- 5 Jahre für allgemeine Logs zu lang (DSGVO-Spannung).
- 5 Jahre für finanzielle Logs zu kurz (GoBD verlangt 10).

### Soft-Archivierung in günstigeren Storage nach X Jahren
Erwogen, vorgesehen für Phase 2.
- Aktuell: alles in PostgreSQL.
- Phase 2: alte Daten in dedizierte Archiv-Tabelle oder S3-kompatibles Storage migrieren.

## Konsequenzen

### Positiv
- **DSGVO-konform** durch Retention der nicht-finanziellen Logs nach 90 Tagen.
- **GoBD-konform** durch 10-Jahres-Retention der finanziellen Logs.
- **Performance** der allgemeinen Audit-Queries durch kleinere Tabelle.
- **Append-only** durch DB-Trigger — Manipulationsschutz auf DB-Ebene.

### Negativ
- **Klare Klassifizierung** nötig: jeder neue auditierbare Vorgang muss einer Tabelle zugewiesen werden. Bei Unsicherheit lieber `financial_audit_log` (sicher).
- **Backup-Größe** wächst durch financial_audit_log über 10 Jahre.
- **Retention-Job** muss zuverlässig laufen, sonst Drift.

### Neutral
- Beide Tabellen haben gleiches Schema-Pattern, Code zur Abfrage ähnlich.

## Querverweise

- [`../architecture/audit-log.md`](../architecture/audit-log.md)
- [`../compliance/gdpr.md`](../compliance/gdpr.md)
- [`../compliance/gobd-tse.md`](../compliance/gobd-tse.md)
