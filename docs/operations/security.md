# Sicherheits-Architektur

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Sicherheit wird nicht „eingebaut", sondern ist Quer­schnittsthema. Dieses Dokument fasst alle Maßnahmen zusammen.

## Bedrohungs-Modell (kurz)

Hauptangriffsflächen:

1. **Internet-Frontend:** Online-Bestellseite, Endkunden-Login, öffentliche API-Endpoints.
2. **Tenant-zu-Tenant:** ein böswilliger oder kompromittierter Tenant darf nicht auf andere Tenants zugreifen.
3. **Geräte:** physisch zugängliche Tablets/POS in Restaurants.
4. **Insider-Bedrohung:** Mitarbeiter mit Zugang zum Plattform-Server.
5. **Backup-Diebstahl:** Backup-Storage als Angriffsziel.

## Sicherheits-Schichten

```
┌─────────────────────────────────────────┐
│  Network/Edge   (Caddy, fail2ban, UFW)  │
├─────────────────────────────────────────┤
│  Authentifizierung (Argon2, Sessions,   │
│  Tokens, OAuth2 in Phase 2)             │
├─────────────────────────────────────────┤
│  Autorisierung   (Permissions, Scopes)  │
├─────────────────────────────────────────┤
│  Daten-Layer     (RLS, Connection-Pool- │
│  Trennung, Audit-Log)                   │
├─────────────────────────────────────────┤
│  Daten at-rest   (LUKS, Backup-GPG)     │
└─────────────────────────────────────────┘
```

## Network-Sicherheit

### Firewall (UFW)

```
Default policy: DENY incoming, ALLOW outgoing
Allowed inbound:
  - Port 22 (SSH) von definierten Admin-IPs
  - Port 80 (HTTP, redirect zu HTTPS via Caddy)
  - Port 443 (HTTPS)
```

### SSH-Härtung

- Public-Key-Only, kein Passwort-Login.
- `PermitRootLogin no`.
- Optional: Port von 22 verschoben (Security through obscurity, geringer Effekt).
- fail2ban gegen Brute-Force.

### Caddy

- Auto-TLS via Let's Encrypt.
- HTTPS-only, HTTP redirected auf HTTPS.
- Strict Transport Security Header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
- Content Security Policy:
  ```
  Content-Security-Policy:
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    connect-src 'self';
    font-src 'self';
  ```
  (Genau ausgestalten je nach UI-Anforderungen — Vaadin braucht ggf. mehr.)
- Weitere Header: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.

### Rate-Limiting

- Caddy-Plugin oder Spring-Filter.
- Pro IP / Endpoint-Klasse, siehe API-Doku.
- DDoS-Mitigation grundlegend abgedeckt; bei größeren Angriffen wäre Cloudflare/CDN nötig (Phase 2 erwägen).

## Authentifizierungs-Sicherheit

Detail siehe [`../architecture/authentication.md`](../architecture/authentication.md).

Highlights:
- Argon2id für alle Passwort- und PIN-Hashes.
- Lockout nach 10 Fehlversuchen, exponentiell verlängert.
- Magic-Link-Tokens 30 min Gültigkeit, einmal-konsumierbar.
- Geräte-Tokens nur als Hash gespeichert.

## Autorisierung / Mandantentrennung

Detail siehe [`../architecture/tenancy.md`](../architecture/tenancy.md).

Drei Schichten Defense-in-Depth:
1. Service-Layer Permission-Check.
2. Repository-Filter nach `tenantId`.
3. PostgreSQL RLS.

## Sensible Daten

| Daten-Typ | Schutz |
|---|---|
| Passwörter | Argon2id-Hash in DB |
| PINs | Argon2id-Hash in DB |
| Geräte-Tokens | Argon2id-Hash in DB |
| Sessions | Cookie HttpOnly+Secure+SameSite=Lax, Session-Daten im Server-Store |
| API-Keys (Stripe, fiskaly) | Environment-Variablen, NICHT im Repo |
| TOTP-Secrets (Phase 2) | Verschlüsselt at-rest mit App-Key |
| Datenbank | LUKS at-rest |
| Backups | GPG-verschlüsselt, separater Key auf Backup-Host |
| Logs | Sensible Daten nicht loggen (siehe Monitoring) |

## Eingabe-Validierung

- Jakarta Bean Validation auf allen DTOs.
- jOOQ und JPA verwenden Prepared Statements → SQL-Injection per se ausgeschlossen.
- HTML-Escaping in Vaadin/Thymeleaf default-aktiv → XSS reduziert.
- Datei-Uploads (Bilder von MenuItems): Content-Type-Validierung, Größenlimits, Malware-Scan (Phase 2).

## SQL-Injection und Co.

- **Kein dynamisches SQL aus User-Input** — ausschließlich Prepared Statements.
- jOOQ mit typsicheren Builders → kein String-Building.
- ORDER BY auf User-Input nur nach Whitelist (sortable Felder definiert).

## XSS / CSRF

- **XSS:** Vaadin/Thymeleaf escapen by default. Manuelles Bypass erfordert explizites Disabling (Code-Review-Pflicht).
- **CSRF:** Spring Security CSRF-Filter aktiv für alle state-mutating Endpoints. Frontend (Vaadin/HTMX) sendet CSRF-Token. API-Endpoints für Geräte verwenden Token-Auth → CSRF nicht relevant.

## Manipulation der Audit-Logs

- DB-Trigger blockt UPDATE/DELETE auf `audit_log` und `financial_audit_log`.
- DB-User `app` hat nur INSERT-Recht auf diese Tabellen.
- Retention-Job läuft als separater DB-User.
- Backup ist Off-Site, GPG-verschlüsselt — nachträgliche Manipulation an Backups erschwert.

## Geräte-Sicherheit

- Geräte sind physisch zugänglich → Token könnte gestohlen werden.
- **Mitigation:**
  - Token kann jederzeit revoked werden.
  - Auto-Revocation nach 7 Tagen Inaktivität.
  - Bei Verdacht: Owner kann Token sofort widerrufen, neuen ausstellen.
  - PIN-Policy auf Geräten verhindert, dass jeder vorbeikommende Gast direkt handeln kann.

## Insider-Bedrohung

Plattform-Owner hat technisch Voll-Zugriff. **Kein technischer Schutz dagegen — diese Bedrohung ist im MVP akzeptiert.** In Phase 2 erwägen:
- Audit-Log-Streaming an externes System.
- Vier-Augen-Prinzip auf kritische Operationen.
- Tampering-Erkennung via Hash-Chains.

## Backup-Sicherheit

Detail siehe [`backup-recovery.md`](backup-recovery.md).

- Backups sind GPG-verschlüsselt mit asymmetrischem Schlüssel.
- Private-Key NICHT auf Hauptserver, nur auf Backup-Host und offline.
- Off-Site-Storage in zweitem RZ — physisch getrennt von Primärserver.

## Incident Response

### Kategorisierung

| Schwere | Beispiele | Reaktion |
|---|---|---|
| P1 (Critical) | Datenleck Cross-Tenant, Login-Bypass, Daten-Verlust | Sofort, Service ggf. abschalten, Tenants informieren |
| P2 (High) | XSS in produktivem Endpoint, Auth-Schwäche | Innerhalb 24 h Fix, Communication |
| P3 (Medium) | DoS-Möglichkeit, kleine Privilege Escalation | Innerhalb Woche Fix |
| P4 (Low) | Theoretisches Risiko ohne praktische Ausnutzbarkeit | Backlog |

### Response-Plan

1. **Detect:** Alert oder externes Hinweis (z.B. Bug-Report).
2. **Contain:** Service ggf. abschalten, Zugriff einschränken.
3. **Eradicate:** Root Cause finden und Fix entwickeln.
4. **Recover:** Service wieder hochfahren, Stichproben-Test.
5. **Communicate:** Tenants informieren, ggf. DSGVO-Pflicht-Meldung an LDA innerhalb 72 h.
6. **Postmortem:** schriftlich dokumentieren.

### DSGVO-Meldepflicht

Bei Datenleck mit personenbezogenen Daten MUSS innerhalb 72 h Meldung an Landesdatenschutzbehörde erfolgen, ggf. an Betroffene (Endkunden, Tenants).

## Vulnerability-Management

### Dependency-Vulnerabilities

- OWASP Dependency Check im CI.
- Dependabot oder Renovate für automatische PRs.
- Critical-CVEs blockieren Build.
- Wöchentliche Sichtung neuer Findings.

### Statische Analyse

- SpotBugs / SonarQube im CI.
- Schwellwerte: 0 Critical, max. 10 Major.

### Dynamische Tests

- E2E-Tests prüfen Auth-Wege.
- (Phase 2) Externer Penetration-Test vor Echtkunden-Go-Live.

## OWASP Top 10 — Checkliste

| Risiko | Mitigation |
|---|---|
| A01 Broken Access Control | Defense-in-Depth (Service, Repo, RLS) |
| A02 Cryptographic Failures | Argon2id, TLS 1.2+, GPG für Backups, LUKS |
| A03 Injection | Prepared Statements (JPA, jOOQ), Input-Validation |
| A04 Insecure Design | Hexagonal-Arch, Bedrohungsmodell explizit |
| A05 Security Misconfiguration | Konfigurations-Review, Docker-Security-Best-Practices |
| A06 Vulnerable Components | OWASP DC, Renovate, Updates wöchentlich |
| A07 Identification & Auth Failures | Brute-Force-Schutz, sichere Sessions, kein "remember me" mit langen Tokens |
| A08 Software & Data Integrity | Audit-Log append-only, Code-Reviews, Reproducible Builds |
| A09 Security Logging & Monitoring | Strukturiertes Logging, Audit-Log, Alerts auf Anomalien |
| A10 SSRF | Webhook-URLs validiert, allowlist für externe Calls |

## Geheimnis-Management

- Secrets ausschließlich via Environment-Variablen.
- `.env`-Datei auf Prod-Server, NICHT im Git.
- `.env.example` im Repo dokumentiert erwartete Variablen.
- (Phase 2) Vault oder ähnliches Tool erwägen.

## Compliance-Anforderungen mit Sicherheits-Bezug

- **GoBD:** Audit-Log 10 Jahre, manipulationsgeschützt.
- **DSGVO:** Datenexport, Löschung, AVV — siehe Compliance-Doku.
- **KassenSichV:** TSE-Anbindung, sicheres Schlüssel-Management der TSE — Verantwortung beim TSE-Provider.

## Wartungs-Sicherheit

- Plattform-Owner-Server-Zugriff via SSH-Key, kein root-Login.
- Updates von Server-OS und Docker monatlich.
- Updates von kritischen Dependencies sofort (Critical-CVE).
- Wartungsfenster werden in Audit-Log dokumentiert.

## Restbedarf für Phase 2

Vor Echtkunden-Go-Live:
- Externer Penetration-Test.
- 2FA für Inhaber/Manager-Konten.
- Verbesserte Session-Hygiene (Aktive Session-Liste pro Konto, mit Revocation).
- Hardware-Security-Module (HSM) für Backup-Schlüssel-Generierung optional.
- Status-Page mit Sicherheits-Incidents-Historie.
