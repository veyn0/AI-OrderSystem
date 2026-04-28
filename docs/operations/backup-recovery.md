# Backup und Disaster Recovery

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

## Recovery-Ziele

| Maß | MVP | Phase 2 |
|---|---|---|
| RPO (max. Datenverlust) | 5 Minuten | 1 Minute |
| RTO (max. Wiederanlaufzeit) | 4 Stunden | 1 Stunde (Cold-Standby) |
| Backup-Retention täglich | 30 Tage | 30 Tage |
| Backup-Retention monatlich | 12 Monate | 12 Monate |
| Off-Site-Backup-Lokation | zweites RZ | zweites RZ |
| Backup-Verschlüsselung | GPG (asymmetrisch) | GPG |

## Datenbasis

- **Hauptserver:** RAID1 (2× 1 TB NVMe gespiegelt) im primären RZ.
- **Backup-Storage:** 1 TB im zweiten RZ, separates Netzwerk.
- **Verschlüsselung at-rest:** LUKS auf Hauptserver-Datenpartition.

## Backup-Mechanismus

### Kontinuierliches WAL-Streaming

PostgreSQL Write-Ahead-Log wird kontinuierlich gestreamt:

```
postgresql.conf:
  wal_level = replica
  max_wal_size = 8GB
  archive_mode = on
  archive_command = '/scripts/archive_wal.sh "%p" "%f"'
```

`archive_wal.sh`:
```bash
#!/bin/bash
WAL_FILE_PATH="$1"
WAL_FILE_NAME="$2"
BACKUP_DEST="/mnt/backup/wal"

# GPG-verschlüsseln und nach Off-Site
gpg --encrypt --recipient backup@deinrestaurant \
    --output "${BACKUP_DEST}/${WAL_FILE_NAME}.gpg" \
    "${WAL_FILE_PATH}"

# Über SSH ans zweite RZ
rsync -e ssh "${BACKUP_DEST}/${WAL_FILE_NAME}.gpg" backup-host:/backup/wal/

# Lokal aufräumen
rm "${BACKUP_DEST}/${WAL_FILE_NAME}.gpg"

exit 0
```

**Frequenz:** WAL-Segmente werden bei Füllung (default 16 MB) oder via `archive_timeout = 5min` auf jeden Fall alle 5 min archiviert. Damit ist RPO = 5 min garantiert.

### Tägliches Base-Backup

Cron-Job auf Backup-Host:

```bash
# 03:00 Uhr UTC täglich
0 3 * * * /scripts/daily_basebackup.sh
```

`daily_basebackup.sh`:
```bash
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d-%H%M)
BACKUP_DIR="/backup/base/${TIMESTAMP}"

mkdir -p "${BACKUP_DIR}"

# pg_basebackup mit Streaming-WAL
pg_basebackup \
    --host=primary-server \
    --port=5432 \
    --username=replication \
    --pgdata="${BACKUP_DIR}" \
    --format=tar \
    --gzip \
    --compress=9 \
    --wal-method=stream \
    --checkpoint=fast \
    --progress

# Verschlüsselung
for f in ${BACKUP_DIR}/*.tar.gz; do
    gpg --encrypt --recipient backup@deinrestaurant \
        --output "${f}.gpg" "${f}"
    rm "${f}"
done

# Manifest schreiben
echo "Base-Backup von ${TIMESTAMP}" > "${BACKUP_DIR}/MANIFEST"
echo "PostgreSQL Version: $(psql --version)" >> "${BACKUP_DIR}/MANIFEST"
sha256sum ${BACKUP_DIR}/*.gpg >> "${BACKUP_DIR}/MANIFEST"

# Cleanup alter Base-Backups
find /backup/base -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +
```

### Monatliche Long-Term-Backups

Zum 1. jeden Monats wird ein Snapshot in `/backup/longterm/YYYY-MM/` kopiert und 12 Monate aufbewahrt.

## Backup-Inhalte

Was gesichert wird:
- PostgreSQL-Datenbank (komplett, inkl. RLS, Functions, Trigger).
- Anwendungs-Konfiguration (`/opt/restaurant`).
- Caddy-Daten und Zertifikate.
- Hochgeladene Dateien (Bilder von MenuItems): in einem dedizierten Volume oder Bucket.

Was NICHT gesichert wird:
- Container-Images: aus GitHub Container Registry wieder ladbar.
- Logs: 30 Tage lokal, kein Backup.
- Prometheus-/Grafana-Daten: bei Verlust akzeptabel (nicht geschäftskritisch).

## Restore-Prozess

### Szenario 1: Versehentlich gelöschte Daten (Tabelle)

1. Aus letztem Base-Backup eine **separate** PostgreSQL-Instanz wiederherstellen.
2. WAL bis zum gewünschten Zeitpunkt einspielen (Point-in-Time Recovery).
3. Betroffene Tabellen-Inhalte mit `pg_dump` extrahieren und in Live-DB einspielen.
4. Audit-Log-Eintrag „Restore aus Backup vom X bezüglich Y".

### Szenario 2: Komplett-Datenverlust Hauptserver

**Geschätzte Dauer:** 2-4 Stunden (innerhalb RTO 4 h).

1. Neuer Server mit Ubuntu LTS, LUKS, Docker bereitstellen (manuell oder per Ansible).
2. Letztes Base-Backup vom Off-Site-Storage holen.
3. GPG-Entschlüsselung mit Backup-Private-Key.
4. Wiederherstellen:
   ```bash
   tar -xzf base-YYYYMMDD.tar.gz -C /var/lib/postgresql/data
   ```
5. WAL einspielen bis aktueller Stand:
   ```
   recovery.conf:
     restore_command = 'cp /backup/wal/%f.gpg /tmp/%f.gpg && gpg --decrypt /tmp/%f.gpg > %p'
     recovery_target_timeline = 'latest'
   ```
6. PostgreSQL starten, recover.
7. Anwendungs-Container starten.
8. Caddy-Zertifikate werden automatisch neu von Let's Encrypt geholt.
9. DNS ggf. auf neue IP umlegen, falls erforderlich.

### Szenario 3: Korrupte Datenbank (Block-Level)

1. PostgreSQL stoppt, Schaden lokalisieren.
2. Wenn nur einzelne Tabellen betroffen: aus Backup partielles Restore.
3. Wenn großflächig: kompletter Restore wie Szenario 2.

## Backup-Tests

**Pflicht-Frequenz:** monatlicher Test-Restore.

**Prozess:**
1. Auf einem isolierten Test-Server letztes Base-Backup wiederherstellen.
2. WAL bis zu einem zufälligen Zeitpunkt der letzten 24 h einspielen.
3. Verifikation: einige Stichproben-Queries (z.B. „Anzahl Bestellungen", „Tenant-Liste").
4. Sukzess oder Fehler in einem Backup-Test-Log dokumentieren.

**Fehlt der Test 2 Monate hintereinander:** kritische Eskalation, Audit-Eintrag.

## Schlüssel-Management (GPG)

- **Backup-Public-Key:** auf Hauptserver, verschlüsselt damit.
- **Backup-Private-Key:** ausschließlich auf Backup-Host UND offline auf USB-Stick im Tresor.
- **Backup-Private-Key DARF NICHT auf Hauptserver liegen** — sonst wäre Backup-Verschlüsselung sinnlos.

**Verlust-Szenario Private-Key:** alle Backups unbrauchbar. Daher:
- Sicherungskopie des Private-Keys an mindestens 2 Orten (Tresor + verschlüsselter Cloud-Speicher z.B. mit anderem GPG).
- Test der Wiederherstellbarkeit alle 6 Monate.

## Cold-Standby (Phase 2)

In Phase 2: zweiter Server konfiguriert als PostgreSQL Streaming-Replikation-Replica:
- Schreib-Zugriff nur auf Primary.
- Replica wird kontinuierlich aktuell gehalten (Async-Replikation).
- Bei Primary-Ausfall: Failover-Skript promoted Replica zu Primary, DNS umlegen.
- RTO ≈ 1 h.

Dafür reicht der bestehende Server nicht — separater Host nötig.

## Hot-Standby (Phase 3+)

Synchrone Replikation, Load-Balancer vor beiden Servern. Out of Scope für mittelfristig.

## Audit / Compliance

- Backups sind Bestandteil GoBD-konformer Aufbewahrung.
- 10-Jahres-Aufbewahrung der finanziellen Daten ist über Application-Audit-Logs sichergestellt; zusätzliche Backup-Sicherung dient der operativen Sicherheit.
- DSGVO: Backup enthält personenbezogene Daten. Bei Lösch-Anfragen MUSS dokumentiert werden, dass:
  - Daten in der Live-DB anonymisiert sind.
  - Daten in Backups bleiben bis Backup-Ablauf (max. 12 Monate).
  - Bei Restore eines Backups würden anonymisierte Daten wiederhergestellt — daher ist Restore-Prozess so zu gestalten, dass nach Restore eine Re-Anonymisierung ausgeführt wird (Liste der zu anonymisierenden Customer-IDs separat aufbewahrt).

## Notfall-Plan

Bei einem Disaster-Recovery-Vorfall:

1. **Detect:** UptimeRobot alerts → Owner wird benachrichtigt.
2. **Assess:** Owner prüft, was kaputt ist (Hardware, Software, Daten).
3. **Communicate:** Status-Seite (Phase 2) wird auf „incident" gesetzt; Tenants werden via Mail informiert.
4. **Restore:** entsprechendes Szenario abarbeiten.
5. **Verify:** funktionalen Stichproben-Test ausführen.
6. **Communicate Resolution:** Tenants informieren.
7. **Postmortem:** schriftlich dokumentieren, was passiert ist und was geändert wird.

Owner ist der primäre Operator. Im Falle von Owner-Unverfügbarkeit (Krankheit, Urlaub) ist im MVP **kein Stand-In definiert** — das ist ein anerkanntes Risiko (siehe Risiko-Register).
