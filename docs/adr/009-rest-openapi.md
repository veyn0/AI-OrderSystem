# ADR-009: REST + OpenAPI statt GraphQL

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform exponiert eine API für Geräte (MVP) und native Mobile (Phase 2). Die Klienten sind kontrolliert (von uns selbst entwickelt), nicht öffentlich für Drittentwickler.

Standard-Optionen: REST (mit JSON), GraphQL, gRPC, SOAP.

## Entscheidung

**REST** mit JSON-Bodies. **OpenAPI 3.x** als Vertragsdokumentation, automatisch generiert über `springdoc-openapi`.

Versionierung im URL-Pfad: `/api/v1/...`.

## Alternativen

### GraphQL
Erwogen, verworfen, weil:
- **Klienten sind unsere eigenen** — wir wissen, welche Daten gebraucht werden.
- **Kein Multi-Source-Aggregations-Bedarf** — alle Daten kommen aus einer DB.
- **Caching-Komplexität** in GraphQL höher.
- **Authentifizierung pro Field** komplex.
- **N+1-Query-Problem** ist GraphQL-typisch.
- **Lernkurve** für den Solo-Entwickler vermeidbar.

GraphQL glänzt bei öffentlichen APIs mit unbekannten Klienten (z.B. GitHub API). Das ist nicht unsere Situation.

### gRPC
Verworfen.
- Bessere Performance, aber HTTP/REST ist für uns ausreichend.
- Browser-Klienten müssen Web-gRPC-Proxy verwenden — Komplexität.
- Tooling für mobile Klienten weniger reif.

### SOAP
Verworfen. XML, schwer für moderne JSON-basierte Klienten, veraltetes Pattern.

### tRPC / Custom RPC
Verworfen. Tooling-Lock-in, weniger Standard.

## Konsequenzen

### Positiv
- **Einfacher Standard**: jeder kann REST.
- **OpenAPI-Generierung** automatisch aus Spring-Annotationen.
- **Swagger-UI** für interaktives Debugging in Demo/Dev.
- **HTTP-Caching** trivial (ETags, etc.).
- Pagination, Filtering, Sorting via Query-Parameter — Standard-Konventionen.
- Idempotenz-Header (`Idempotency-Key`) Standard-konform.

### Negativ
- **Over-Fetching** möglich — Klienten bekommen ggf. mehr Felder als sie brauchen. Für interne Klienten akzeptabel.
- **Multiple Round-Trips** bei zusammengesetzten Views möglich. Bislang nicht problematisch.
- Versionierung-Strategie nötig — wir verwenden URL-Pfad-Versionierung.

### Neutral
- HATEOAS / Hypermedia-API NICHT angestrebt — Overkill für unsere Klienten.

## Querverweise

- [`../api/api-design-principles.md`](../api/api-design-principles.md)
- [ADR-008](008-sse-over-websocket.md)
