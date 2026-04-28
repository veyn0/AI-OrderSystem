# Architektur-Übersicht

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument gibt die High-Level-Architektur in C4-Stil (Level 1: System Context, Level 2: Container).

## Architektur-Prinzipien

Die Plattform folgt vier Kernprinzipien, die in den ADRs detailliert begründet sind:

1. **Hexagonal Architecture (Ports & Adapters)** — externe Abhängigkeiten sind hinter Interfaces gekapselt; Domänenlogik kennt keine Infrastruktur. ([ADR-001](../adr/001-hexagonal-architecture.md))
2. **Modulith** — eine deploybare Einheit, intern in klar getrennte Module zerlegt. ([ADR-002](../adr/002-modulith-over-microservices.md))
3. **Multi-Tenancy by Design** — strikt getrennte Mandanten via Shared DB + Row-Level Security. ([ADR-005](../adr/005-multi-tenancy.md))
4. **Standalone-First** — vollständig lauffähig ohne externe Dienste. ([ADR-006](../adr/006-standalone-mode.md))

## Level 1: System Context

```
                          ┌──────────────────────┐
                          │   Plattform-Admin    │
                          │   (Plattform-Owner,  │
                          │    Support-Mitarbeiter) │
                          └──────────┬───────────┘
                                     │
                                     │  HTTPS (Web)
                                     │
   ┌──────────────┐                  ▼
   │  Restaurant- │           ┌──────────────────────────────┐                    ┌──────────────────┐
   │  Inhaber,    ├─HTTPS────▶│                              │◀───── HTTPS ───────┤  Endkunde        │
   │  Manager,    │  (Web)    │                              │       (Web/PWA)    │  (Online-Bestellung)│
   │  Standortleiter│         │                              │                    └──────────────────┘
   │  Mitarbeiter │           │                              │
   └──────────────┘           │   Restaurantverwaltungs-     │
                              │   Plattform                  │
   ┌──────────────┐           │                              │                    ┌──────────────────────┐
   │  Geräte      │           │                              │                    │  Externe Dienste     │
   │  (Tablet,KDS,├─HTTPS────▶│                              │◀── via Adapter ────┤  (Phase 2+):         │
   │   POS,Fahrer)│  (Token)  │                              │                    │  - Payment           │
   └──────────────┘           │                              │                    │  - TSE               │
                              │                              │                    │  - Mail              │
                              └──────────────────────────────┘                    │  - DATEV             │
                                                                                   └──────────────────────┘
```

**Externe Akteure:**

| Akteur | Interaktion |
|---|---|
| Plattform-Admin | Web-UI für Tenant-Verwaltung |
| Restaurant-Benutzer | Web-UI für Restaurant-Verwaltung, Bestellaufnahme, Reporting |
| Endkunde | Web-UI für Online-Bestellung, Status-Tracking, Konto-Verwaltung |
| Geräte | Token-authentifiziertes API für Bestelltablet, KDS, POS, Fahrer-App, Self-Order |

**Externe Systeme (alle via Ports, im MVP nur Fake-Adapter):**

| System | Port | Adapter MVP | Adapter Prod (Phase 2+) |
|---|---|---|---|
| Payment-Provider | `PaymentPort` | `FakePaymentAdapter` | Stripe / Mollie / Adyen |
| TSE | `TsePort` | `FakeTseAdapter` | fiskaly / Epson / Swissbit |
| Mail | `MailPort` | `LogMailAdapter` | Brevo / Mailjet / SES |
| Push/SMS | `NotificationPort` | `FakeNotificationAdapter` | Twilio / FCM |
| Bondrucker | `ReceiptPrinterPort` | `PdfReceiptPrinterAdapter` | ESC/POS via Netzwerk/USB |
| Buchhaltung | `AccountingExportPort` | (kein MVP-Adapter) | DATEV-Export |

## Level 2: Container

### Demo-Profil (Standalone)

```
┌──────────────────────────────────────────────────────────────────┐
│  Single JAR Process                                              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │   Spring Boot Application                                  │ │
│  │                                                            │ │
│  │   ┌──────────────────────────────────────────────────┐    │ │
│  │   │  Web-Layer                                       │    │ │
│  │   │   - Vaadin Flow (Admin/Backoffice/Geräte-UI)    │    │ │
│  │   │   - Thymeleaf+HTMX (öffentliche Online-Bestellseite, │    │ │
│  │   │     virtuelles Postfach Demo-UI)                │    │ │
│  │   │   - REST-API Controller (/api/v1/*)             │    │ │
│  │   │   - SSE-Endpoints                               │    │ │
│  │   └──────────────────────────────────────────────────┘    │ │
│  │                                                            │ │
│  │   ┌──────────────────────────────────────────────────┐    │ │
│  │   │  Modulith-Module                                 │    │ │
│  │   │   platform / tenant / auth / menu / order /     │    │ │
│  │   │   payment / tse / customer / device /           │    │ │
│  │   │   notification / reporting / api / web /        │    │ │
│  │   │   demo                                           │    │ │
│  │   └──────────────────────────────────────────────────┘    │ │
│  │                                                            │ │
│  │   ┌──────────────────────────────────────────────────┐    │ │
│  │   │  Adapter-Layer (Demo-Profil)                     │    │ │
│  │   │   FakePaymentAdapter / FakeTseAdapter            │    │ │
│  │   │   LogMailAdapter / FakeNotificationAdapter       │    │ │
│  │   │   PdfReceiptPrinterAdapter                       │    │ │
│  │   └──────────────────────────────────────────────────┘    │ │
│  │                                                            │ │
│  │   ┌──────────────────────────────────────────────────┐    │ │
│  │   │  In-Memory Session Store                         │    │ │
│  │   │  (MapSessionRepository)                          │    │ │
│  │   └──────────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │   Embedded PostgreSQL (zonky)                              │ │
│  │   Datenverzeichnis: ./data/embedded-pg                     │ │
│  │   Schema versioniert via Flyway                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Port: 8080 (HTTP, kein TLS im Demo)                             │
└──────────────────────────────────────────────────────────────────┘

Start-Befehl:  java -jar restaurant-app.jar
Default-Profil:  demo
```

### Prod-Profil

```
                                  Internet
                                     │
                                     │ HTTPS
                                     ▼
                          ┌────────────────────┐
                          │  Caddy              │
                          │  (Reverse Proxy,    │
                          │   TLS-Termination,  │
                          │   Rate-Limiting)    │
                          └──────────┬──────────┘
                                     │
                                     │ HTTP intern
                                     ▼
┌──────────────────────────────────────────────────────────────────┐
│  Backend Container                                               │
│  Spring Boot, identische Modulstruktur,                          │
│  ABER: Echte Adapter (Stripe, fiskaly, Brevo, ...)               │
│         OAuth2 Resource Server für native Mobile (Phase 2)       │
└──────────┬─────────────────────────────────────┬─────────────────┘
           │                                     │
           ▼                                     ▼
   ┌──────────────────┐                ┌──────────────────┐
   │  PostgreSQL 16   │                │  Redis           │
   │  (nativ oder     │                │  (Session-Store, │
   │   im Container)  │                │   Cache)         │
   └──────────────────┘                └──────────────────┘
           │
           ▼ (kontinuierliches WAL-Streaming)
   ┌──────────────────┐
   │  Off-Site Backup │
   │  (zweites RZ)    │
   └──────────────────┘

Zusätzlich (nicht im Datenfluss):
   Prometheus + Grafana   (Metriken)
   UptimeRobot (extern)   (Uptime-Monitoring)
```

## Datenfluss-Beispiele

### Beispiel 1: Endkunde bestellt online (Lieferung)

1. Endkunde öffnet `https://<restaurant>.platform.de` → Caddy → Backend.
2. Backend rendert Standort-Speisekarte (Thymeleaf, gefüllt aus DB).
3. Endkunde befüllt Warenkorb (clientseitig in Session).
4. Checkout: Backend ruft `PaymentPort.initiatePayment()`.
5. Endkunde wird zu Provider-Seite geleitet (im Demo: simulierte Seite).
6. Webhook trifft `/api/v1/payments/webhook` → `Payment.SUCCEEDED`.
7. `OrderConfirmedEvent` wird publiziert.
8. `AcceptancePolicyService` evaluiert → setzt Order-Status.
9. SSE-Push an KDS-Geräte des Standorts.
10. Endkunde sieht Bestätigungsseite mit Tracking-Link.

### Beispiel 2: Bestellung am Tisch via Tablet

1. Mitarbeiter loggt sich am Tablet via Geräte-Token + PIN ein.
2. Tablet ruft `/api/v1/floor-plan` ab → Tische des Standorts.
3. Mitarbeiter wählt Tisch → `/api/v1/orders` POST → neue `Order` mit `status=DRAFT`.
4. Items hinzufügen → `/api/v1/orders/{id}/items` POST.
5. „An Küche" → `/api/v1/orders/{id}/transitions/send-to-kitchen` POST → Status-Update + Event.
6. SSE-Push an alle KDS-Geräte des Standorts.
7. Küche bestätigt Items → Order wechselt durch Lifecycle.
8. Abrechnung am Tablet → `/api/v1/orders/{id}/charge` POST → Payment, Bondruck, TSE-Signierung (alle via Ports).

## Architektur-Patterns

### Hexagonal Architecture
- **Domain-Core** kennt nur eigene Entitäten und Ports (Interfaces).
- **Adapters** implementieren Ports und kapseln externe Systeme.
- **Application Services** orchestrieren Domain-Logik und Ports.
- **UI/API-Layer** ist ein weiterer Adapter-Typ ("Driving Adapter").

### Domain-Driven Design (light)
- **Aggregates** wie definiert in [`../domain/domain-model.md`](../domain/domain-model.md).
- Cross-Aggregate-Referenzen nur über IDs.
- **Domain-Events** für Cross-Module-Kommunikation.
- Kein vollständiges DDD-Tactical-Pattern (keine Repositories als reine Abstraktionen, kein Factory-Overkill) — pragmatisch.

### Event-Driven Communication (innerhalb der Anwendung)
- Spring `ApplicationEventPublisher` für Domain-Events.
- Listener mit `@TransactionalEventListener(phase = AFTER_COMMIT)` — Events werden erst nach erfolgreichem Commit verarbeitet.
- **Kein Message-Broker** im MVP — Events sind In-Process. Externalisierung (Kafka/RabbitMQ) wäre Phase 3+ und bei Skalierung.

### CQRS-light
- Reads für Reporting via jOOQ direkt gegen DB-Views, optimiert für Aggregationen.
- Writes via JPA-Aggregates mit eingebauten Geschäftsregeln.
- Keine separate Read-DB.

## Quality Attributes (Architektonisch verankert)

### Sicherheit
- Mandantentrennung doppelt: RLS + Service-Layer-Check.
- Sessions HttpOnly, Secure, SameSite=Lax.
- Argon2id für Passwörter.
- Geräte-Token sicher gehasht.
- Audit-Log append-only.

### Performance
- Datenbank-Queries mit Indizes nach `tenant_id`, `location_id`, häufigen Filterspalten.
- Reporting-Queries via Materialized Views (regenerated nightly) bei großem Volumen, Phase 2.
- SSE statt Polling reduziert Last.
- Connection-Pooling über HikariCP (Spring-Default).

### Testbarkeit
- Hexagonal Architecture macht Domain-Logik trivial unit-testbar (Fake-Adapter im Test).
- Spring Modulith generiert Verifizierungs-Tests gegen Modul-Grenzen.
- Testcontainers für Integration-Tests gegen echte PostgreSQL.

### Wartbarkeit
- Module sind eigenständig, klein, fokussiert.
- ADRs dokumentieren Entscheidungen inkl. Alternativen.
- Inline-Code-Doku via Javadoc auf Service-Layer.

## Bekannte Architektur-Trade-Offs

### Single-Server, kein HA
**Trade-Off:** Einfacher Betrieb vs. Verfügbarkeit.
**Risiko:** Hardware-Ausfall = Downtime.
**Mitigations:** RAID1, Off-Site-Backup, Cold-Standby in Phase 2.

### Online-only Geräte
**Trade-Off:** Architektur-Einfachheit vs. Resilience bei Netzausfall.
**Risiko:** Internet-Ausfall = Stillstand im Restaurant.
**Mitigations:** Idempotenz-fähige API von Tag 1, Offline-Modus nachrüstbar.

### Modulith statt Microservices
**Trade-Off:** Einfache Entwicklung/Deployment vs. unabhängige Skalierung einzelner Komponenten.
**Risiko:** Bei sehr starkem Wachstum müssen Module evtl. extrahiert werden.
**Mitigations:** Spring Modulith erzwingt klare Modul-Grenzen, Extraction wäre möglich.

### Vaadin + Thymeleaf statt SPA-Framework
**Trade-Off:** Java-zentrierte Produktivität für Solo-Entwickler vs. moderne SPA-Ökosystem-Vorteile.
**Risiko:** Vaadin Flow ist Server-Roundtrip-basiert und für sehr interaktive UIs auf schwacher Verbindung suboptimal.
**Mitigations:** Online-Bestellseite (kritisch für Endkunden) verwendet Thymeleaf+HTMX, dort kein Roundtrip-Problem.

### In-Process-Events statt Message-Broker
**Trade-Off:** Kein Operations-Overhead vs. keine Persistenz, keine Cross-Process-Events.
**Risiko:** Bei Server-Crash zwischen Commit und Event-Verarbeitung gehen Events verloren.
**Mitigations:** Spring Modulith bietet Event-Persistenz für kritische Events (Phase 2 erwägen).
