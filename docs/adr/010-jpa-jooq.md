# ADR-010: JPA für CRUD + jOOQ für Reporting

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform hat zwei sehr unterschiedliche Daten-Zugriffs-Profile:

1. **Schreibende Operationen + einfache Reads** (CRUD): Order anlegen, Item hinzufügen, Status setzen. Aggregate-zentriert, ACID-Transaktionen, optimistische Sperrung.

2. **Reporting-Queries**: Komplexe Aggregationen (Tagesumsatz, Top-10, Heatmap), JOINs über mehrere Tabellen, Window-Functions, Group-By's.

Eine Technologie für beides ist möglich, aber selten optimal.

## Entscheidung

- **Spring Data JPA + Hibernate** für CRUD und Aggregate-Persistence.
- **jOOQ** (OSS-Edition) für komplexe Reporting-Queries.

Beide gleichzeitig im Projekt. Klare Trennung:
- Aggregate-Klassen (Order, MenuItem, Tenant) sind JPA-Entities.
- Reporting-Service-Methoden verwenden jOOQ direkt für SQL-Aggregations.

## Alternativen

### Nur JPA
- Reporting-Queries in JPQL/Criteria sind möglich, aber:
  - Komplexe SQL-Features (Window-Functions, FILTER-Clauses, Subqueries in SELECT) sind in JPQL umständlich.
  - Native Queries als String → keine Type-Sicherheit.
- Verworfen für Reporting-Bereich.

### Nur jOOQ
- Aggregate-Persistence ohne JPA bedeutet manuelle Implementierung von Optimistic Locking, Lifecycle-Callbacks, lazy Loading.
- Aufwand für CRUD wäre höher.
- Verworfen.

### MyBatis
- Mittel-Schicht zwischen JPA und jOOQ. Akzeptabel, aber jOOQ hat bessere Type-Sicherheit.

### Spring JdbcTemplate
- Manueller, low-level. Kein Type-Schema, keine Aggregate-Persistence.
- Verworfen.

### R2DBC (reaktive DB)
- Reaktive Programmierung Overhead nicht gerechtfertigt für unseren Scope.
- Verworfen.

## Konsequenzen

### Positiv
- **Best-of-both-worlds**: JPA für aggregat-orientierte Schreib-Operationen, jOOQ für SQL-Power bei Reads.
- **Type-Sichere SQL** in jOOQ: Compile-Time-Validation.
- **Schema-Generation** in jOOQ aus DDL — Code-Generator integriert in Build.
- Migration zu nur einem Tool später möglich, aber kein Druck.

### Negativ
- **Zwei Tools im Codebase** → Lernen und Pflege beider.
- **jOOQ-Code-Gen-Schritt** im Maven-Build (zusätzliche 30-60 s).
- Dependency-Footprint größer.

### Neutral
- jOOQ OSS-Edition ist Apache 2.0 lizenziert — passt zur Open-Source-Politik.

## Verwendungs-Richtlinien

- **JPA verwenden für:** Aggregate-Schreiben, Aggregate-Lesen mit klaren Beziehungen, Lifecycle (createdAt, updatedAt), Optimistic Locking.
- **jOOQ verwenden für:** Reporting-Aggregationen, Multi-Table-JOINs, Window-Functions, Bulk-Reads ohne Aggregat-Bedarf.
- **NICHT mischen** in derselben Methode — eine Methode entweder JPA oder jOOQ.

## Querverweise

- [`../architecture/tech-stack.md`](../architecture/tech-stack.md)
- [`../architecture/data-model.md`](../architecture/data-model.md)
