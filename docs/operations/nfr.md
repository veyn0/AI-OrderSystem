# Nicht-funktionale Anforderungen

> **Status:** Accepted (MVP-Scope)
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument definiert messbare Qualitätsanforderungen. Jede NFR hat ein **Ziel**, eine **Mess-Methode** und eine **Verbindlichkeit** (MVP / Phase 2).

## Performance

### Antwortzeiten

| Operation | P50 | P95 | P99 | Verbindlichkeit |
|---|---|---|---|---|
| Bestelltablet UI-Aktion (Item hinzufügen, Status-Update) | < 100 ms | < 300 ms | < 500 ms | MVP |
| Bestellung an Küche senden (Roundtrip + SSE-Push) | < 500 ms | < 2 s | < 3 s | MVP |
| Online-Bestellseite Initial-Load | < 800 ms | < 2 s | < 3 s | MVP |
| Reporting-Dashboard Initial-Load | < 1 s | < 3 s | < 5 s | MVP |
| API-Endpoint mit DB-Read | < 50 ms | < 200 ms | < 500 ms | MVP |
| API-Endpoint mit DB-Write (mit Audit-Log) | < 100 ms | < 400 ms | < 800 ms | MVP |
| SSE-Event-Latenz (Domain-Event → Client) | < 200 ms | < 500 ms | < 1 s | MVP |

**Mess-Methode:** Spring Boot Actuator Micrometer-Histogramme, exportiert nach Prometheus, visualisiert in Grafana mit P50/P95/P99-Linien.

**Bei Überschreitung:** Alert über Prometheus AlertManager.

### Durchsatz

| Maß | Ziel | Verbindlichkeit |
|---|---|---|
| Bestellungen / Stunde Plattform-weit | 5.000 (Spitze) | MVP |
| Bestellungen / Stunde pro Standort | 200 (Spitze) | MVP |
| HTTP-Requests / s Plattform-weit | 500 | MVP |
| Simultane SSE-Verbindungen | 1.000 | MVP |
| Simultane Web-Sessions | 2.000 | MVP |

**Hinweis:** Der praktische Last-Spitze-Bereich ist bei realistischer Tenant-Anzahl (initial < 50 Restaurants) deutlich kleiner. Architektur muss aber für Wachstum auf Plattform-Ebene gerüstet sein.

### Datenbank-Performance

| Maß | Ziel |
|---|---|
| 95 % der Queries < 50 ms | MVP |
| Reporting-Aggregations-Queries < 3 s | MVP |
| Connection-Pool: max. 80 % Auslastung in Spitze | MVP |

## Verfügbarkeit

| Maß | Ziel | Verbindlichkeit |
|---|---|---|
| Verfügbarkeit (Echtkunden-Go-Live) | 99.5 % monatlich (= max. ~3.6 h Downtime/Monat) | Pre-Echtkunden |
| Verfügbarkeit Phase 2 | 99.9 % monatlich | Phase 2 |
| Geplante Wartungsfenster | werktags 04:00-05:00 Uhr, max. 1× pro Monat, max. 30 min | Pre-Echtkunden |

**Mess-Methode:** UptimeRobot externer Probe, prüft `/api/v1/health/ready` alle 60 s.

**Verfügbarkeit zählt nur Service-Zeit:** Wartungsfenster werden in der Berechnung **nicht** ausgenommen. 30 min/Monat geplante Downtime ist Teil der 0.5 % Toleranz.

## Recovery-Anforderungen

| Maß | Ziel | Verbindlichkeit |
|---|---|---|
| RPO (Recovery Point Objective, max. Datenverlust) | 5 Minuten | MVP |
| RTO (Recovery Time Objective, max. Wiederanlaufzeit) | 4 Stunden | MVP |
| RTO Phase 2 | 1 Stunde (Cold-Standby) | Phase 2 |

Konkrete Backup-Strategie siehe [`backup-recovery.md`](backup-recovery.md).

## Skalierung / Wachstumshorizont

Architektur ist ausgelegt für:
- 50 Restaurants (Tenants)
- 3 Standorte / Restaurant
- 5 aktive Geräte / Standort
- → 750 simultane Geräte
- → ~5.000 Bestellungen / h plattformweit (Spitze)

Darüber hinaus erfordert es Architektur-Anpassungen:
- Multi-Server-Setup mit verteiltem Session-Store.
- SSE-Pub/Sub via Redis.
- Read-Replicas für Reporting-Queries.

Diese sind nicht Teil des MVP-Plans, aber durch die Architektur nicht ausgeschlossen (siehe [`overview.md`](../architecture/overview.md), Trade-Offs).

## Datenintegrität

| Anforderung | Verbindlichkeit |
|---|---|
| Mandantentrennung: 0 Cross-Tenant-Reads/Writes | MVP, **kritisch** |
| Order-Lifecycle-Konsistenz: keine ungültigen Transitionen | MVP, **kritisch** |
| Audit-Log: append-only, manipulationsgeschützt auf DB-Ebene | MVP, **kritisch** |
| Geld-Berechnungen: 0 Rundungsfehler durch konsequente Cent-Arithmetik | MVP, **kritisch** |
| Optimistic Locking auf veränderbaren Aggregaten | MVP |

**Test-Anforderung:** Pflicht-Tests pro Punkt, im CI durchsetzend. PR ohne entsprechende Tests darf nicht gemerged werden.

## Sicherheit

| Anforderung | Verbindlichkeit |
|---|---|
| TLS 1.2+ in Prod | MVP |
| Argon2id für alle Passwort-/PIN-Hashes | MVP |
| HttpOnly+Secure+SameSite=Lax Cookies | MVP |
| Sessions invalidiert bei Rolle-Änderung, Logout | MVP |
| Rate-Limiting auf Login, Passwort-Reset, Magic-Link | MVP |
| Brute-Force-Schutz mit progressivem Lockout | MVP |
| LUKS Festplatten-Verschlüsselung Server | MVP |
| Off-Site-Backups GPG-verschlüsselt | MVP |
| Dependency-Vulnerability-Scan (OWASP Dependency Check) im CI | MVP |
| Statische Code-Analyse (z.B. SpotBugs, SonarQube) im CI | MVP |
| 2FA für Restaurant-Inhaber/Manager | Phase 2 |
| Penetration-Test extern | Phase 2 |

Detail siehe [`security.md`](security.md).

## Datenschutz / DSGVO

| Anforderung | Verbindlichkeit |
|---|---|
| Auskunfts-Anfrage-Erfüllung max. 30 Tage | Pre-Echtkunden |
| Lösch-/Anonymisierungs-Anfrage max. 30 Tage | Pre-Echtkunden |
| Datenexport Endkunde als JSON+CSV | Pre-Echtkunden |
| AVV-Vertrag mit jedem Tenant abgeschlossen | Pre-Echtkunden |
| Datenschutzerklärungs-Generator pro Tenant | Pre-Echtkunden |

Detail siehe [`../compliance/gdpr.md`](../compliance/gdpr.md).

## Compliance (DE-spezifisch)

| Anforderung | Verbindlichkeit |
|---|---|
| GoBD: 10 Jahre Retention finanzieller Daten | Pre-Echtkunden |
| KassenSichV/TSE: zertifizierte TSE-Anbindung | Pre-Echtkunden |
| LMIV: Allergen-Pflichtangaben pro Artikel im Online-Verkauf | MVP |
| Impressum, AGB, Datenschutzerklärung pro Tenant | MVP |

Detail siehe [`../compliance/`](../compliance/).

## Wartbarkeit

### Code-Qualität

| Maß | Ziel |
|---|---|
| Test-Coverage (Unit + Integration) | > 70 % auf Service-Layer |
| Mutationstest-Score (Phase 2) | > 60 % |
| Statische Analyse: 0 kritische Findings, max. 10 Major | MVP |
| Modulgrenzen-Verletzungen (ArchUnit) | 0 |

### Build und Release

| Maß | Ziel |
|---|---|
| CI-Build-Zeit (PR) | < 5 min |
| Voller Test-Lauf (CI) | < 15 min |
| Deployment-Zeit (von Merge zu Prod) | < 30 min |
| Rollback-Zeit | < 5 min |

### Doku

- ADRs für alle Architektur-Entscheidungen.
- Glossar gepflegt.
- OpenAPI-Doku auto-generiert.
- Inline-Javadoc auf allen public Service-Methoden.
- README im Repo mit Onboarding-Anleitung (Standalone und Dev).

## Bedienbarkeit / UX

(weniger streng messbar, aber dokumentiert)

| Anforderung | Verbindlichkeit |
|---|---|
| Bestelltablet: Komplett-Bestellung in < 30 s pro Bestellung möglich | MVP |
| Endkunde: Online-Bestellung in < 2 min vom Öffnen bis Zahlung möglich | MVP |
| Restaurant-Onboarding: erste Bestellung in < 60 min einrichtbar | MVP |
| Mobile-First für Endkunden-Bestellseite: Lighthouse Mobile Score > 80 | MVP |
| Barrierefreiheit (WCAG 2.1 AA) für Endkunden-Bestellseite | Phase 2 |
| Mehrsprachigkeit der UI | Phase 3 |

## Browser-/Geräte-Kompatibilität

| Plattform | Min. Version |
|---|---|
| Chrome / Edge / Brave | aktuelle - 2 Major |
| Firefox | aktuelle - 2 Major |
| Safari | aktuelle - 2 Major (iOS Safari für Endkunden-Mobil) |
| Android (PWA) | API 26 (Android 8.0) |
| iOS (Safari) | iOS 14+ |
| Internet Explorer | NICHT unterstützt |

## Logging und Monitoring

| Anforderung | Verbindlichkeit |
|---|---|
| Strukturierte JSON-Logs | MVP |
| Korrelations-IDs (Trace-Header durchgereicht) | MVP |
| 30 Tage Log-Retention | MVP |
| Prometheus-Metriken (Spring Actuator) | MVP |
| Grafana-Dashboards für Top-KPIs | MVP |
| Alerts auf SLO-Verletzungen | MVP |
| Externe Uptime-Probe | MVP |
| Log-Forwarding zu zentralem System (Loki/Elasticsearch) | Phase 2 |

Detail siehe [`monitoring.md`](monitoring.md).

## Lizenz / Open-Source

- Alle Abhängigkeiten haben Open-Source-Lizenz aus erlaubter Liste (Apache 2.0, MIT, BSD, EPL, LGPL, MPL).
- Keine Abhängigkeit von kommerziellen Bibliotheken oder „Free for Non-Commercial".
- License-Report wird im Build erstellt und als CI-Artefakt gespeichert.

## Cross-Cutting NFR-Verifikation im CI

Pflicht-Checks im CI, die Builds blockieren:

1. ArchUnit / Spring Modulith Modul-Grenzen.
2. RLS-Policy-Existenz auf neuen Tabellen.
3. OWASP Dependency Check (kritische CVEs blockieren).
4. License-Report (verbotene Lizenzen blockieren).
5. SpotBugs / Statische Analyse mit Schwellwerten.
6. Test-Coverage-Schwellen.
7. OpenAPI-Diff-Check (Breaking Changes erfordern Major-Version-Bump).
8. Contract-Tests gegen Fake-Adapter.
