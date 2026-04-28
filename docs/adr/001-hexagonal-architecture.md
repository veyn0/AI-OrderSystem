# ADR-001: Hexagonal Architecture (Ports & Adapters)

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform muss langfristig wartbar sein, mehrere externe Abhängigkeiten austauschbar halten (Payment, TSE, Mail) und einen Standalone-Modus ohne externe Dienste unterstützen. Außerdem braucht die Domänenlogik testbarkeit ohne Mock-Geflechte.

Klassische Schichtenarchitektur (UI → Service → Repository → DB) verwischt die Grenze zwischen Geschäftslogik und Infrastruktur. Externe Abhängigkeiten wie Payment-Provider werden direkt in Service-Klassen aufgerufen, was Tests und Adapter-Wechsel erschwert.

## Entscheidung

Wir verwenden **Hexagonal Architecture** (Alistair Cockburn, „Ports & Adapters"):

- **Domänenkern** ist isoliert. Er kennt keine Spring-Annotationen, keine JPA, keine HTTP-Bibliotheken.
- **Ports** sind Interfaces, die der Domänenkern definiert (z.B. `PaymentPort`, `TsePort`).
- **Driving Adapters** rufen die Domäne auf (z.B. REST-Controller, Vaadin Views).
- **Driven Adapters** implementieren die Ports (z.B. `StripePaymentAdapter`, `FakePaymentAdapter`).

Konkret in der Modulith-Struktur:
- `<modul>/domain/` — Aggregate, Entities, Value Objects, ohne Framework.
- `<modul>/api/` — exponierte Service-Interfaces und Events (Ports).
- `<modul>/internal/` — Implementierungen, JPA-Repositories, Adapter.

## Alternativen

### Klassische Schichtenarchitektur
Verworfen. Verwischt Grenze zwischen Domäne und Infrastruktur. Tests verlangen schwere Mock-Setups oder In-Memory-Frameworks. Adapter-Wechsel (Stripe → Mollie) berührt Service-Code.

### Clean Architecture (Robert C. Martin)
Sehr ähnlich zu Hexagonal Architecture. Wird in der Praxis oft synonym verwendet. Wir nennen es „Hexagonal", weil das Vokabular der Ports und Adapter konkreter ist.

### Onion Architecture
Im Wesentlichen Hexagonal mit zusätzlichen Schichten. Mehr Komplexität, kein nennenswerter Mehrwert für unseren Scope.

### CQRS / Event Sourcing
Für Reporting interessant, aber für die schreibende Domäne übertrieben. Wir verwenden CQRS-light: Reporting-Reads via jOOQ direkt, Writes via JPA-Aggregates. Event Sourcing nicht im Scope.

## Konsequenzen

### Positiv
- Domänenlogik testbar mit POJOs, ohne Spring-Context oder Datenbank.
- Externe Abhängigkeiten austauschbar (Stripe ↔ Mollie ↔ Fake).
- Standalone-Modus ist eine direkte Folge: andere Adapter-Auswahl, fertig.
- Modul-Grenzen sind klar und werden durch ArchUnit / Spring Modulith verifiziert.

### Negativ
- Mehr Klassen (Interface + Implementation für jede Operation).
- Initial mehr Boilerplate.
- Disziplin nötig: Domäne darf nicht Spring-Annotationen tragen, sonst leakt Infrastruktur.

### Neutral
- Lernkurve für Entwickler, die nur Schichtenarchitektur kennen — aber dokumentiert in [`../architecture/overview.md`](../architecture/overview.md) und [`../architecture/modulith-structure.md`](../architecture/modulith-structure.md).

## Querverweise

- [`../architecture/overview.md`](../architecture/overview.md)
- [`../architecture/modulith-structure.md`](../architecture/modulith-structure.md)
- [`../api/ports-and-adapters.md`](../api/ports-and-adapters.md)
- [ADR-002](002-modulith-over-microservices.md), [ADR-014](014-ports-adapters.md)
