# Monitoring, Logging, Alerting

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

## Logging

### Format

JSON, strukturiert. Beispiel-Entry:

```json
{
  "timestamp": "2026-04-28T12:34:56.789Z",
  "level": "INFO",
  "logger": "de.deinrestaurant.order.OrderService",
  "message": "Order paid",
  "thread": "http-nio-8080-exec-3",
  "tenantId": "...",
  "userId": "...",
  "orderId": "...",
  "traceId": "abc-123",
  "spanId": "def-456"
}
```

Konfiguration in `logback-spring.xml` mit JsonEncoder (logstash-logback-encoder).

### Log-Levels und Verwendung

| Level | Verwendung |
|---|---|
| ERROR | Unerwartete Fehler, die Operation-Aufmerksamkeit erfordern (z.B. Adapter-Fehlschlag, DB-Ausnahmen) |
| WARN | Auffällige, aber nicht kritische Vorgänge (Lockouts, Abgewiesene Webhooks) |
| INFO | Geschäfts-relevante Events (Login, Order-Created, Order-Paid) |
| DEBUG | Diagnose-Details (NICHT in Prod aktiv außer manuell für Debugging) |
| TRACE | Sehr detailliert (NICHT in Prod) |

### Korrelations-IDs

- Jeder eingehende HTTP-Request bekommt eine `traceId` (UUID).
- `traceId` wird in MDC (Mapped Diagnostic Context) gehalten und in jeden Log-Eintrag eingefügt.
- Antwort-Header `X-Trace-Id` enthält die ID — nützlich für Bug-Reports.
- Bei Domain-Events wird `traceId` weitergereicht.

### Sensible Daten in Logs

**Verboten:**
- Passwörter / Klartext-Passwörter (nicht mal hashed).
- PINs / Klartext-PINs.
- Geräte-Tokens.
- Vollständige Kreditkarten-Daten.
- API-Keys.

**Erlaubt mit Vorsicht:**
- E-Mail-Adressen (Identifikation).
- Adressen (nur in Audit-Log und nur wo nötig).

**Pflicht:** Code-Review prüft Log-Statements. Keine `LOG.info("User logged in: " + user)` mit `toString()`-Defaults, die alles ausgeben.

### Log-Retention

- 30 Tage lokal in JSON-Files via Docker-Logging-Driver.
- 30 Tage in zentraler Aggregation (Phase 2).

## Metriken

### Quelle

- **Spring Boot Actuator + Micrometer** mit Prometheus-Exporter.
- **PostgreSQL Exporter** (`postgres_exporter`) für DB-Metriken.
- **Node Exporter** für Host-Metriken (CPU, RAM, Disk, Netzwerk).
- **Caddy Metriken** native Prometheus-Endpunkt.

### Wichtige Metriken (Application)

| Metrik | Was sie misst |
|---|---|
| `http.server.requests` | Histogramm der Request-Latenzen, gelabelt nach Endpoint, Method, Status |
| `jvm.memory.used` | Heap-Verbrauch |
| `jvm.gc.pause` | GC-Pausen-Längen |
| `hikaricp.connections.active` | Aktive DB-Connections |
| `hikaricp.connections.usage` | Connection-Pool-Auslastung |
| `application.orders.created` (Counter) | Bestellungen erstellt, gelabelt nach Tenant, OrderType |
| `application.orders.paid` (Counter) | Bestellungen bezahlt |
| `application.orders.cancelled` (Counter) | Stornos |
| `application.acceptance.timeouts` (Counter) | Annahme-Timeouts (verwerfen) |
| `application.sse.subscriptions.active` (Gauge) | Aktive SSE-Verbindungen |
| `application.tse.signature.duration` (Histogramm) | TSE-Signatur-Latenz |
| `application.payment.duration` (Histogramm) | Payment-Latenz, Provider-gelabelt |

### Wichtige Metriken (System)

| Metrik | Schwelle Alert |
|---|---|
| Host CPU Usage | > 80 % für 10 min |
| Host RAM Usage | > 90 % für 5 min |
| Host Disk Usage `/` | > 80 % |
| Host Disk Usage `/var/lib/postgresql` | > 70 % |
| Postgres Connections | > 80 % von max_connections |
| Postgres Long Queries | > 5 s laufend |
| Postgres Replication Lag | > 5 min |
| WAL Archive Backlog | > 10 unverarbeitete WAL-Files |

## Dashboards (Grafana)

### Dashboard „Plattform-Übersicht"
- Bestellungen/h Plattform-weit (letzten 24 h).
- Aktive Tenants, aktive Geräte (Heartbeat letzte 5 min).
- HTTP-Latenz P50/P95/P99.
- Error-Rate (5xx) der API.
- DB-Connection-Pool-Auslastung.

### Dashboard „Pro Tenant"
- Bestellungen/h für einen ausgewählten Tenant.
- Pro Standort: Bestelltyp-Verteilung.
- Annahme-Timeouts, Stornos.

### Dashboard „System Health"
- Host-Metriken.
- DB-Performance.
- WAL-Archivierung-Status.

### Dashboard „Backups"
- Letzter erfolgreicher Base-Backup.
- WAL-Backlog.
- Letzter erfolgreicher Test-Restore.

## Alerting

### Setup

- **Prometheus AlertManager** sendet Alerts.
- **Empfänger:** E-Mail an Owner (initial), in Phase 2 Telegram/Discord-Bot.

### Alert-Klassen

| Severity | Reaktionszeit | Beispiele |
|---|---|---|
| **Critical** | sofort (innerhalb 15 min) | Service down, DB-Verbindung weg, 5xx-Rate > 5 % |
| **Warning** | innerhalb 1 h | hohe CPU, Disk-Voll-Warnung, lange Queries |
| **Info** | werktäglich Review | Backup-Test fällig, Dependency-Updates verfügbar |

### Konkrete Alert-Definitionen

```yaml
groups:
  - name: critical
    rules:
      - alert: BackendDown
        expr: up{job="backend"} == 0
        for: 2m
        annotations:
          summary: "Backend ist nicht erreichbar"

      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m]))
          > 0.05
        for: 5m
        annotations:
          summary: "5xx-Rate > 5 % seit 5 min"

      - alert: DatabaseConnectionsExhausted
        expr: hikaricp_connections_pending > 0
        for: 2m

      - alert: DiskFillingFast
        expr: |
          predict_linear(node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"}[6h], 24*3600) < 0
        for: 30m
        annotations:
          summary: "DB-Disk wird voraussichtlich in < 24 h voll"

  - name: warning
    rules:
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[10m])) by (le, uri))
          > 2
        for: 15m

      - alert: WalArchiveBacklog
        expr: pg_wal_archive_pending_count > 10
        for: 5m
```

### Anti-Noise-Regeln

- Nachts (22:00-06:00 Uhr) nur Critical-Alerts an E-Mail; Warnings warten auf den Morgen.
- Alerts werden nach Resolve nochmal bestätigt.
- Bekannte Wartungsfenster werden über `silences` in AlertManager unterdrückt.

## Externes Monitoring

- **UptimeRobot** Free-Tier: HTTP-Probe alle 60 s gegen `/api/v1/health/ready`.
- Bei Ausfall: SMS / E-Mail an Owner.
- Status-Page (Phase 2) öffentlich für Tenants.

## Audit-Log-Monitoring

Auch das Audit-Log selbst wird überwacht:
- Plötzliches Vielfaches an `LOGIN_FAILED`-Events → mögliche Angriffsversuche → Alert.
- Plötzlich viele `ORDER_CANCELLED` aus einem Standort → mögliche Manipulation → Info-Alert.
- `IMPERSONATION_STARTED`-Events → Liste täglich an Owner-Mail.

Diese Auswertungen laufen als geplante SQL-Queries gegen die DB.

## Tracing (Phase 2)

Aktuell nicht im MVP. In Phase 2:
- OpenTelemetry-Instrumentierung.
- Tempo / Jaeger als Backend.
- Auto-Instrumentierung für Spring Boot, JDBC, externe Calls.
- Verlinkung Logs ↔ Traces über `traceId`.

## Performance-Profiling

Bei Bedarf manuell:
- Java Flight Recorder im Prod aktivieren auf konkrete Bestellung.
- Async-Profiler.
- pg_stat_statements für DB-Queries.

Nicht permanent aktiv, da Performance-Overhead.

## Health-Endpoints

| Endpoint | Zweck |
|---|---|
| `/api/v1/health` | Liveness (immer 200, solange Backend läuft) |
| `/api/v1/health/ready` | Readiness — prüft DB-Verbindung, Redis-Verbindung, kritische Adapter-Konnektivität |

Beide ohne Auth, aber rate-limited (max. 10 / s pro IP).

## Run-Books

Pro Critical-Alert ein **Run-Book** (Markdown-Datei in Repo) mit:
- Was bedeutet der Alert?
- Erste Diagnose-Schritte (welche Logs anschauen, welche Queries).
- Mögliche Root-Causes.
- Behandlungs-Schritte.
- Eskalation.

Ablage: `docs/runbooks/`.

Beispiel: `runbooks/backend-down.md`, `runbooks/database-connections-exhausted.md`.
