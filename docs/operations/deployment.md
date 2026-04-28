# Deployment

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

## Deployment-Modelle

Drei Modelle, je nach Zielumgebung:

### Modell A: Standalone (Demo/Eigene Maschine)

Eine JAR. Ein Befehl. Keine externen Dienste.

```
$ java -jar restaurant-app.jar
```

Detail siehe [`../architecture/standalone-mode.md`](../architecture/standalone-mode.md). Dieses Modell ist Teil des MVP.

### Modell B: Docker Compose (Prod, Single-Host)

Empfohlene Produktiv-Bereitstellung im MVP. Alle Komponenten auf einem Server (i9-9900K, 128 GB RAM).

```
docker compose up -d
```

Komponenten siehe unten.

### Modell C: Kubernetes (Phase 3+)

Bei Multi-Host- oder Auto-Scale-Bedarf. **NICHT im MVP**.

## Docker-Compose-Stack (Prod)

```yaml
version: "3.9"

services:

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      - backend

  backend:
    image: ghcr.io/<owner>/restaurant-app:${APP_VERSION}
    restart: unless-stopped
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/restaurant
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_DATASOURCE_PLATFORM_USERNAME: ${PLATFORM_DB_USER}
      SPRING_DATASOURCE_PLATFORM_PASSWORD: ${PLATFORM_DB_PASSWORD}
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      APP_PAYMENT_ADAPTER: stripe
      APP_PAYMENT_STRIPE_SECRET: ${STRIPE_SECRET}
      APP_TSE_ADAPTER: fiskaly
      APP_TSE_FISKALY_API_KEY: ${FISKALY_API_KEY}
      APP_MAIL_ADAPTER: brevo
      APP_MAIL_BREVO_API_KEY: ${BREVO_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_DB: restaurant
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=4GB"
      - "-c"
      - "effective_cache_size=12GB"
      - "-c"
      - "wal_level=replica"
      - "-c"
      - "max_wal_size=8GB"
      - "-c"
      - "archive_mode=on"
      - "-c"
      - "archive_command=/scripts/archive_wal.sh %p %f"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${DB_USER}", "-d", "restaurant"]
      interval: 10s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    command: ["redis-server", "--appendonly", "yes"]

  prometheus:
    image: prom/prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"

  grafana:
    image: grafana/grafana
    restart: unless-stopped
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
    ports:
      - "127.0.0.1:3000:3000"

volumes:
  caddy-data:
  caddy-config:
  postgres-data:
  redis-data:
  prometheus-data:
  grafana-data:
```

## Caddy-Konfiguration

```
{
    email admin@example.com
}

platform.example.com {
    reverse_proxy backend:8080 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        # SSE Timeout-Verlängerung
        flush_interval -1
    }
    encode gzip
}

# Custom-Domains pro Tenant via On-Demand TLS
:443 {
    tls {
        on_demand
    }
    reverse_proxy backend:8080
}
```

**On-Demand TLS** ermöglicht Tenants, eigene Custom-Domains zu nutzen — Caddy holt automatisch ein Zertifikat per Let's Encrypt, wenn ein Domain auf den Server zeigt.

**Sicherheits-Anforderung:** On-Demand TLS verlangt einen `ask`-Endpoint, der validiert, ob ein Domain für TLS-Erstellung berechtigt ist. Backend exponiert `/internal/tls-validation?domain=...` (nur von localhost erreichbar) und prüft `tenant_domains`-Tabelle.

## Image-Build

`Dockerfile`:
```dockerfile
FROM eclipse-temurin:21-jre-jammy

# Non-root user
RUN addgroup --system app && adduser --system --ingroup app app

WORKDIR /app
COPY --chown=app:app target/restaurant-app.jar app.jar

USER app
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl --silent --fail http://localhost:8080/api/v1/health || exit 1

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-XX:+ExitOnOutOfMemoryError", "-jar", "app.jar"]
```

**Image-Größe:** Ziel < 250 MB (Distroless oder Alpine + Temurin).

## CI/CD-Pipeline (GitHub Actions)

Siehe `.github/workflows/`:

### `ci.yml` — auf jeder PR
1. Maven build + tests.
2. ArchUnit/Modulith-Verifizierung.
3. RLS-Policy-Check.
4. OWASP Dependency Check.
5. License Report.
6. JaCoCo Coverage Check.
7. Docker Image bauen, NICHT pushen.

### `release.yml` — auf Tag-Push (`v*.*.*`)
1. Volle CI-Pipeline.
2. Docker Image taggen mit Version.
3. Push nach GitHub Container Registry.
4. GitHub Release mit Changelog erstellen.

### `deploy-prod.yml` — manuell oder auf bestimmten Tag
1. SSH zu Prod-Server.
2. `docker compose pull`.
3. `docker compose up -d --no-deps backend`.
4. Health-Check abwarten (max. 60 s).
5. Bei Fehler: automatischer Rollback auf vorherige Version.

## Deployment-Prozess

### Manueller Standard-Deploy
1. Tag setzen: `git tag v1.4.0 && git push --tags`.
2. CI baut und pusht Image.
3. Auf Server SSHen: `cd /opt/restaurant && docker compose pull && docker compose up -d`.
4. Migrations laufen automatisch beim Start (Flyway).
5. Health-Check via UptimeRobot innerhalb 60 s grün.

### Hotfix-Prozess
1. Branch von `main` ausgehend, Fix entwickeln.
2. PR mit Fast-Track-Label.
3. CI muss durchlaufen.
4. Merge + Tag.
5. Deploy wie oben.

### Rollback
1. Letzte funktionierende Version: `docker compose pull <oldVersion>` und `up -d`.
2. **Bei Migrations-Backout:** Risiko: Flyway erlaubt kein Down-Migrate. Lösung:
   - Migrations sind **strict additive** — alte Tabellen-/Spalten-Strukturen bleiben.
   - Neue Spalten sind zuerst nullable; erst nach Stabilisierung verpflichtend.
   - Bei wirklicher Notwendigkeit: manueller SQL-Rollback (selten).

## Migrations-Strategie

- **Flyway** mit Migrations in `db/migration/V*__name.sql`.
- Migrations laufen automatisch beim Start des Backends.
- **Verbot:** Daten-Mutationen in Schema-Migrations. Daten-Migrations sind separate `V*__data_*.sql` mit Vorsicht.
- Migrations haben Idempotenz wo möglich.
- Vor jedem Deploy: Migration manuell auf Stage geprüft.

## Konfigurations-Management

### Environment-Variablen
- DB-Credentials, API-Keys, sensible Konfiguration.
- Speicherung in `.env`-Datei auf Prod-Server, NICHT im Git.
- `.env.example` im Repo zur Doku.

### `application-prod.yml`
- Nicht-sensitive Konfiguration im Repo.
- Sensitives via `${ENV_VAR}`-Platzhalter.

### Secrets-Rotation
- Manuelle Rotation alle 6 Monate für DB-Passwörter, API-Keys.
- Keine automatisierte Rotation im MVP.

## Server-Vorbereitung (Initial-Setup)

Reproduzierbar via Ansible-Playbook (oder ähnliches Tool, im MVP ggf. dokumentiertes Shell-Skript):

1. Ubuntu LTS Basis-Installation, voll-aktualisiert.
2. SSH-Härtung: kein Passwort-Login, nur Key-Auth, Port 22 oder verschoben.
3. UFW Firewall: 22, 80, 443 — sonst alles dicht.
4. fail2ban für SSH, Caddy.
5. LUKS-Verschlüsselung der Daten-Partition.
6. Docker + Docker Compose installiert.
7. User `restaurant` mit Docker-Gruppe.
8. Verzeichnis-Struktur:
   ```
   /opt/restaurant/
     ├── docker-compose.yml
     ├── .env
     ├── caddy/
     ├── postgres/
     ├── prometheus/
     ├── grafana/
     ├── backups/
     └── logs/
   ```
9. Cron-Jobs: Backup-Job, Audit-Retention-Job, GC-Jobs (im Backend ggf., aber Cron für system-level).
10. systemd-Service für Auto-Start beim Boot:
    ```
    [Unit]
    Description=Restaurant App Stack
    Requires=docker.service
    [Service]
    WorkingDirectory=/opt/restaurant
    ExecStart=/usr/bin/docker compose up
    ExecStop=/usr/bin/docker compose down
    [Install]
    WantedBy=multi-user.target
    ```

## Logging im Prod

- Container-Logs: JSON, an Docker-Default Logging-Driver.
- Backend schreibt strukturiertes JSON via Logback.
- 30 Tage Retention via `--log-opt max-size=...` und `max-file=...`.
- Phase 2: Loki + Promtail für zentrales Logging.

## Backup-Strategie

Detail siehe [`backup-recovery.md`](backup-recovery.md). Kurz:
- `pg_basebackup` initial.
- WAL-Streaming kontinuierlich an zweites RZ, GPG-verschlüsselt.
- Tägliche Snapshots, monatliche Long-Term.

## Kapazitäts-Monitoring

Grafana-Dashboards für:
- CPU-/RAM-/Disk-Auslastung des Hosts.
- DB-Connection-Pool-Auslastung.
- HTTP-Request-Rate, Latenz.
- Bestellungen/Stunde Plattform-weit.
- DB-Query-Latenzen.

Alerts:
- CPU > 80 % für 10 min.
- Disk > 80 % belegt.
- DB-Connection-Pool > 90 % ausgelastet.
- Backend Health-Check schlägt fehl.
- WAL-Streaming hängt > 5 min hinterher.

## Disaster Recovery

Detail in [`backup-recovery.md`](backup-recovery.md).

## Demo-Bereitstellung

Demo-Variante des Backend kann öffentlich zugänglich sein unter `https://demo.platform.example.com`. Konfiguration:
- Spring-Profil `demo`.
- Embedded PostgreSQL auf Container-Volume.
- Auto-Reset alle 24 h via Cron-Job.
- IP-Rate-Limiting (etwas strenger als Prod).
