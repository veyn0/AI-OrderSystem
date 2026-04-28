# Roadmap

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Plattform wird in drei Phasen entwickelt. Aufwand und Reihenfolge sind als Solo-Entwickler-Schätzung gedacht; konkrete Kalenderdaten werden NICHT gesetzt, da die Verfügbarkeit des Entwicklers Hobby-getrieben ist.

## Phasen-Überblick

| Phase | Ziel | Geschäftlicher Status |
|---|---|---|
| **MVP** | Lauffähige Demo aller Kern-Use-Cases mit Stub-Adaptern | Privat, Lernprojekt, Showcase |
| **Phase 2** | Echtkunden-tauglich | Erste zahlende Tenants möglich |
| **Phase 3** | Voll-funktionsfähige Branchen-Lösung | Skalierung, Wettbewerbsfähigkeit |

## MVP

### Ziel
Eine lauffähige `restaurant-app.jar`, die in `demo`-Profil alle MVP-Use-Cases (siehe [`use-cases.md`](domain/use-cases.md)) demonstriert. Adapter sind Stubs, Daten sind in eingebetteter PostgreSQL, Demo-Reset jederzeit möglich.

### Umfang (Features)

#### Plattform und Tenants
- Plattform-Admin-Konten mit drei Rollen (SuperAdmin/SupportAdmin/BillingAdmin).
- Tenant-Lifecycle: Anlegen, Suspendieren, Reaktivieren, Löschen.
- Lizenz-Konfiguration pro Tenant (Datenmodell vorhanden, Billing-Funktionen offen).
- Setup-Token-Mechanismus für Inhaber-Onboarding.

#### Restaurant-Verwaltung
- Standorte, Tische, Tisch-Gruppen.
- Restaurant-Benutzer mit vier Rollen (Inhaber/Geschäftsführer/Standortleiter/Mitarbeiter).
- Geräte-Verwaltung mit Token-Lifecycle.
- Modus-Wechsel auf Geräten.

#### Speisekarten
- Stamm-Speisekarte (tenant-weit).
- Standort-Override (Kacheln deaktivieren, Preise abweichend).
- Kategorien, MenuItems, Varianten, Optionen.
- Allergene und Zusatzstoffe pro MenuItem (Soft-Modus, „auf Anfrage" als Fallback).
- Verfügbarkeits-Zeitfenster pro MenuItem (Frühstück nur 8-11 Uhr etc.).
- Steuersätze pro MenuItem.

#### Bestellung
- Bestellaufnahme am Tablet:
  - Modus `OPEN_TAB` (offene Tisch-Bestellung).
  - Modus `SELF_ORDER_INSTANT` (Selbstbedienung mit Sofort-Zahlung).
- Item-Lifecycle (REQUESTED, ACCEPTED, IN_PREPARATION, READY).
- Order-Lifecycle (DRAFT, CONFIRMED, IN_PROGRESS, READY, SERVED, PAID, CANCELLED).
- Storno mit Begründung; Manager-PIN für Storno nach IN_PROGRESS.
- Steuer-Override mit Permission und Begründung.

#### Online-Bestellung
- Endkunden-Konto pro Tenant (Registrierung, Login, Magic-Link).
- Online-Speisekarte (Thymeleaf+HTMX).
- Bestellung mit Online-Zahlung (über `PaymentPort`-Stub).
- Annahme-Policy mit drei Modi und Bedingungs-Regeln.
- Status-Tracking-Seite für Endkunden mit Live-Updates.

#### Tisch-Bestellung vom Smartphone
- QR-Code pro Tisch.
- Permanenter Token, Rate-Limiting.
- Personal sieht Liste aktiver Tisch-Sessions, kann Tisch blockieren.

#### Kellner-Ruf
- Tisch-QR-Seite mit „Kellner rufen"-Button.
- SSE-Push an Personal-Geräte.

#### Bezahlung am Tisch
- Bargeld-Zahlung mit Beleg-Druck (PDF im virtuellen Postfach).
- Karten-Zahlung über Payment-Stub.
- Geteilte Rechnung pro Position.
- TSE-Signatur-Stub mit deutlich erkennbarem Demo-Hinweis.

#### Reporting
- Tagesumsatz (heute, gestern, letzte 7 Tage, frei wählbar).
- Top-10 Artikel.
- Durchschnittlicher Bonwert.
- Stornorate.
- Heatmap (Bestellungen nach Tageszeit/Wochentag).
- Drill-Down von KPI bis zur einzelnen Bestellung.

#### Audit
- 90-Tage-Audit-Log allgemein.
- 10-Jahre-Audit-Log finanziell.
- Append-only DB-Trigger.
- Audit-View pro Tenant und Plattform-weit.

#### Demo-Funktionen
- Demo-Reset-Endpoint mit Bestätigung.
- Zeit-Sprung-Funktion.
- Virtuelles Postfach (E-Mails, Belege als PDF).
- Plattform-Default-Login-Daten in Demo-Daten.

### Nicht im MVP

- Echtes Payment / TSE / Mail.
- Vollwertiger POS (mit Spezialfunktionen wie Trinkgeld-Berechnung, Wechselgeld-Vorschlag, Tagesabrechnung mit Z-Bon-Druck etc.).
- Eigenständige KDS-UI (KDS-Modus ist im MVP eine vereinfachte Vaadin-View, keine Touch-optimierte UI mit Lautsprecher-Ansage).
- DATEV-Export.
- Treueprogramme.
- Lizenz-Billing (kostenpflichtige Lizenz-Verwaltung).
- Reservierungs-Modul.
- Lager-Modul.
- Fahrer-App.
- Schicht-Management.
- 2FA.
- Native Mobile-App.
- Hot-/Cold-Standby-Setup.
- Penetration-Test.
- Status-Page.
- Offline-Modus für Geräte.

### Aufwand-Schätzung
**Sehr grob, Solo-Entwickler in Hobby-Modus:** Mehrere Monate konzentrierter Arbeit. Genaue Schätzung wird mit dem Implementierungs-Backlog in einem späteren Schritt erarbeitet.

## Phase 2 — Echtkunden-Tauglich

### Ziel
Plattform kann erste reale Restaurant-Kunden onboarden. Alle rechtlichen, sicherheits- und betrieblichen Voraussetzungen für DE-Markt sind erfüllt.

### Umfang (über MVP hinaus)

#### Echte Adapter
- **Echtes Payment:** Stripe oder Mollie integriert, Webhooks, Refunds, PSD2/SCA.
- **Echte TSE:** fiskaly Cloud-TSE integriert (oder Hardware-TSE als zweiter Adapter). Vollständig KassenSichV-konform.
- **Echte Mail:** Brevo (oder Mailjet) integriert. Templates für Bestätigungen, Magic-Links, Passwort-Resets.
- **Echte Push/SMS-Notifikation:** für Statuswechsel-Mitteilungen.
- **Echter Bondrucker:** ESC/POS über Netzwerk an Bondrucker im Restaurant.

#### Vollwertiger POS
- Touch-optimierte UI auf POS-Terminal (Vaadin oder eigene HTML).
- Trinkgeld-Eingabe.
- Wechselgeld-Vorschlag.
- Z-Bon (Tagesabrechnung) pro Schicht/Tag mit TSE-Export.
- Kassenstand-Tracking (Anfangsbestand, Bargeldbewegungen).
- Geteilte Rechnungen mit komplexen Aufteilungen.

#### Eigenständige KDS-UI
- Touch-optimiert, große Buttons.
- Akustische Signale bei neuen Bestellungen.
- Konfigurierbare Stationen (Pizza-Station, Salat-Station etc.) mit Filterung.
- Bons-Layout mit Tisch/Bestelltyp/Zeit-Visualisierung.

#### DATEV-Export
- `AccountingExportPort` mit DATEV-CSV-Generator.
- SKR03/SKR04 konfigurierbar.
- Export-Workflow im Backoffice.

#### Treueprogramme
- Punkte sammeln pro EUR Umsatz.
- Konfigurable Punkt-zu-Rabatt-Konvertierung.
- Endkunden-Konto zeigt Punktestand.
- Restaurant-Konfiguration über Backoffice.

#### Lizenz-Billing
- Monatliche Rechnung pro Tenant (basierend auf Lizenz-Paket).
- Stripe Subscriptions oder ähnliches.
- Mahnwesen bei nicht-bezahlten Rechnungen.
- Suspendierungs-Workflow bei Zahlungsausfall.

#### Reservierungs-Modul
- Endkunden können Tisch reservieren.
- Restaurant verwaltet Reservierungs-Slots.
- Verknüpfung mit `tables`-Aggregat.

#### 2FA
- TOTP-basiert (Google Authenticator, Authy etc.).
- Pflicht-Aktivierung pro Tenant konfigurierbar.
- Recovery-Codes.

#### Native Android-App
- Mobile-Bestelltablet, KDS, POS.
- OAuth2 + PKCE Authentifizierung.
- Architektur ist im MVP vorbereitet.

#### Cold-Standby
- Zweiter Server konfiguriert als PostgreSQL Streaming-Replica.
- Failover-Skript.
- RTO 1 h.

#### Status-Page
- Öffentliche Seite mit Plattform-Status (Up/Down/Degraded).
- Incidents-Historie.
- Geplante Wartungsfenster.

#### Penetration-Test
- Externer Sicherheits-Test vor erster Echtkunden-Aktivierung.
- Findings priorisiert beheben.

#### Datenschutzerklärungs-Generator
- Aus Tenant-Stammdaten + Plattform-Bausteinen → personalisierte Datenschutzerklärung.
- Tenant kann Bausteine personalisieren.
- Versionierung der Datenschutzerklärung pro Tenant.

#### AGB-Generator
- Template-AGB-Texte pro Restaurant-Typ (Pizzeria, Imbiss, Café etc.).
- Personalisierung über Variablen.
- Versionierung.

#### Cloud-Speicher für Speisekarten-Bilder
- Aktuell auf Server-Volume; Phase 2: S3-kompatible Lösung (z.B. MinIO oder Cloud-Provider).
- CDN für Auslieferung.

### Aufwand-Schätzung
Mehrere Monate über MVP hinaus.

## Phase 3 — Voll-funktionsfähig / Skalierung

### Umfang

#### Lager-Modul
- Bestände pro Standort.
- Verbrauchsabbuchung bei Bestellung (Konfiguration: 1 Pizza Margherita = 200g Mehl + 150g Tomaten + ...).
- Wareneingangs-Erfassung.
- Inventur-Workflow.
- Lieferanten-Stammdaten.
- Bestellempfehlungen anhand Mindest-Bestand.

#### Fahrer-App
- Eigene App-Variante für Fahrer.
- Zuteilung von Lieferungen.
- Optimierte Routen (mit OSM oder ähnlicher Karten-Lösung).
- Status-Updates (gestartet, geliefert).
- Trinkgeld-Erfassung.

#### Schicht-Management
- Schichtpläne pro Standort.
- Mitarbeiter-Verfügbarkeit.
- Tausch-Workflow.
- Zeiterfassung am Gerät.
- Lohnabrechnung-Daten-Export.

#### Integrierte EC-Terminals
- Anbindung an EC-Terminal-Hardware (z.B. Concardis, SumUp, Verifone).
- Bargeldlose Zahlung direkt am Tisch über das Bestelltablet.

#### Hot-Standby
- Synchrone Replikation, Load-Balancer.
- Aktiv-Aktiv-Setup.

#### Mehrsprachigkeit
- UI-Sprachen: Deutsch, Englisch, Türkisch, Polnisch, Italienisch.
- Endkunden-Bestellseite mehrsprachig.

#### Barrierefreiheit
- WCAG 2.1 AA für Endkunden-Bestellseite.
- Screen-Reader-Tests.

#### KI-Funktionen (optional, viel später)
- Bestellempfehlungen für Endkunden basierend auf Historie.
- Speisekarten-Texte automatisch generieren (Vorschläge).
- Automatische Allergen-Erkennung aus Beschreibung.

#### Multi-Region-Setup
- Wenn Tenants über DACH hinaus erweitern wollen.
- Regional-Specifics (Steuerrecht in AT/CH unterscheidet sich).

### Aufwand-Schätzung
Mehrere Jahre, optional je nach Geschäftserfolg.

## Feature-Mapping zu Phasen

| Feature | MVP | Phase 2 | Phase 3 |
|---|---|---|---|
| Plattform-Admin | ✓ | | |
| Tenant-Lifecycle | ✓ | | |
| Standorte, Tische | ✓ | | |
| Restaurant-Benutzer | ✓ | | |
| Geräte mit Tokens | ✓ | | |
| Speisekarten Stamm + Standort | ✓ | | |
| Allergene/Zusatzstoffe (Soft-Modus) | ✓ | | |
| Allergene Strict-Modus | | ✓ | |
| Verfügbarkeits-Zeitfenster | ✓ | | |
| Bestellaufnahme Tablet (Open Tab) | ✓ | | |
| Self-Order-Instant | ✓ | | |
| Online-Bestellung | ✓ | | |
| Annahme-Policy | ✓ | | |
| Endkunden-Konto | ✓ | | |
| Tisch-Bestellung Smartphone | ✓ | | |
| Kellner-Ruf | ✓ | | |
| TSE-Stub | ✓ | | |
| Echte TSE | | ✓ | |
| Payment-Stub | ✓ | | |
| Echtes Payment | | ✓ | |
| Mail-Stub (virtuelles Postfach) | ✓ | | |
| Echtes Mail | | ✓ | |
| Reporting Basis | ✓ | | |
| Reporting erweitert | | ✓ | |
| DATEV-Export | | ✓ | |
| Audit-Log | ✓ | | |
| Vollwertiger POS | | ✓ | |
| KDS eigene UI | | ✓ | |
| Treueprogramme | | ✓ | |
| Lizenz-Billing | | ✓ | |
| 2FA | | ✓ | |
| Native Mobile | | ✓ | |
| Reservierungen | | ✓ | |
| Status-Page | | ✓ | |
| Cold-Standby | | ✓ | |
| Penetration-Test | | ✓ | |
| Lager-Modul | | | ✓ |
| Fahrer-App | | | ✓ |
| Schicht-Management | | | ✓ |
| EC-Terminals integriert | | | ✓ |
| Hot-Standby | | | ✓ |
| Mehrsprachigkeit | | | ✓ |
| Barrierefreiheit WCAG AA | | | ✓ |
| Offline-Modus Geräte | | | ✓ |

## Voraussetzungen für Phasen-Übergänge

### MVP → Phase 2 (Echtkunden-Aktivierung)
- TSE-Adapter integriert (echt, KassenSichV-konform).
- Payment-Adapter integriert (echt, PSD2-konform).
- AVV-Workflow umgesetzt.
- Datenschutzerklärung/AGB-Generator oder klar dokumentiertes manuelles Verfahren.
- LMIV-Strict-Modus erzwingbar.
- Penetration-Test bestanden.
- Backup-Konzept erprobt (mind. ein erfolgreicher Test-Restore).
- Status-Page live.
- 2FA verfügbar (Pflicht für Inhaber-Konten).

### Phase 2 → Phase 3
- Mindestens 10 zahlende Tenants stabil im Echtbetrieb.
- Skalierungs-Bedarf erkennbar (z.B. Performance-Engpässe oder Feature-Wünsche, die nur in Phase 3 sinnvoll sind).

## Lessons-Learned-Loops

Nach MVP-Fertigstellung und vor Phase 2:
- Demo öffentlich machen, Feedback sammeln (von potenziellen Tenants oder Branchenvertretern).
- Feature-Backlog anpassen.

Nach Echtkunden-Onboarding (erste 3 Tenants):
- Reale Pain-Points dokumentieren.
- Phase 2 / 3 Priorisierung anpassen.

## Was NICHT auf der Roadmap ist

- Eigene Bezahl-Lösung (kein Payment-Provider werden).
- Eigene Lieferdienst-Plattform (à la Lieferando-Konkurrenz).
- B2B-Marktplatz (Restaurants kaufen voneinander).
- Social-Features.
- Bewertungs-Plattform (im Sinne von Google-Reviews-Ersatz).

Diese sind bewusst außerhalb des Fokus, um Komplexität zu vermeiden.
