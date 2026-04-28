# Tech-Stack

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Verbindlicher Technologie-Stack der Plattform. Abweichungen erfordern einen ADR.

## Backend

| Bereich | Wahl | Version | Lizenz |
|---|---|---|---|
| Sprache | Java | 21 (LTS) | GPLv2+CE |
| Framework | Spring Boot | 3.x | Apache 2.0 |
| Modulstruktur | Spring Modulith | aktuelle stabile | Apache 2.0 |
| Build & Dependency | Maven | 3.9+ | Apache 2.0 |
| Datenbank-Server | PostgreSQL | 16+ | PostgreSQL License |
| Embedded DB (Demo) | zonky-test/embedded-postgres | aktuelle | Apache 2.0 |
| ORM | Spring Data JPA + Hibernate | aktuell mit Spring Boot | LGPL 2.1+ |
| Komplexe Queries | jOOQ | OSS-Edition | Apache 2.0 |
| DB-Migrationen | Flyway | OSS-Edition | Apache 2.0 |
| Connection-Pool | HikariCP | aktuell mit Spring | Apache 2.0 |
| Sessions | Spring Session | aktuell | Apache 2.0 |
| Security | Spring Security | aktuell | Apache 2.0 |
| OpenAPI | springdoc-openapi | aktuell | Apache 2.0 |
| Logging | Logback + JSON-Encoder | aktuell | EPL 1.0 / LGPL |
| Metriken | Spring Boot Actuator + Micrometer + Prometheus-Client | aktuell | Apache 2.0 |
| Validierung | Jakarta Bean Validation (Hibernate Validator) | aktuell | Apache 2.0 |
| JSON | Jackson | aktuell | Apache 2.0 |
| Passwort-Hashing | Argon2 (Spring Security) | aktuell | Apache 2.0 |
| QR-Code | ZXing Core | aktuell | Apache 2.0 |
| PDF-Generierung | Apache PDFBox | aktuell | Apache 2.0 |
| HTTP-Client | Spring `RestClient` / `WebClient` | aktuell | Apache 2.0 |

## Frontend (Web)

| Bereich | Wahl | Lizenz |
|---|---|---|
| Admin/Backoffice/Geräte-UI | Vaadin Flow Core | Apache 2.0 |
| Online-Bestellseite | Thymeleaf | Apache 2.0 |
| Interaktivität (öffentl.) | HTMX | BSD-2 |
| Styling (öffentl.) | Tailwind CSS | MIT |
| Client-Logik (öffentl.) | Alpine.js | MIT |
| CSS-Build | Tailwind CLI | MIT |

**Begründung Hybrid-Frontend:** Backoffice-UIs sind komplexe Geschäftsanwendungen mit hoher Interaktivität — Vaadin ist hier produktiv. Öffentliche Online-Bestellseite muss schnell laden, SEO-fähig sein, mobil-optimiert — Thymeleaf+HTMX ist dafür ideal.

## Mobile (Phase 2 / 3)

Im MVP nicht entwickelt, nur architektonisch vorbereitet.

| Bereich | Wahl |
|---|---|
| Plattform | Native Android |
| Sprache | Kotlin |
| UI-Toolkit | Jetpack Compose |
| Min. Android API | 26 (Android 8.0) |
| HTTP-Client | Retrofit + OkHttp |
| JSON | Kotlinx Serialization |
| Auth | OAuth2 Authorization Code Flow + PKCE |
| Lokaler Speicher (für Offline-Modus Phase 3) | Room |

## Testing

| Bereich | Wahl |
|---|---|
| Test-Framework | JUnit 5 |
| Assertions | AssertJ |
| Mocking | Mockito |
| Integration-Tests | Testcontainers (PostgreSQL) |
| API-Tests (Web-Layer) | MockMvc |
| E2E-Tests | Playwright (Java-Bindings) |
| Architektur-Tests | ArchUnit (genutzt durch Spring Modulith) |
| Code-Coverage | JaCoCo |

## Operations

| Bereich | Wahl |
|---|---|
| Containerisierung | Docker + Docker Compose |
| Image-Base | Eclipse Temurin JRE 21 (slim) |
| CI/CD | GitHub Actions |
| Image-Registry | GitHub Container Registry |
| Reverse Proxy | Caddy (auto Let's Encrypt) |
| Session-/Cache-Store (Prod) | Redis 7+ |
| Monitoring | Prometheus + Grafana |
| Uptime-Monitoring (extern) | UptimeRobot Free Tier |
| Backup-Tooling | pg_basebackup + WAL-Streaming via PostgreSQL nativ |
| Backup-Verschlüsselung | GnuPG |

## Hardware (Server)

| Komponente | Spezifikation |
|---|---|
| CPU | Intel i9-9900K |
| RAM | 128 GB DDR4 |
| Storage | 2× 1 TB NVMe (RAID1, gespiegelt) |
| Backup-Storage | 1 TB im zweiten RZ |
| Betriebssystem | Linux (Ubuntu LTS empfohlen) |
| Festplatten-Verschlüsselung | LUKS |

## Verbotene Technologien

Die folgenden Technologien dürfen NICHT verwendet werden, ohne dass ein ADR mit Ausnahmegenehmigung existiert:

- **Kommerzielle Bibliotheken** (auch "Free for Non-Commercial Use"). Open-Source-only ist eine harte Anforderung.
- **GraphQL als API-Schicht** — REST + OpenAPI ist gesetzt.
- **MongoDB / NoSQL** als Hauptdatenbank — PostgreSQL ist gesetzt.
- **WebSocket-basierte Live-Updates** — SSE ist gesetzt (es sei denn, ein zukünftiger Use Case erfordert bidirektionalen Stream).
- **Offline-Frameworks (PouchDB, RxDB) im MVP** — Architektur ist idempotenz-fähig vorbereitet, aber Offline ist nicht aktiv.
- **JavaScript-SPAs (React, Vue, Angular, Svelte) als Hauptframework** — Java-zentriert ist gesetzt. Kleine Inseln von Vanilla-JS / Alpine.js sind okay.
- **Float / Double für Geldbeträge** — Klasse `Money` mit `Long`-Cents ist gesetzt.

## Versionierungs-Politik

- **Java LTS only:** wir bleiben bei Java 21 bis Java 25 stabil ist; Migration ist ein eigener ADR.
- **Spring Boot:** wir folgen Major-Releases nach 6 Monaten Stabilität.
- **Bibliotheken:** Patches automatisch via Dependabot, Minor-Updates wöchentlich, Major-Updates kontrolliert per ADR.
- **PostgreSQL:** wir bleiben auf einer Major-Version, bis EOL nahe ist. Migration auf neuere Major mit explizitem Plan.

## Dependency-Management-Regeln

- **Renovate** oder **Dependabot** aktiv im Repo.
- **Maven Enforcer Plugin** verhindert Versions-Konflikte.
- **OWASP Dependency Check** im CI für bekannte CVEs.
- Jede neue Top-Level-Dependency wird im PR begründet.

## Lizenz-Compliance

- Beim Build wird ein **License-Report** erzeugt (Maven License Plugin).
- Erlaubt: Apache 2.0, MIT, BSD, EPL, LGPL, MPL.
- Verboten ohne Einzelfallprüfung: GPLv2/v3 ohne Classpath Exception, AGPL, kommerzielle Lizenzen, "Custom" Lizenzen.

## Nicht im Stack

Bewusste Auslassungen, die in vergleichbaren Projekten oft auftauchen:

- **Kein Kafka/RabbitMQ/NATS** — Events sind In-Process via Spring.
- **Kein Elasticsearch/OpenSearch** — Suche via PostgreSQL Full-Text Search.
- **Kein Kubernetes** — Single-Host Docker Compose ist ausreichend.
- **Kein API-Gateway** — Caddy macht das Wenige, was nötig ist.
- **Kein Service Mesh** — Modulith hat keine Microservices.
- **Kein separates BI-Tool (Metabase/Superset)** — Reporting ist in der Anwendung integriert.
