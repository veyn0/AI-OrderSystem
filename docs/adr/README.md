# Architecture Decision Records (ADRs)

> **Status:** Aktiv
> **Letzte Aktualisierung:** 2026-04-28

ADRs dokumentieren wichtige Architektur-Entscheidungen mit Kontext und Konsequenzen. Ziel: Spätere Lesende verstehen, **warum** etwas so gewählt wurde.

## Format

Jeder ADR folgt dem gleichen Aufbau:

1. **Status** — Proposed / Accepted / Superseded by ADR-XXX / Deprecated
2. **Kontext** — Welches Problem? Welche Randbedingungen?
3. **Entscheidung** — Was wurde entschieden?
4. **Alternativen** — Was wurde abgewogen, und warum verworfen?
5. **Konsequenzen** — Was folgt aus der Entscheidung (positiv und negativ)?

## Pflicht zur ADR-Erstellung

Ein ADR ist Pflicht für:
- Wahl von Sprache, Framework, Datenbank.
- Architektur-Patterns (Modulith vs Microservices, Hexagonal vs Layered).
- Multi-Tenancy-Modell.
- Sicherheits-Modelle (Auth, Tokens).
- Kommunikations-Patterns (REST vs gRPC, SSE vs WebSocket).
- Wahl externer Provider (Payment, TSE, Mail).
- Lizenz-Politik.

ADRs sind **nicht** verpflichtend für:
- Routine-Implementierungen.
- Kleine Bibliotheks-Wechsel.
- Internes Refactoring.

## Numerierung

ADRs werden fortlaufend nummeriert: `001-...`, `002-...`. Lücken nicht auffüllen — Reihenfolge der Entscheidungen ist Teil der Geschichte.

## Wenn eine Entscheidung revidiert wird

- Status des alten ADR auf `Superseded by ADR-XXX` setzen, Begründung kurz hinzufügen.
- Neuer ADR mit eigener Nummer, Verweis auf abgelöste Entscheidung.
- Alter ADR bleibt im Repo erhalten — Historie bleibt sichtbar.

## ADR-Liste

| Nr. | Titel | Status |
|---|---|---|
| [001](001-hexagonal-architecture.md) | Hexagonal Architecture | Accepted |
| [002](002-modulith-over-microservices.md) | Modulith statt Microservices | Accepted |
| [003](003-java-spring.md) | Java + Spring Boot als Backend-Stack | Accepted |
| [004](004-postgresql.md) | PostgreSQL als Datenbank | Accepted |
| [005](005-multi-tenancy.md) | Shared DB + Row-Level Security | Accepted |
| [006](006-standalone-mode.md) | Standalone-First mit Profilen | Accepted |
| [007](007-hybrid-frontend.md) | Hybrid Frontend (Vaadin + Thymeleaf) | Accepted |
| [008](008-sse-over-websocket.md) | SSE statt WebSocket | Accepted |
| [009](009-rest-openapi.md) | REST + OpenAPI statt GraphQL | Accepted |
| [010](010-jpa-jooq.md) | JPA für CRUD + jOOQ für Reporting | Accepted |
| [011](011-sessions-oauth.md) | Server-Sessions Web, OAuth2 Mobile | Accepted |
| [012](012-device-tokens.md) | Geräte-Tokens (Bearer, langlebig) | Accepted |
| [013](013-staff-pin.md) | Mitarbeiter-PIN am Gerät | Accepted |
| [014](014-ports-adapters.md) | Ports & Adapters für externe Dienste | Accepted |
| [015](015-audit-retention.md) | Audit-Retention 90 Tage / 10 Jahre | Accepted |
| [016](016-oss-only.md) | Open-Source-only-Politik | Accepted |
| [017](017-customer-per-tenant.md) | Endkunden-Konten pro Tenant | Accepted |
| [018](018-table-qr-anti-abuse.md) | Tisch-QR-Code Anti-Missbrauch | Accepted |

## Referenz-ADRs (Vorlagen für andere Projekte)

Diese Sammlung folgt dem Stil von Michael Nygards „Documenting Architecture Decisions" (2011), erweitert um den Punkt „Alternativen" als eigene Sektion für bessere Nachvollziehbarkeit.
