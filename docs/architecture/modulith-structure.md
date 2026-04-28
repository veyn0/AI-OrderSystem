# Modulith-Struktur

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

## Modulgrenzen

Die Anwendung ist in 14 Module aufgeteilt. Jedes Modul ist ein Java-Package unter `de.deinrestaurant`. Spring Modulith erzwingt die Modul-Grenzen via ArchUnit-Verifikation im CI.

```
de.deinrestaurant
├── platform        Plattform-Admin, Tenant-Lifecycle, Plattform-Config
├── tenant          Restaurant, Standort, Tisch, Restaurant-Benutzer
├── auth            Authentifizierung, Sessions, Permission-Checks, Audit-Log
├── menu            Stamm- und Standort-Speisekarten, Artikel, Varianten, Optionen
├── order           Bestell-Lifecycle, OrderItem, Annahme-Policy, Steuer
├── payment         PaymentPort, FakePaymentAdapter
├── tse             TsePort, FakeTseAdapter
├── customer        Endkunden-Konten, Adressen, Bestellhistorie
├── device          Geräte-Konten, Token-Management, Heartbeat, Modi
├── notification    MailPort, NotificationPort, ReceiptPrinterPort + Adapter
├── reporting       KPI-Aggregationen, Drill-Down-Queries
├── web             UI-Layer (Vaadin-Views + Thymeleaf-Templates)
├── api             REST-Controller (versioniert /api/v1/...)
├── demo            DemoDataLoader, Reset-Endpoint, virtuelles Postfach UI
└── shared          Querschnitts-Utilities (Money, TimeRange, Result-Types)
```

## Modul-Detail

### `platform`
**Verantwortlich:** Plattform-Admin-Konten, Tenant-Lifecycle (Create/Suspend/Delete), Plattform-Defaults, Lizenz-Verwaltung.
**Exponiertes API:** `TenantManagementService`, `PlatformAdminService`, `LicenseService`.
**Abhängig von:** `auth` (für Admin-Permissions), `notification` (Welcome-Mails), `shared`.

### `tenant`
**Verantwortlich:** Restaurant-Stammdaten, Standorte, Tische, Tisch-Gruppen, Restaurant-Benutzer (Users), Rollen-Zuweisungen.
**Exponiertes API:** `LocationService`, `TableService`, `UserService`, `RoleAssignmentService`.
**Abhängig von:** `auth`, `notification`, `shared`.

### `auth`
**Verantwortlich:** Login (E-Mail+Passwort), Session-Management, Permission-Auswertung, Audit-Log-Schreiben.
**Exponiertes API:** `AuthenticationService`, `PermissionEvaluator`, `AuditLogger`, `SessionService`.
**Abhängig von:** `shared`. **(Kern-Modul, sollte minimale Abhängigkeiten haben.)**

### `menu`
**Verantwortlich:** Stamm-Speisekarte, Standort-Overrides, Kategorien, MenuItems mit Varianten/Optionen, Allergene, Steuersätze, Verfügbarkeits-Zeitfenster.
**Exponiertes API:** `MasterMenuService`, `LocationMenuService`, `EffectiveMenuService` (gibt für Standort die effektive Speisekarte zurück), `TaxRateResolver`.
**Abhängig von:** `tenant` (für Location-Existenz), `auth`, `shared`.

### `order`
**Verantwortlich:** Bestell-Aggregat (Order, OrderItem), Lifecycle-Transitionen, AcceptancePolicy, Steuer-Berechnung, Domain-Events bei Statuswechsel.
**Exponiertes API:** `OrderService`, `OrderQueryService`, `AcceptancePolicyService`, `WaiterCallService`.
**Abhängig von:** `menu`, `tenant`, `customer` (für Endkunden-Bestellungen), `payment` (synchron für Zahlung), `tse` (synchron für Signierung), `auth`, `shared`.

### `payment`
**Verantwortlich:** `PaymentPort`-Interface und Fake/Real-Adapter.
**Exponiertes API:** `PaymentPort`, `PaymentService` (orchestriert Provider-Aufrufe).
**Abhängig von:** `auth`, `shared`. **(Sollte keine Abhängigkeit auf `order` haben — `order` ruft `payment`, nicht umgekehrt.)**

### `tse`
**Verantwortlich:** `TsePort`-Interface und Fake/Real-Adapter, Verwaltung von TSE-Zustand (Transaktionszähler, Zertifikat).
**Exponiertes API:** `TsePort`, `TseService`.
**Abhängig von:** `auth`, `shared`.

### `customer`
**Verantwortlich:** Endkunden-Konten, Adressen, Bestellhistorie-Sicht, DSGVO-Anonymisierung.
**Exponiertes API:** `CustomerService`, `CustomerAddressService`, `CustomerAnonymizationService`.
**Abhängig von:** `tenant` (Mandanten-Bezug), `auth` (eigene Customer-Auth), `notification`, `shared`.

### `device`
**Verantwortlich:** Geräte-Registrierung, Token-Verwaltung, Modus-Wechsel, Heartbeat, Auto-Lockout.
**Exponiertes API:** `DeviceService`, `DeviceTokenService`, `DeviceHeartbeatService`.
**Abhängig von:** `tenant`, `auth`, `shared`.

### `notification`
**Verantwortlich:** `MailPort`, `NotificationPort` (Push/SMS), `ReceiptPrinterPort` mit Fake/Real-Adaptern. Im Demo-Profil zusätzlich Verwaltung des virtuellen Postfachs.
**Exponiertes API:** `MailPort`, `NotificationPort`, `ReceiptPrinterPort`, `VirtualMailboxService` (nur Demo).
**Abhängig von:** `auth`, `shared`.

### `reporting`
**Verantwortlich:** Aggregations-Queries (KPIs, Drill-Down) auf `order`-Daten. Im MVP Live-Queries gegen DB; in Phase 2 ggf. Materialized Views.
**Exponiertes API:** `ReportingService`, `KpiQueryService`.
**Abhängig von:** `order` (für Daten-Zugriff über Query-Service), `auth`, `shared`.
**Hinweis:** `reporting` greift NICHT direkt auf `order`-Tabellen zu, sondern nur über exponierte Query-Methoden des `order`-Moduls.

### `web`
**Verantwortlich:** UI-Layer mit Vaadin Flow (Backoffice/Geräte) und Thymeleaf+HTMX (Online-Bestellseite, virtuelles Postfach).
**Inhalt:** Views, Components, Templates, Controllers für UI-Routen.
**Abhängig von:** ALLE anderen Module (außer `api` und `demo`). UI ist konzeptuell ein „Driving Adapter".

### `api`
**Verantwortlich:** REST-Controller für externe Aufrufer (Geräte, später Mobile, ggf. Webhooks).
**Inhalt:** Controller, DTO-Mapping, OpenAPI-Annotationen.
**Abhängig von:** alle Service-Module (außer `web` und `demo`).

### `demo`
**Verantwortlich:** Demo-Daten-Loader, Reset-Endpoint, Zeit-Sprung-Funktionalität, virtuelles Postfach-UI.
**Aktivierung:** Nur im Demo-Profil (`@Profile("demo")`).
**Abhängig von:** alle Service-Module für Datenmanipulation.

### `shared`
**Verantwortlich:** Querschnitts-Wertobjekte und Utilities. KEINE Geschäftslogik.
**Inhalt:** `Money`, `PostalAddress`, `TimeRange`, `DateRange`, `Result<T,E>`-Typen, `Tenant-Aware-Annotation`-Marker.
**Abhängig von:** keinem anderen Modul.

## Abhängigkeits-Diagramm

```
                          ┌────────┐    ┌────────┐
                          │  web   │    │  api   │
                          └────┬───┘    └────┬───┘
                               │             │
              ┌────────────────┼─────────────┼────────────────┐
              │                │             │                │
              ▼                ▼             ▼                ▼
        ┌─────────┐     ┌──────────┐   ┌──────────┐    ┌─────────────┐
        │platform │     │  order   │   │ customer │    │ reporting   │
        └────┬────┘     └────┬─────┘   └────┬─────┘    └──────┬──────┘
             │               │              │                 │
             │               ├──────┐       │                 │
             │               │      │       │                 │
             │               ▼      ▼       │                 │
             │           ┌──────┐ ┌─────┐   │                 │
             │           │payment│ │ tse │   │                 │
             │           └───┬──┘ └──┬──┘   │                 │
             │               │       │      │                 │
             ▼               ▼       ▼      ▼                 ▼
       ┌──────────┐     ┌────────┐ ┌──────┐ ┌──────────┐ ┌─────────────┐
       │  tenant  │     │  menu  │ │device│ │notification│ │     auth    │
       └─────┬────┘     └────┬───┘ └───┬──┘ └─────┬────┘ └──────┬──────┘
             │               │         │          │             │
             └───────────────┴─────────┴──────────┴─────────────┘
                                       │
                                       ▼
                                  ┌────────┐
                                  │ shared │
                                  └────────┘
```

## Modul-Kommunikationsregeln

### Erlaubte Kommunikation

1. **Synchroner Service-Call mit erforderlicher Konsistenz:**
   - `order` → `payment` (Zahlung muss synchron erfolgen, bevor Order CONFIRMED wird).
   - `order` → `tse` (Signierung muss synchron beim PAID erfolgen).
   - `order` → `menu` (Effektive Preise und Steuersätze).
   - Service-Calls über exponierte Service-Interfaces des Ziel-Moduls.

2. **Asynchrone Domain-Events (Spring Application Events) für Cross-Module-Reaktionen:**
   - `order` publiziert `OrderConfirmedEvent`.
   - `notification` lauscht und schickt Bestätigungsmail.
   - `reporting` lauscht und aktualisiert evtl. Caches.
   - Events werden mit `@TransactionalEventListener(phase = AFTER_COMMIT)` verarbeitet.

3. **Read-only Queries über exponierte Query-Services:**
   - `reporting` ruft `OrderQueryService.findByLocationAndDateRange(...)` auf, niemals direkt auf Order-Repository zugreifen.

### Verbotene Kommunikation

1. **Direkter Zugriff auf interne Klassen anderer Module.**
   Modul-internes ist alles in `internal`-Sub-Packages. Spring Modulith verifiziert das.

2. **Zirkuläre Abhängigkeiten.**
   Beispiel: `order` darf `payment` aufrufen, aber `payment` darf NICHT `order` aufrufen. Webhook-Verarbeitung in `payment` publiziert ein Event, das `order` konsumiert.

3. **Direkter DB-Zugriff über Modul-Grenzen.**
   `reporting` greift NICHT direkt auf Tabellen des `order`-Moduls zu. Stattdessen exponiert `order` Query-Services.

4. **UI/API-Module untereinander.**
   `web` und `api` rufen sich nicht gegenseitig auf — beide sind parallele UI-Adapter.

## Modul-interne Struktur

Innerhalb jedes Moduls gilt folgende Layered Structure:

```
de.deinrestaurant.<modul>
  ├── api/                # public — exponierte Interfaces, DTOs, Events
  │     ├── service       # Service-Interfaces (z.B. OrderService)
  │     ├── dto           # DTOs (z.B. OrderDTO, OrderItemDTO)
  │     └── events        # Domain-Events (z.B. OrderConfirmedEvent)
  ├── domain/             # public — Aggregate, Entities, Value Objects, Domain-Logik
  └── internal/           # package-private — Implementierungen, Repositories, internes
        ├── persistence    # JPA-Entities, Repositories
        ├── service        # Service-Implementierungen
        └── web            # Falls Modul UI-Komponenten hat (selten)
```

**ArchUnit-Regeln:**
- `internal` ist nicht von außerhalb des Moduls importierbar.
- `api`-Klassen dürfen `internal`-Klassen nicht importieren.
- `domain`-Klassen dürfen keine Spring/JPA-Annotationen tragen (Domain-Reinheit).

## Beispiel: Daten-Fluss bei „Bestellung wird bezahlt"

1. Tablet ruft `POST /api/v1/orders/{id}/charge`.
2. `api.OrderController.charge(...)` ruft `order.OrderService.chargeOrder(...)`.
3. `OrderService` validiert Order-Status, Permissions.
4. `OrderService` ruft `payment.PaymentService.processPayment(...)` synchron auf.
5. `PaymentService` ruft `PaymentPort.initiatePayment(...)` (FakeAdapter im Demo).
6. Bei Erfolg: `OrderService` ruft `tse.TseService.signOrder(...)` synchron auf.
7. `TseService` ruft `TsePort.signTransaction(...)`.
8. `OrderService` setzt Order-Status `PAID`, persistiert.
9. `OrderService` publiziert `OrderPaidEvent`.
10. Listener in `notification`: schickt Bondruck via `ReceiptPrinterPort` (asynchron).
11. Listener in `reporting`: aggregiert KPI-Daten (asynchron).
12. SSE-Push an Statusseite des Endkunden über `web`-Modul.

## Spring Modulith Konfiguration

```
@SpringBootApplication
@Modulithic
public class RestaurantApplication { ... }
```

**Test-Verifikation (in `RestaurantApplicationTests`):**
```java
@Test
void verifyModularity() {
    var modules = ApplicationModules.of(RestaurantApplication.class);
    modules.verify();
}
```

**Generierter Modul-Report:** Spring Modulith erzeugt im Build-Output einen Modul-Report mit allen Abhängigkeiten und Verstößen.
