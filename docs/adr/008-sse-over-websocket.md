# ADR-008: Server-Sent Events statt WebSocket

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform braucht Live-Updates an Geräte und Endkunden:

- Neue Bestellung an KDS pushen.
- Status-Änderungen an Bestelltablets pushen.
- Annahme-Anforderungen an Personal-Geräte pushen.
- Status-Tracking an Endkunden-Statusseite pushen.

Anwendungen können das mit verschiedenen Technologien lösen.

## Entscheidung

**Server-Sent Events (SSE)** für Server→Client Push.

Aktionen vom Client zum Server laufen über klassische REST-Endpoints. Keine bidirektionale Echtzeit-Kommunikation nötig.

## Alternativen

### WebSocket
Erwogen, verworfen, weil:
- **Bidirektional**, was wir nicht brauchen — Client→Server geht über REST, Server→Client genügt SSE.
- Mehr Komplexität: Heartbeats, Reconnect-Logik, Frame-Parsing.
- Weniger Tooling-Freundlichkeit: Reverse Proxies, Load Balancer, Browser-DevTools haben bei SSE einen Vorteil (HTTP-basiert).
- WebSockets sind anfälliger für Probleme mit Corporate-Firewalls und Proxies.
- SSE läuft über HTTPS und ist transparent für Caddy-Reverse-Proxy.

### Long-Polling
Verworfen. Höhere Latenz, höherer Overhead durch ständige neue HTTP-Anfragen.

### Polling (Client fragt regelmäßig)
Verworfen. Hohe Last, hohe Latenz, schlechte UX.

### gRPC Streaming
Verworfen. Sprachwechsel-ähnlich, mehr Tooling, REST-Stack ist bereits gesetzt.

### Externer Pub/Sub (Redis Pub/Sub direkt an Client)
Verworfen. Nicht direkt aus dem Browser ansprechbar, Sicherheits-Komplikationen.

## Konsequenzen

### Positiv
- **Standardisierter Browser-API** (`EventSource`) — keine zusätzlichen Bibliotheken.
- **Auto-Reconnect** ist Browser-built-in.
- **HTTP/HTTPS-basiert** → arbeitet durch alle Reverse Proxies und Firewalls hindurch.
- **Einfacher Debug** mit Browser-DevTools.
- Server-Implementation in Spring Boot trivial via `SseEmitter`.
- Kein zusätzlicher Pub/Sub-Layer im MVP nötig.

### Negativ
- **Unidirektional** — Client kann nicht über denselben Channel Daten an Server schicken (aber das brauchen wir nicht).
- **Kein binäres Format** — alles als Text. Für unsere kleinen JSON-Payloads kein Problem.
- **Connection-Limit pro Browser**: 6 SSE-Verbindungen pro Origin (HTTP/1.1). HTTP/2 hebt das auf. In Prod nutzen wir HTTP/2 via Caddy.
- **Bei Multi-Server-Skalierung** brauchen wir Pub/Sub-Brücke (z.B. Redis), um SSE-Events servern-übergreifend zu verteilen. MVP ist Single-Server, daher kein Problem.

### Neutral
- EventSource-API erlaubt keine Custom-Headers — Auth via Query-Parameter (kurzlebiger Subscription-Token). Akzeptable Lösung.

## Querverweise

- [`../architecture/live-updates.md`](../architecture/live-updates.md)
- [`../api/api-design-principles.md`](../api/api-design-principles.md)
