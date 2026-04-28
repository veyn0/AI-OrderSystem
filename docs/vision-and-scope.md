# Vision und Scope

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

## Vision

Eine modulare, mandantenfähige Restaurantverwaltungs-Software für den deutschen Markt, ausgerichtet auf kleine bis mittlere Betriebe (Pizzerien, Imbisse, kleine Ketten von 1–10 Standorten). Aufgebaut als selbst-gehostete Single-Tenant-Plattform, die als Webapp (zentrale Geschäftsfunktionen) und durch ergänzende native Android-Apps (Geräte-spezifische Anwendungsfälle) zugänglich ist.

## Charakter und Zielsetzung

Das Projekt wird als **privates Lernprojekt eines Solo-Entwicklers** betrieben, mit der bewussten Option auf **kommerzielle Nutzung**. Architektur und Codequalität werden so ausgelegt, dass eine Produktivnutzung mit Echtkunden in Deutschland möglich ist, ohne fundamentale Umbauten. Insbesondere müssen Compliance-Anforderungen (DSGVO, KassenSichV/TSE, LMIV) umsetzbar sein, sobald sie aktiviert werden.

Eine **vollständige Lauffähigkeit als Standalone-Anwendung ohne externe Dienste** ist eine zentrale Anforderung, um Demo-, Test- und Lernbetrieb ohne Infrastrukturkosten zu ermöglichen.

## Zielmarkt

- **Geographie:** Deutschland (DACH-Erweiterung später möglich, nicht im Scope).
- **Kundensegmente:** Einzelrestaurants (Pizzerien, Imbisse, Bistros) sowie kleine Ketten mit bis zu 10 Standorten.
- **Wachstumshorizont (Architekturplanung):** bis zu 50 Restaurants × 3 Standorte × 5 Geräte = ca. 750 simultan aktive Geräte.
- **Wachstumserwartung (realistisch):** unbestimmt, eher unterhalb des Architekturhorizonts.

## Geschäftsmodell

- **Preismodell:** Monatliche Grundgebühr inklusive einer definierten Anzahl Standorte und Geräte.
- **Zusatzgebühren:** gestaffelte Aufpreise pro überzähligem Standort und Gerät.
- **Konkrete Preispunkte:** zu einem späteren Zeitpunkt festzulegen (nicht im Scope dieser Spezifikation).
- **Lizenz-Billing:** **NICHT im MVP**, sondern in Phase 2 als eigene Funktion (Rechnungsstellung an Restaurantkunden).

## Strategischer Architektur-Ansatz

- **Hexagonal Architecture (Ports & Adapters)** als Grundprinzip.
- Jede externe Abhängigkeit (Zahlung, TSE, E-Mail, SMS, Bondrucker, Buchhaltung) ist als **Port (Interface)** definiert mit:
  - **Fake-Adapter** für Demo-/Testbetrieb (im Repo enthalten, ohne externe Dienste lauffähig).
  - **Echtem Adapter** für Produktivbetrieb (vor Echtkunden-Go-Live nachgerüstet).
- Anwendung läuft als **selbstausführbare JAR-Datei** im Demo-Profil ohne weitere Konfiguration.

## Scope: MVP

Im Minimum Viable Product enthalten:

- **Mandanten- und Berechtigungsverwaltung** (Plattform-Admins, Restaurantkunden-Konten, Geräte-Konten).
- **Restaurant-, Standort-, Tisch- und Geräteverwaltung**.
- **Speisekarten- und Artikelverwaltung** (Stamm-Karte mit standortspezifischen Overrides, Varianten, Optionen, Allergene).
- **Bestellaufnahme am Tablet** (zwei Modi: Self-Order und offener Tisch-Vorgang).
- **Einfache Kassenabrechnung** (Bestellung schließen, Zahlart wählen, Beleg-Stub, Steuersatz-Override).
- **Online-Bestellung durch Endkunden** (Vor-Ort über Tisch-QR, Abholung, Lieferung) mit Online-Zahlung über PaymentPort (Fake-Adapter im MVP).
- **Annahme-Policy** für eingehende Online-Bestellungen (Auto/Manuell mit konfigurierbaren Bedingungen).
- **Basis-Reporting** für Restaurantbesitzer (Tagesumsatz, Top-Artikel, Bestellzahlen, Stornorate, Tageszeit-Heatmap, Umsatz pro Standort, Drill-Down).
- **Audit-Logging** mit zwei Retention-Klassen (90 Tage allgemein, 10 Jahre finanziell).
- **TSE-Port** mit Stub-Adapter (echte Integration vor Echtkunden-Go-Live).
- **Demo-Modus**: Embedded PostgreSQL, Fake-Adapter, Demo-Daten, virtuelles Postfach.

## Scope: Phase 2

- Vollwertiger POS / Kasse (über die einfache Abrechnung hinaus).
- Küchenanzeige (KDS) als eigene optimierte UI (im MVP nutzt Küche das Bestelltablet im KDS-Modus).
- Buchhaltungs-Export (DATEV).
- Treueprogramm / Kundenbindung.
- Eigenes Billing für Software-Lizenzen (Rechnungsstellung an Restaurantkunden).
- Tischreservierung.
- Echte Payment-Integration (vor erstem Echtkunden-Go-Live).
- Echte TSE-Integration (vor erstem Echtkunden-Go-Live in DE).
- Echte Mail-Integration.
- ESC/POS-Bondrucker-Adapter.
- Standalone-EC-Terminal-Workflow (manuelle Eingabe).
- Cookie-Banner / Consent (falls Tracking-Funktionen hinzukommen).
- 2FA-Aktivierung für Personen-Konten.
- Cold-Standby für Disaster Recovery.

## Scope: Phase 3 und später

- Lager-/Bestandsverwaltung.
- Lieferdienst-App für eigene Fahrer.
- Mitarbeiter-/Schichtverwaltung.
- Integrierte EC-Terminal-Anbindung (SECPOS/OPI).
- Barcode-Scanner für Lager.
- Kundendisplay (zweiter Monitor).
- Polygon-basierte Lieferzonen.
- Mehrsprachige UI für Endkunden-Speisekarten.
- Hot-Standby / HA-Cluster.
- Penetration-Test (extern).

## Explizit NICHT im Scope (auch langfristig nicht)

- Integrierte Buchhaltung (nur Export, keine eigene Buchführung).
- Lohnabrechnung.
- Kassenbuch in Compliance-Form (entfällt durch TSE-Integration).
- Webseiten-Builder für allgemeine Restaurant-Homepage (nur Online-Bestellseite).
- Integriertes Marketing-Tool / Newsletter.
- Großhandels-Bestellwesen / Wareneingang vom Lieferanten.
- Multi-Mandanten-Konzern-Funktionen (Holdings, konsolidierte Reports über Restaurants hinweg).
- Nicht-DE-Lokalisierung im Kern (architektonisch i18n-fähig, aber nicht ausgeliefert).
- Eigenes Reservierungssystem als alleinige Funktion (nur als Teil der Plattform in Phase 2).

## Architektur-Voraussetzungen, die früh feststehen müssen

Diese Punkte sind **fundamental** und werden später nicht mehr in Frage gestellt:

1. **Multi-Tenancy** mit strikter Mandantentrennung von Tag 1, via Shared DB + Row-Level Security.
2. **Hexagonal Architecture** für jede externe Abhängigkeit.
3. **Standalone-Lauffähigkeit** ohne externe Dienste als first-class Feature.
4. **Java 21 + Spring Boot 3.x** als Backend-Plattform.
5. **PostgreSQL 16+** als einzige Datenbank.
6. **Spring Modulith** mit klar getrennten Modulen.
7. **REST-API mit OpenAPI** als einziges externes API-Format (kein GraphQL).
8. **Vaadin Flow** für Admin/Backoffice/Geräte-UIs, **Thymeleaf+HTMX** für öffentliche Online-Bestellseiten.
9. **Open-Source-only** für alle Abhängigkeiten.

## Erfolgskriterien

Das MVP ist erfolgreich abgeschlossen, wenn:

- Eine einzelne JAR-Datei alle MVP-Use-Cases im Demo-Profil ausführbar macht.
- Ein Restaurantkunde mit zwei Standorten, fünf Tischen je Standort, Speisekarte und drei Geräten vollständig eingerichtet werden kann.
- Endkunden auf der Online-Bestellseite Vor-Ort-, Abhol- und Lieferbestellungen aufgeben können.
- Bestellungen den definierten Lifecycle inkl. Küchen-Bestätigung und Annahme-Policy durchlaufen.
- Reporting alle definierten KPIs darstellt.
- Architektur ist so vorbereitet, dass echte Adapter (Stripe, fiskaly, Brevo) ohne Domänen-Code-Änderungen ergänzt werden können.

## Risiko-Akzeptanz und Bekannte Trade-Offs

- **Verkaufsargument-Risiko:** Etablierte Wettbewerber (Lightspeed, orderbird, Gastronovi, ready2order, Vectron) haben jahrzehntelangen Domänenvorsprung. Differenzierung ist offen und wird nicht als MVP-Ziel definiert.
- **Online-only-Architektur:** Alle Geräte erfordern Internetverbindung. Internet-Ausfall im Restaurant = Stillstand. Architektur wird idempotenz-fähig vorbereitet, sodass spätere Offline-Modus-Nachrüstung möglich ist.
- **Single-Server-Hosting:** Hauptserver ist Single Point of Failure. RTO 4 h im MVP akzeptiert. Cold-Standby in Phase 2.
- **Solo-Entwickler:** Keine Code-Reviews durch Zweite. Erhöhtes Risiko für architektonische Eigenheiten.

## Verbindlichkeit

Dieses Dokument ist **bindend für die Implementierung**. Abweichungen von Vision oder Scope erfordern:

1. Eine schriftliche Begründung.
2. Einen aktualisierten Eintrag in diesem Dokument oder einen neuen ADR.
3. Eine Aktualisierung der Roadmap.

Detail-Entscheidungen unterhalb der Architektur-Ebene (UI-Layouts, konkrete Bibliotheken innerhalb des Stacks, Edge-Case-Verhalten) erfordern keine Vision/Scope-Anpassung.
