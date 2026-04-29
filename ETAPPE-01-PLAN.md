# Etappe 01 — Projekt-Skelett

**Status:** In Planung
**Erstellt:** 2026-04-29
**Freigabe durch:** Plattform-Owner (ausstehend)

---

## Aufgabe dieser Etappe

Lauffähiges Spring-Boot-Skelett mit vollständiger Modul-Struktur und automatischer
Modul-Grenz-Verifikation. Kein Business-Code, keine Datenbank-Anbindung.

---

## Was gebaut wird

### Build-System

- `pom.xml` mit:
  - Java 21, Spring Boot **3.5.13**, Spring Modulith (aktuelle zu 3.5 kompatible Version)
  - Maven Enforcer Plugin (Versions-Konflikte verhindern)
  - Maven License Plugin (License-Report)
  - JUnit 5, AssertJ, ArchUnit als Test-Dependencies

### Package-Root und Namensgebung

- **Package-Root:** `dev.veyno.restaurant`
  - `dev.veyno` ist der organisationsweite Namespace, unter dem später weitere
    Projekte koexistieren können.
  - `restaurant` ist der Projekt-Identifier dieser Plattform.
  - Begründung: Entspricht der Vorgabe des Plattform-Owners; spätere Projekte
    (z.B. `dev.veyno.billing`) können sauber daneben leben.
- **Maven Group-ID:** `dev.veyno`
- **Maven Artifact-ID:** `restaurant-app`
- **JAR-Name:** `restaurant-app.jar` (Start-Befehl: `java -jar restaurant-app.jar`)

### Modul-Struktur (15 Module)

Alle 15 Module sind vollwertige Spring-Modulith-Module unter `dev.veyno.restaurant`:

```
dev.veyno.restaurant
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
└── shared          Querschnitts-Utilities (Money, TimeRange, DateRange, Result-Types)
```

**Doku-Korrektur-Hinweis (für spätere Etappe):** `docs/architecture/modulith-structure.md`
und ADR-002 sprechen von „14 Modulen", listen aber `shared` zusätzlich — macht 15.
`shared` ist ein vollwertiges Spring-Modulith-Modul. Die genannten Dokumente sind
in einer späteren Etappe zu korrigieren (kein Scope dieser Etappe).

**Sonderregel für `shared`:**
- Enthält ausschließlich Querschnitts-Wertobjekte und Utilities
- KEINE Geschäftslogik
- Importiert KEINE anderen Module (keine Abhängigkeiten)
- Darf von ALLEN anderen Modulen importiert werden
- KEINE Spring-, JPA- oder sonstige Framework-Annotationen in `shared`

### Pro Modul: interne Skeleton-Struktur

```
dev.veyno.restaurant.<modul>
  ├── api/            public — exponierte Interfaces, DTOs, Events (vorerst leer)
  ├── domain/         public — Aggregate, Entities, Value Objects (vorerst leer)
  └── internal/       package-private — Implementierungen (vorerst leer)
```

Jedes Sub-Package erhält eine `package-info.java` als Package-Anker.
`shared` bekommt kein `internal/`-Sub-Package (Reinheitsregel: alles in `shared` ist public).

### Hauptklasse

- `RestaurantApplication.java` mit `@SpringBootApplication` + `@Modulithic`
- Liegt in `dev.veyno.restaurant` (direkt im Root-Package)

### Konfigurations-Dateien

- `application.yml` — `spring.profiles.default=demo`
- `application-demo.yml` — Skelett-Konfiguration (Adapter-Keys, Flags)
- `application-dev.yml` — Skelett-Konfiguration
- `application-prod.yml` — Skelett-Konfiguration

### Shared-Klassen (ab Tag 1 zwingend)

- `Money.java` — Value Object mit `long cents`, Währungs-Code, grundlegende
  Arithmetik (kein Float/Double)
- `ApplicationClockConfiguration.java` — `Clock`-Bean (systemDefaultZone als Default,
  überschreibbar im Demo-Profil)

### Tests

1. **Modul-Verifikationstest** (`ApplicationModulesTest.java`):
   `ApplicationModules.of(RestaurantApplication.class).verify()` — verifiziert
   alle Modul-Grenzen und `internal`-Zugriffsregeln via Spring Modulith / ArchUnit.

2. **`shared`-Reinheitstest** (`SharedModulePurityTest.java`, ArchUnit):
   Prüft, dass `shared` keine anderen Module importiert und keine
   Spring/JPA/Framework-Annotationen trägt.

3. **Smoke-Test** (`ApplicationSmokeTest.java`):
   Spring-Kontext startet im Demo-Profil ohne Datenbankverbindung.

---

## Quell-Dokumente

| Dokument | Verwendeter Inhalt |
|---|---|
| `docs/architecture/modulith-structure.md` | Modul-Namen, interne Struktur, ArchUnit-Regeln, Verifikationstest |
| `docs/architecture/tech-stack.md` | Stack-Festlegungen, Lizenz-Politik, erlaubte Lizenzen |
| `docs/architecture/standalone-mode.md` | Profil-Konzept, Default-Profil `demo`, Konfigurations-Schema |
| `docs/architecture/overview.md` | Architektur-Prinzipien, `@Modulithic`-Konfiguration |
| `docs/adr/001-hexagonal-architecture.md` | Hexagonal-Muster, domain/-Reinheit (keine Framework-Annotationen) |
| `docs/adr/002-modulith-over-microservices.md` | Modul-Liste, Spring Modulith |
| `docs/adr/003-java-spring.md` | Java 21, Spring Boot, Maven |
| `docs/adr/006-standalone-mode.md` | Profile, Adapter-Konfiguration per `@ConditionalOnProperty` |
| `docs/adr/016-oss-only.md` | Open-Source-only-Lizenz-Politik |

---

## Versionsfestlegungen (diese Etappe)

| Komponente | Version | Begründung |
|---|---|---|
| Java | 21 (LTS) | Doku-Vorgabe, ADR-003 |
| Spring Boot | **3.5.13** | Letzte 3.x-Linie (26. März 2026), langer Support-Horizont; kein Sprung auf 4.x ohne eigenständigen ADR |
| Spring Modulith | kompatibel zu 3.5 | Aktuelle stabile Version zur Spring Boot 3.5-Linie |
| Maven | 3.9+ | Doku-Vorgabe |

**Hinweis Spring Boot 4.x:** Ein Upgrade auf Spring Boot 4.x erfordert einen
separaten ADR und eine eigene Etappe, da es Breaking Changes in Spring Modulith
und dem Ökosystem erwarten lässt.

---

## Akzeptanz-Kriterien

1. `mvn verify` läuft ohne Fehler durch.
2. `ApplicationModulesTest` (Modul-Verifikation via Spring Modulith) ist grün.
3. `SharedModulePurityTest` (ArchUnit) ist grün: `shared` hat keine
   Framework-Annotationen und keine Abhängigkeiten auf andere Module.
4. `ApplicationSmokeTest` bestätigt: Spring-Kontext startet im Demo-Profil.
5. Alle 15 Packages existieren mit `api/`-, `domain/`-, `internal/`-Skelett-Struktur
   (außer `shared`: keine `internal/`-Ebene).
6. `Money`-Klasse verwendet `long`, keinen `float`/`double`.
7. `Clock`-Bean ist konfiguriert; in keiner Klasse wird `LocalDateTime.now()` oder
   `Instant.now()` direkt aufgerufen (verifizierbar via ArchUnit-Regel).
8. Maven License Plugin erzeugt einen License-Report ohne verbotene Lizenzen.

---

## Explizit NICHT in dieser Etappe

- Datenbank-Anbindung (kein Flyway, kein JPA/Hibernate, kein embedded PostgreSQL)
- Entitäten, Repositories, Services mit Geschäftslogik
- REST-Controller, Vaadin-Views, Thymeleaf-Templates
- Spring Security-Konfiguration
- Demo-Daten-Loader
- Fake-Adapter-Implementierungen (Port-Interfaces existieren noch nicht)
- Authentifizierung und Session-Management
- Multi-Tenancy / RLS
- Features aus Phase 2 / Phase 3

---

## Geplante Commits (Reihenfolge)

1. `Füge ETAPPE-01-PLAN.md hinzu`
2. `Initialisiere Maven-Projektstruktur mit Spring Boot 3.5.13 und Spring Modulith`
3. `Lege alle 15 Modul-Packages mit Skelett-Struktur an`
4. `Füge shared-Querschnittsklassen hinzu (Money, Clock)`
5. `Füge Modul-Verifikationstest und shared-Reinheitstest hinzu`
