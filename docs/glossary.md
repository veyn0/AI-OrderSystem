# Glossar

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Glossar definiert alle Fachbegriffe verbindlich. **Implementierung MUSS diese Begriffe konsistent verwenden** (Klassennamen, Variablen, UI-Texte, API-Felder, Doku).

## Plattform-Ebene

**Plattform**
: Die gesamte vom Plattform-Owner betriebene Software-Instanz. Eine logische Einheit, eine Datenbank, ein Backend-Cluster.

**Plattform-Owner**
: Die Person/Organisation, die die Plattform betreibt (initial: der Solo-Entwickler).

**Admin (Plattform-Admin)**
: Konto auf Plattform-Ebene mit administrativen Rechten. Hat eine Plattform-Rolle (`SuperAdmin`, `SupportAdmin`, `BillingAdmin`). Authentifiziert sich mit E-Mail + Passwort.

**Tenant**
: Synonym für Restaurant aus Plattform-Sicht. Mandantengrenze: alle Daten unterhalb gehören strikt einem Tenant.

**Lizenz / Lizenzpaket**
: Vertragliche Konfiguration eines Tenants — wie viele Standorte/Geräte sind im Grundpreis enthalten, wie hoch die Aufpreise.

## Restaurant-Ebene

**Restaurant**
: Ein Restaurantkunde der Plattform. Entspricht 1:1 einem Tenant. Kann ein einzelnes Lokal sein oder eine Kette mit mehreren Standorten. Auch ein einzelnes Lokal hat konzeptuell ein Restaurant + einen Standort.

**Standort (Location)**
: Physischer Ort, an dem ein Restaurant operiert. Hat eigene Adresse, Öffnungszeiten, Steuersitz, Annahme-Policy, Tische, Geräte, Standort-Speisekarte. Ein Restaurant hat 1..n Standorte.

**Tisch (Table)**
: Physische Sitzeinheit eines Standorts. Hat eine Nummer (standardmäßig inkrementell) und optionale Tags (`drinnen`, `draußen`, `theke`, `terrasse`, beliebig erweiterbar). Hat eine Position im visuellen Tischplan. Hat einen permanent ausgehängten QR-Code für Gast-Smartphones.

**Tisch-Gruppe (Table Group)**
: Temporäre Zusammenlegung mehrerer Tische zu einer Bestelleinheit (z.B. Tisch 7+8 für eine Gruppe von 8 Personen). Wird vom Personal aktiviert/aufgelöst.

**Tischplan (Floor Plan)**
: Visuelle Anordnung der Tische pro Standort. Editierbar via Drag-and-Drop in der Webapp.

## Speisekarten-Domäne

**Stamm-Speisekarte (Master Menu)**
: Auf Restaurant-Ebene gepflegte zentrale Speisekarte. Enthält alle Artikel, Varianten, Optionen, Allergene, Steuersätze. Quelle für alle Standort-Speisekarten.

**Standort-Speisekarte (Location Menu)**
: Erbt von der Stamm-Speisekarte. Kann Artikel deaktivieren, Preise überschreiben, eigene zusätzliche Artikel definieren. Änderungen in der Stamm-Speisekarte propagieren automatisch, außer für überschriebene Eigenschaften.

**Artikel (MenuItem)**
: Verkaufbare Einheit (z.B. Pizza Margherita, Cola 0,5l). Hat Name, Beschreibung, Preis, Steuersatz, Allergene, Kategorie. Kann Varianten und Optionen haben.

**Variante (Variant)**
: Feste Auswahl unter mehreren Größen oder Ausführungen eines Artikels. Genau eine wird gewählt. Beispiel: klein/mittel/groß.

**Option (Option / Modifier)**
: Add-on, das zu einem Artikel hinzugefügt werden kann. Mehrfach wählbar. Beispiel: extra Käse, ohne Zwiebel, scharfe Soße.

**Menü / Kombination (Combo)**
: Bündel aus mehreren Artikeln mit eigenem Preis. Beispiel: Burger + Pommes + Cola. **Phase 2.**

**Kategorie (Category)**
: Gruppierung von Artikeln zur Anzeige (Pizza, Pasta, Getränke, Desserts).

**Allergen / Zusatzstoff**
: Pflichtangabe nach LMIV. Wird pro Artikel gepflegt und auf der Online-Bestellseite vor Kauf angezeigt.

**Steuersatz (Tax Rate)**
: Prozentsatz, der auf Artikel × Bestellkontext angewendet wird. In DE: 7 % für Speisen außer Haus (Lieferung/Abholung), 19 % für Speisen vor Ort und Getränke. Wird automatisch ermittelt, mit manueller Override-Möglichkeit.

**Zeitfenster-Verfügbarkeit**
: Optionale Konfiguration eines Artikels, in welchen Zeiträumen er verfügbar ist (z.B. Frühstück bis 11 Uhr, Mittagskarte 11–14 Uhr).

## Personen-Konten

**Restaurant-Benutzer (User)**
: Person mit Login-Konto, gehört zu **genau einem** Restaurant. Authentifiziert sich mit E-Mail + Passwort. Hat eine oder mehrere Rollen-Zuweisungen mit Scope (TENANT_WIDE oder Liste von Standorten).

**Inhaber (Owner)**
: Restaurant-Rolle mit allen Permissions, scope-frei (TENANT_WIDE). Kann andere Inhaber ernennen, Inhaberschaft übertragen, Restaurant löschen.

**Geschäftsführer**
: Restaurant-Rolle mit allen operativen Permissions, scope-frei. Kann nicht: Restaurant löschen, Inhaberschaft übertragen, Inhaber/Geschäftsführer entfernen.

**Standortleiter**
: Restaurant-Rolle, scoped auf 1..n Standorte. Volle operative Hoheit über die zugewiesenen Standorte.

**Mitarbeiter**
: Restaurant-Rolle, scoped auf 1..n Standorte. Bestellungen aufnehmen, ändern (mit Begründung bei späten Stornos via Manager-PIN), abrechnen.

**Endkunde (Customer)**
: Person, die online bei einem Restaurant bestellt. Konto **pro Restaurant** (nicht plattformweit). Login optional — Gast-Bestellung möglich.

## Geräte-Ebene

**Gerät (Device)**
: Physische Hardware (Tablet, Bondrucker, Lieferdienst-Smartphone, Lager-Scanner, KDS-Display), gehört zu einem Standort. Authentifiziert sich mit langlebigem Bearer-Token (statisch bis manuelle Revocation).

**Geräte-Modus (Device Mode)**
: Anwendungsfunktion eines Geräts. Mögliche Modi:
- `BestellTablet` — Tablet zur Bestellaufnahme am Tisch durch Personal
- `KuechenAnzeige` — KDS-ähnliche Sicht für Küchenpersonal
- `POSTerminal` — Kassenterminal (Phase 2)
- `LagerScanner` — Lager-Erfassungsgerät (Phase 3)
- `FahrerApp` — Lieferdienst-App (Phase 3)
- `SelfOrderTerminal` — Festes Tablet für Self-Order an der Theke/Wand

Ein Gerät kann mehrere zugewiesene Modi haben und zwischen ihnen wechseln. Modus-Wechsel-Bestätigung ist pro Gerät konfigurierbar (`ohne` / `Manager-PIN`).

**Geräte-Token**
: Bearer-Token zur Authentifizierung des Geräts. Wird beim Anlegen einmalig angezeigt, in der DB nur als Hash gespeichert. Statisch gültig bis Revocation. Token-Pairing über manuelle Eingabe oder QR-Code.

**Mitarbeiter-PIN (User PIN)**
: 4–6-stellige PIN pro Restaurant-Benutzer. Genutzt zur Identifikation am Gerät, getrennt vom Login-Passwort. Pro Gerät konfigurierbar als `deaktiviert` / `optional` / `verpflichtend`.

**Manager-PIN**
: PIN eines Benutzers mit höherer Rolle, der am Gerät bestätigt, was ein normaler Mitarbeiter nicht darf (z.B. Storno nach Versand). Audit-Log attribuiert die Aktion auf den Manager mit Hinweis auf den initiierenden Mitarbeiter.

**Heartbeat**
: Periodischer Ping des Geräts an den Server (alle 60 s). Signalisiert Aktivität. Bei mehr als 7 Tagen Inaktivität wird der Token automatisch deaktiviert (manuell wieder aktivierbar).

## Bestell-Domäne

**Bestellung (Order)**
: Zentrale Transaktion. Hat:
- **Bestelltyp:** `VorOrt`, `Abholung`, `Lieferung`
- **Bestellmodus:** `SelfOrderInstant` (sofort bestellen + zahlen) oder `OffenerVorgang` (offen während Gastbesuch, am Ende abgerechnet)
- **Lifecycle-Status** (siehe `order-lifecycle.md`)

**Bestellposition (OrderItem)**
: Eine Zeile in der Bestellung. Enthält: Artikel-Referenz, gewählte Variante, gewählte Optionen, Menge, Preis-Snapshot (zum Bestellzeitpunkt), eigener Status (siehe Order Lifecycle), Hinweistext.

**Preis-Snapshot**
: Beim Anlegen einer Bestellposition wird der zu diesem Zeitpunkt gültige Preis kopiert und unveränderlich in `OrderItem.priceSnapshot` gespeichert. Spätere Preisänderungen am Artikel haben keinen Einfluss auf bestehende Bestellungen.

**Annahme-Policy (Acceptance Policy)**
: Regelwerk pro Standort für eingehende Online-Bestellungen. Drei Modi:
- `AutoAnnahme` — System akzeptiert ohne Personal-Aktion
- `ManuellAnnahme` — Bestellung wird verworfen, wenn Personal sie nicht in einem konfigurierten Zeitfenster aktiv akzeptiert
- `ManuellAblehnung` — Bestellung wird automatisch akzeptiert, wenn Personal sie nicht in einem Zeitfenster ablehnt

Modi können bedingt konfiguriert werden (z.B. AutoAnnahme bei Lieferzeit < 30 min, sonst ManuellAnnahme) und temporär überschrieben („für 3 h nicht automatisch annehmen").

**Auslastungs-Status (Workload)**
: Abgeleitet aus offenen `IN_PROGRESS`-Bestellungen pro Standort, mit Schwellwerten. Manuell überschreibbar durch Personal. Beeinflusst Annahme-Policy und Lieferzeit-Schätzung.

**Steuersatz-Override**
: Manuelle Korrektur des Steuersatzes auf einer Bestellung durch berechtigtes Personal (Permission `ORDER_TAX_OVERRIDE`). Beispiel: Gast hat zur Lieferung bestellt (7 %), entscheidet sich aber, vor Ort zu essen (19 %).

**Storno (Cancellation)**
: Aufhebung einer Bestellung oder Bestellposition. Vor Versand an Küche durch jeden mit `ORDER_CANCEL_PRE_KITCHEN`. Nach `IN_PROGRESS` nur durch Permission `ORDER_CANCEL_POST_KITCHEN` mit Pflichtbegründung; bei Mitarbeitern via Manager-PIN.

**Refund**
: Rückerstattung einer bezahlten Bestellung. Im MVP **manuell durch Personal** (Bargeld zurück oder POS-Korrektur). Automatischer Refund über Payment-Provider in Phase 2.

**Kellner-Ruf (Waiter Call)**
: Aktion eines Endkunden über das Tisch-QR, der das Personal an seinen Tisch ruft. Mit Rate-Limiting (max. 1× pro 3 min pro Tisch). Personal sieht aktive Rufe in einer Liste und kann Tische temporär blocken.

## Externe Schnittstellen (Ports)

**Port**
: Interface in der Anwendung, das eine externe Abhängigkeit abstrahiert. Die Domänenlogik kennt nur den Port, nicht den konkreten Adapter.

**Adapter**
: Konkrete Implementierung eines Ports.

**Fake-Adapter**
: Adapter, der einen externen Dienst simuliert, ohne ihn aufzurufen. Im Demo- und Dev-Profil aktiv. Beispiele:
- `FakePaymentAdapter` — simuliert Zahlung, 95 % Erfolg
- `FakeTseAdapter` — generiert plausibel aussehende Signaturen
- `LogMailAdapter` — schreibt E-Mails ins virtuelle Postfach
- `FakeNotificationAdapter` — simuliert Push/SMS
- `PdfReceiptPrinterAdapter` — generiert PDFs statt Bondruck

**Echter Adapter (Real Adapter)**
: Adapter, der einen echten externen Dienst aufruft. Im Prod-Profil aktiv. Wird vor Echtkunden-Go-Live entwickelt.

**Virtuelles Postfach**
: Demo-UI im Demo-Profil, in der alle simulierten E-Mails, Push-Nachrichten und PDF-Belege einsehbar sind.

## Operations und Architektur

**Modulith**
: Architekturstil — Single Deployment-Einheit mit klar getrennten internen Modulen. Spring Modulith erzwingt Modulgrenzen via ArchUnit-Tests.

**Spring Profile**
: Spring-Boot-Mechanismus zur Aktivierung unterschiedlicher Konfigurationen. Drei Profile:
- `demo` (Default) — Standalone, alles in-Process
- `dev` — Lokale Entwicklung mit Container-Services
- `prod` — Produktivbetrieb

**RLS (Row Level Security)**
: PostgreSQL-Feature, das Zugriffsregeln auf Zeilenebene erzwingt. Wird zur Mandantentrennung verwendet.

**ADR (Architecture Decision Record)**
: Markdown-Dokument, das eine Architekturentscheidung dokumentiert: Status, Kontext, Entscheidung, Alternativen, Konsequenzen.

**SSE (Server-Sent Events)**
: HTTP-basiertes Protokoll für Server → Client Streaming. Verwendet für Live-Updates (z.B. neue Bestellung in Küchen-Sicht).

**Idempotenz**
: Eigenschaft eines API-Calls, dass mehrfaches Ausführen denselben Effekt hat wie einmaliges Ausführen. Wird über client-generierte Request-IDs erreicht. Voraussetzung für späteren Offline-Modus.

## Compliance-Begriffe

**DSGVO**
: Datenschutz-Grundverordnung. Regelt Verarbeitung personenbezogener Daten in der EU.

**AVV (Auftragsverarbeitungsvertrag)**
: Vertrag zwischen Restaurant (Verantwortlicher) und Plattform-Owner (Auftragsverarbeiter) über die Verarbeitung der Endkundendaten. Pflicht vor Echtkunden-Go-Live.

**GoBD**
: Grundsätze zur ordnungsmäßigen Führung und Aufbewahrung von Büchern. 10-Jahres-Aufbewahrung für steuerrelevante Daten.

**KassenSichV / TSE**
: Kassensicherungsverordnung. Verlangt eine zertifizierte Technische Sicherheitseinrichtung (TSE) für jede elektronische Aufzeichnung von Bargeschäften in DE seit 2020. Plattform integriert TSE über `TsePort`.

**LMIV**
: Lebensmittelinformations-Verordnung (EU). Schreibt Allergen- und Zusatzstoff-Angaben vor Verkauf vor.

## Konventionen für die Verwendung

- **Englische Klassennamen, deutsche Domänen-Begriffe in der UI:** Klassen heißen `Order`, `MenuItem`, `Tenant`, `Location`. UI zeigt „Bestellung", „Artikel", „Restaurant", „Standort".
- **Ein Begriff = ein Konzept:** Wenn in diesem Glossar `Standort` definiert ist, dann **NICHT** synonym `Filiale`, `Niederlassung`, `Branch`, `Outlet` verwenden. Auch nicht in Variablen oder UI.
- **Unklare Begriffe in Code-Reviews:** Wer beim Lesen eines PR einen Begriff nicht im Glossar findet, eskaliert das. Dann entweder neuer Glossar-Eintrag oder bestehender Begriff übernehmen.
