# Risiken

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument listet die identifizierten Risiken zur Plattform — technisch, betrieblich, rechtlich, geschäftlich. Jedes Risiko hat:

- **Schwere** (Kritisch / Hoch / Mittel / Niedrig)
- **Wahrscheinlichkeit** (Hoch / Mittel / Niedrig)
- **Auswirkung**
- **Mitigation** (technisch, prozessual, akzeptiert)
- **Status** (offen, mitigiert, akzeptiert)

## Schwere-Definition

| Schwere | Bedeutung |
|---|---|
| Kritisch | Unmittelbare Gefährdung des Geschäfts, Kunden, Compliance |
| Hoch | Erhebliche Auswirkungen, mit Aufwand behebbar |
| Mittel | Spürbare Auswirkungen, in Phase 2 zu behandeln |
| Niedrig | Akzeptables Restrisiko |

---

## R-001 — Cross-Tenant-Datenleck

**Schwere:** Kritisch
**Wahrscheinlichkeit:** Niedrig (durch Mitigation)
**Auswirkung:** DSGVO-Pflicht-Meldung, reputationaler Totalschaden, mögliche Insolvenz.

**Mitigation (technisch):**
- Defense-in-Depth: Service-Layer + Repository-Filter + RLS.
- ArchUnit erzwingt Modul-Grenzen.
- Pflicht-Tests pro Endpoint mit Cross-Tenant-Versuch.
- Code-Review-Pflicht für Code, der `BYPASSRLS`-Pool verwendet.
- Keine Tenant-übergreifenden Joins außer in dezidierten Plattform-Endpoints.

**Mitigation (prozessual):**
- Externer Pen-Test vor Echtkunden-Go-Live.
- Bug-Bounty erwägen Phase 2.

**Status:** Offen, Mitigation aktiv. Wird über die gesamte Lebensdauer der Plattform überwacht.

---

## R-002 — TSE-Stub im Echtbetrieb

**Schwere:** Kritisch
**Wahrscheinlichkeit:** Niedrig (durch Hard-Block)
**Auswirkung:** Verstoß gegen KassenSichV. Bußgelder bis 25.000 €/Verstoß. Schadensersatzpflicht ggü. Tenant.

**Mitigation:**
- **Hard-Block:** Echtkunden-Onboarding ist technisch erst möglich, wenn ein BSI-zertifizierter TSE-Adapter integriert ist.
- Im Backoffice prominenter „DEMO-Modus, nicht für Echtbetrieb"-Banner, solange Fake-TSE aktiv.
- `is_fake = true` auf jeder Signature → bei Versuch, Beleg auszustellen mit Fake-TSE → klare UI-Warnung.

**Status:** Offen, Hard-Block-Mechanismus ist Teil des MVP-Scopes.

---

## R-003 — Solo-Entwickler-Abwesenheit

**Schwere:** Hoch
**Wahrscheinlichkeit:** Hoch (Krankheit, Urlaub, persönliche Verpflichtungen passieren)
**Auswirkung:** Plattform ist während Abwesenheit ohne Operator. Bei Incident Recovery-Verzögerung.

**Mitigation (MVP):**
- Akzeptiert. Privates Lernprojekt erlaubt diese Annahme.
- Bei Incident: Best-Effort-Reaktion durch Inhaber, ggf. Dienstunterbrechung.
- Externe Uptime-Probe alarmiert Inhaber.
- 99.5 % Verfügbarkeit beinhaltet diese Realität.

**Mitigation (Echtkunden-Go-Live):**
- Schriftliche Run-Books, sodass eine vertraute Person im Notfall einspringen könnte.
- Standby-Operator, wenn finanziell tragbar.

**Status:** Akzeptiert für MVP. Re-Evaluation vor Echtkunden-Go-Live.

---

## R-004 — Internet-Ausfall im Restaurant

**Schwere:** Mittel
**Wahrscheinlichkeit:** Mittel
**Auswirkung:** Restaurant kann keine Bestellungen aufnehmen, abrechnen, KDS funktioniert nicht.

**Mitigation (MVP):**
- Architektur ist von Tag 1 idempotenz-fähig (Idempotency-Key-Header), bereitet Offline-Modus vor.
- Restaurant-Tenant ist informiert, dass das System Online-only ist.

**Mitigation (Phase 3):**
- Offline-Modus für Bestelltablet (lokale Speicherung, Re-Sync bei Verbindung).
- KDS bleibt mit lokalem State funktionsfähig.

**Status:** Akzeptiert für MVP. Erweiterung in Phase 3.

---

## R-005 — Hardware-Ausfall Single-Server

**Schwere:** Hoch
**Wahrscheinlichkeit:** Mittel (Hardware-Lebensdauer 5-7 Jahre, Ausfälle möglich)
**Auswirkung:** Komplett-Ausfall der Plattform. RTO 4 h, dabei Tenants ohne Online-Bestellung.

**Mitigation (MVP):**
- RAID1 schützt vor Einzel-Disk-Ausfall.
- Off-Site-Backup im zweiten RZ → Wiederherstellung möglich.
- Dokumentierter Disaster-Recovery-Prozess.

**Mitigation (Phase 2):**
- Cold-Standby im zweiten RZ → RTO ~1 h.

**Mitigation (Phase 3):**
- Hot-Standby mit synchroner Replikation.

**Status:** Mitigiert für MVP-Niveau. Akzeptiert mit RTO 4 h.

---

## R-006 — Backup-Schlüssel-Verlust

**Schwere:** Kritisch
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Backups sind unbrauchbar — bei DR-Vorfall TOTALVERLUST.

**Mitigation:**
- Backup-Private-Key auf Backup-Host UND offline auf USB-Stick im Tresor.
- Sicherungskopie an mindestens 2 Orten (Tresor + verschlüsselter Cloud-Speicher).
- Test der Wiederherstellbarkeit alle 6 Monate.
- Schriftliche Anleitung zur Schlüssel-Wiederherstellung.

**Status:** Mitigiert.

---

## R-007 — GoBD-Verstoß durch Stub-Betrieb

**Schwere:** Kritisch
**Wahrscheinlichkeit:** Niedrig (durch Hard-Block)
**Auswirkung:** Kasse als „nicht-konform" eingestuft, Schätzung statt Buchführung — finanzieller Schaden für Tenant. Plattform-Owner haftbar.

**Mitigation:**
- Identisch zu R-002.
- Plus: Demo-Belege haben Wasserzeichen „Demo-Modus".
- Onboarding-Workflow erzwingt echte TSE-Konfiguration vor Online-Bestellfunktion.

**Status:** Offen, durch Hard-Block adressiert.

---

## R-008 — DSGVO-Pannen (Datenleck, mangelhafte Auskunft)

**Schwere:** Hoch
**Wahrscheinlichkeit:** Mittel (komplexe Domäne, fehlende Erfahrung)
**Auswirkung:** Bußgelder bis 4 % des weltweiten Jahresumsatzes oder 20 Mio €. Reputational schädlich.

**Mitigation:**
- AVV-Pflicht vor Echtkunden-Go-Live.
- TOMs dokumentiert.
- Auskunfts-/Lösch-Workflow technisch implementiert.
- Externe Datenschutz-Beratung vor Go-Live.
- Datenschutz-Pannen-Meldungs-Prozess vorbereitet.

**Status:** Mitigiert für MVP-Niveau, externe Verifikation erforderlich.

---

## R-009 — Wettbewerber-Vorsprung

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Hoch (es gibt etablierte Anbieter wie Lieferando, Gastronovi etc.)
**Auswirkung:** Schwierigkeiten beim Markteintritt, geringe Tenant-Akquise.

**Mitigation:**
- Kein technischer Schutz möglich.
- Geschäftsstrategie: Differenzierung über Open-Source-Stack, lokale Datenhoheit, faire Preise, Standalone-Betrieb für Datenschutz-bewusste Restaurants.

**Status:** Akzeptiert. Geschäfts- nicht technisches Risiko.

---

## R-010 — Server-Kapazität bei plötzlichem Wachstum

**Schwere:** Mittel
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Performance-Probleme, Ausfälle bei Lastspitzen.

**Mitigation:**
- Hardware ist großzügig dimensioniert (i9, 128 GB RAM) für 50 Tenants × 3 Standorte.
- Kapazitäts-Monitoring (CPU, RAM, DB) alarmiert frühzeitig.
- Architektur erlaubt vertikale Skalierung (mehr RAM, schnellere CPU) ohne Rewrite.
- Bei 80 %+ Auslastung: Hardware-Upgrade oder Architektur-Anpassung (Multi-Server).

**Status:** Mitigiert für MVP-Wachstumshorizont.

---

## R-011 — Domain-Mapping-Missbrauch

**Schwere:** Mittel
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Angreifer richtet ein DNS-A-Record auf unsere Server-IP, lässt Caddy ein Zertifikat ausstellen, betreibt Phishing.

**Mitigation:**
- Caddy `on_demand_tls` mit `ask`-Endpoint, der `tenant_domains`-Tabelle prüft.
- Nur Domains, die im System konfiguriert sind, bekommen Zertifikat.
- Domain-Verifizierung beim Tenant-Onboarding (DNS-TXT-Eintrag oder File-Verification).

**Status:** Mitigiert.

---

## R-012 — Audit-Log-Manipulation

**Schwere:** Hoch
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Verlorene Vorfalls-Aufklärung, GoBD-Verstöße, Vertrauensverlust.

**Mitigation:**
- DB-Trigger blockt UPDATE/DELETE auf Audit-Tabellen.
- DB-User `app` hat nur INSERT-Recht.
- Retention-Job läuft als separater User.
- Backups Off-Site, GPG-verschlüsselt.

**Mitigation (Phase 2):**
- Hash-Chain in Audit-Einträgen — Tampering-Erkennung.

**Status:** Mitigiert für MVP-Niveau.

---

## R-013 — Abmahnung wegen LMIV/Impressum/AGB

**Schwere:** Mittel
**Wahrscheinlichkeit:** Mittel (Abmahn-Industrie ist aktiv)
**Auswirkung:** Anwaltskosten, Unterlassungserklärung, ggf. Vertragsstrafe.

**Mitigation:**
- Onboarding erzwingt vollständige Impressums-Daten.
- Allergen-Pflege wird vor Echtkunden-Go-Live verpflichtend.
- AGB-Vorlage juristisch geprüft (vor Echtkunden-Go-Live).
- Datenschutzerklärung-Generator (Phase 2).

**Status:** Offen, vor Go-Live priorisieren.

---

## R-014 — Bibliotheks-Lizenz-Inkompatibilität

**Schwere:** Mittel
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Versehentliche Verwendung kommerziell-nicht-zulässiger Bibliotheken (z.B. Vaadin Pro Components mit kommerzieller Lizenz statt Vaadin Core).

**Mitigation:**
- License-Report im CI.
- Whitelist erlaubter Lizenzen.
- Vor jedem Major-Update: Lizenz-Review.

**Status:** Mitigiert.

---

## R-015 — Geräte-Token-Diebstahl

**Schwere:** Mittel
**Wahrscheinlichkeit:** Mittel (Tablets in Restaurants sind physisch zugänglich)
**Auswirkung:** Angreifer könnte mit gestohlenem Token Bestellungen platzieren oder einsehen.

**Mitigation:**
- Token-Revocation jederzeit möglich.
- Auto-Lockout nach 7 Tagen Inaktivität.
- PIN-Schutz auf Geräten verhindert direkten Missbrauch.
- Audit-Log macht Anomalien sichtbar.

**Status:** Mitigiert.

---

## R-016 — Domain-Modell-Inkonsistenz bei Race Conditions

**Schwere:** Mittel
**Wahrscheinlichkeit:** Mittel
**Auswirkung:** Bestellung wird zweimal abgerechnet, Items doppelt zugewiesen, Status-Inkonsistenzen.

**Mitigation:**
- Optimistic Locking (`@Version`) auf Order-Aggregat.
- `SELECT ... FOR UPDATE` für kritische Operationen.
- API-Schicht ablehnt Aktionen bei Versions-Konflikt mit klarem Fehler-Code.
- Idempotency-Keys für mutierende Endpoints.

**Status:** Mitigiert.

---

## R-017 — Performance-Degradation bei wachsenden Audit-Tabellen

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Mittel (langfristig)
**Auswirkung:** Audit-Queries werden langsam, Reporting-Performance leidet.

**Mitigation:**
- Indizes auf Audit-Tabellen.
- Retention-Job entfernt allgemeines Audit-Log nach 90 Tagen.
- Phase 2: Partitionierung der `financial_audit_log` nach Jahr.
- Phase 2: Archivierung alter Daten in günstigeren Storage.

**Status:** Akzeptiert für MVP, Phase 2 zu adressieren.

---

## R-018 — Konflikt mit Plattform-Updates

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Mittel (Spring Boot, PostgreSQL Major-Releases)
**Auswirkung:** Breaking Changes erfordern Migrationsaufwand.

**Mitigation:**
- LTS-Politik: Spring Boot Major nach 6 Monaten Stabilität, PostgreSQL bei EOL-Nähe.
- Test-Coverage > 70 % fängt Regressionen.
- ADR pro Major-Update.

**Status:** Akzeptiert.

---

## R-019 — Sicherheits-Lücke in Adapter-Implementierung

**Schwere:** Hoch
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Schwachstelle im Stripe-/fiskaly-/Brevo-Adapter ermöglicht Angriffe.

**Mitigation:**
- Webhook-Signatur-Validierung bei jedem Provider.
- API-Keys nicht im Repo, nur ENV.
- Periodische Sicherheits-Reviews der Adapter.
- Pen-Test deckt diese ab.

**Status:** Mitigiert.

---

## R-020 — Skalierungs-Sackgasse durch Modulith

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Niedrig (für aktuelles Wachstumsziel)
**Auswirkung:** Bei sehr großen Mengen müsste Microservice-Migration erfolgen.

**Mitigation:**
- Spring Modulith erzwingt klare Modul-Grenzen → Extraktion möglich.
- Architektur-Anpassung wäre Phase 4+, nicht im aktuellen Plan.

**Status:** Akzeptiert. Risk-Reward zugunsten der Einfachheit für MVP.

---

## R-021 — Manueller Refund-Workflow im MVP

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Hoch (kommt regelmäßig vor)
**Auswirkung:** Operativer Aufwand für Restaurant-Personal, Fehleranfällig.

**Mitigation:**
- Klares UI mit Hinweis „manuell zurückerstatten" und Audit-Log-Eintrag.
- Phase 2: Automatischer Refund über Payment-Provider-API.

**Status:** Akzeptiert MVP, Phase 2 zu lösen.

---

## R-022 — Fehlende DATEV-Anbindung blockiert manche Restaurants

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Mittel
**Auswirkung:** Restaurants mit Steuerberater-DATEV-Workflow sehen Plattform als unvollständig.

**Mitigation:**
- CSV-Export im MVP (allgemeines Format).
- DATEV-Export Phase 2.

**Status:** Akzeptiert MVP.

---

## R-023 — Datenschutz-Folgenabschätzung übersehen

**Schwere:** Niedrig (DSFA wahrscheinlich nicht erforderlich)
**Wahrscheinlichkeit:** Niedrig
**Auswirkung:** Bei Audit der Aufsichtsbehörde Mängel.

**Mitigation:**
- Externe Datenschutz-Beratung vor Echtkunden-Go-Live.
- DSFA-Bedarf formal geprüft.

**Status:** Offen, vor Go-Live klären.

---

## R-024 — Veraltete Dokumentation

**Schwere:** Niedrig
**Wahrscheinlichkeit:** Mittel
**Auswirkung:** Diskrepanz zwischen Doku und Code, Wartungsprobleme bei späterem Re-Onboarding.

**Mitigation:**
- ADR-Pflicht für Architektur-Änderungen.
- OpenAPI auto-generiert aus Code.
- Periodische Doku-Reviews (z.B. quartalsweise).

**Status:** Akzeptiert. Weicher Prozess.

---

## Risiko-Übersicht

| ID | Risiko | Schwere | Status |
|---|---|---|---|
| R-001 | Cross-Tenant-Datenleck | Kritisch | Offen, mitigiert |
| R-002 | TSE-Stub im Echtbetrieb | Kritisch | Offen, Hard-Block |
| R-006 | Backup-Schlüssel-Verlust | Kritisch | Mitigiert |
| R-007 | GoBD-Verstoß durch Stub | Kritisch | Offen, Hard-Block |
| R-003 | Solo-Entwickler-Abwesenheit | Hoch | Akzeptiert |
| R-005 | Hardware-Ausfall Single-Server | Hoch | Mitigiert |
| R-008 | DSGVO-Pannen | Hoch | Mitigiert, Verifikation nötig |
| R-012 | Audit-Log-Manipulation | Hoch | Mitigiert |
| R-019 | Sicherheits-Lücke in Adapter | Hoch | Mitigiert |
| R-004 | Internet-Ausfall im Restaurant | Mittel | Akzeptiert MVP |
| R-010 | Server-Kapazität bei Wachstum | Mittel | Mitigiert |
| R-011 | Domain-Mapping-Missbrauch | Mittel | Mitigiert |
| R-013 | Abmahnungen LMIV/Impressum/AGB | Mittel | Offen vor Go-Live |
| R-014 | Bibliotheks-Lizenz-Inkompatibilität | Mittel | Mitigiert |
| R-015 | Geräte-Token-Diebstahl | Mittel | Mitigiert |
| R-016 | Race Conditions im Domain | Mittel | Mitigiert |
| R-009 | Wettbewerber-Vorsprung | Niedrig | Akzeptiert |
| R-017 | Audit-Tabellen-Performance | Niedrig | Akzeptiert MVP |
| R-018 | Plattform-Update-Konflikte | Niedrig | Akzeptiert |
| R-020 | Modulith-Skalierungs-Sackgasse | Niedrig | Akzeptiert |
| R-021 | Manueller Refund | Niedrig | Akzeptiert MVP |
| R-022 | Keine DATEV-Anbindung MVP | Niedrig | Akzeptiert MVP |
| R-023 | DSFA übersehen | Niedrig | Offen vor Go-Live |
| R-024 | Veraltete Dokumentation | Niedrig | Akzeptiert |

## Review-Frequenz

- Pre-Echtkunden-Go-Live: vollständige Risiko-Review.
- Danach: quartalsweise.
- Bei jedem Sicherheits-Incident: außerordentliche Review.
