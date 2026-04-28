# Live-Updates (Server-Sent Events)

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Live-Updates werden über **Server-Sent Events (SSE)** umgesetzt. Begründung in [ADR-008](../adr/008-sse-over-websocket.md).

## Use Cases für Live-Updates

| Use Case | Sender | Empfänger |
|---|---|---|
| Neue Bestellung an Küche | Backend | KDS-Geräte des Standorts |
| OrderItem-Status geändert | Backend | Bestelltablets, KDS-Geräte des Standorts |
| Online-Bestellung wartet auf Annahme | Backend | Personal-Geräte des Standorts |
| Bestellung READY | Backend | Bestelltablet des Tisches, Endkunden-Statusseite |
| Kellner-Ruf | Backend | Personal-Geräte des Standorts |
| Bestellung ist NEEDS_RESOLUTION | Backend | Personal-Geräte des Standorts |
| Heartbeat-Tickle | Backend | Alle verbundenen Clients (alle 30 s) |

Bidirektionale Kommunikation ist NICHT erforderlich — Client schickt Aktionen über REST, Server pusht Updates über SSE. Daher SSE statt WebSocket.

## SSE-Endpoints

```
GET /api/v1/sse/staff?location=<locationId>
    Auth: Geräte-Token oder Session
    Stream: alle Events für diesen Standort, gefiltert nach Empfänger-Berechtigung
    Response Content-Type: text/event-stream

GET /api/v1/sse/customer/orders/{orderId}
    Auth: Customer-Session oder anonym mit Order-Token aus Bestätigungslink
    Stream: nur Status-Änderungen dieser einen Bestellung
```

## Event-Format

Jedes Event folgt dem SSE-Standardformat:

```
event: ORDER_ITEM_STATUS_CHANGED
id: 18a4f9e0-7b...
data: {
  "occurredAt": "2026-04-28T12:34:56Z",
  "tenantId": "...",
  "locationId": "...",
  "orderId": "...",
  "orderItemId": "...",
  "previousStatus": "IN_PREPARATION",
  "newStatus": "READY"
}

```

**Event-Type-Liste (offiziell):**
- `ORDER_CREATED`
- `ORDER_STATUS_CHANGED`
- `ORDER_ITEM_ADDED`
- `ORDER_ITEM_STATUS_CHANGED`
- `ORDER_NEEDS_ACCEPTANCE`
- `ORDER_NEEDS_RESOLUTION`
- `WAITER_CALL_TRIGGERED`
- `WAITER_CALL_RESOLVED`
- `HEARTBEAT`

Neue Events werden zur Liste ergänzt, nicht ohne ADR umbenannt.

## Empfänger-Filterung

Nicht jedes Event geht an jeden verbundenen Client desselben Standorts. Server-seitige Filterung anhand:

| Modus / Rolle | Empfangene Events |
|---|---|
| KITCHEN_DISPLAY | ORDER_CREATED, ORDER_ITEM_STATUS_CHANGED, ORDER_NEEDS_RESOLUTION |
| ORDER_TABLET | ORDER_ITEM_STATUS_CHANGED, ORDER_NEEDS_ACCEPTANCE, WAITER_CALL_TRIGGERED, WAITER_CALL_RESOLVED, ORDER_NEEDS_RESOLUTION |
| SELF_ORDER_TERMINAL | (keine relevanten Events) |
| Customer-Statusseite | nur ORDER_STATUS_CHANGED für eigene Order |
| User-Browser im Standort-Scope | je nach geöffneter View |

Server bestimmt anhand des authentifizierten Principals (Device.currentMode oder User.role/scope) den Filter.

## Implementierungs-Pattern

### Server: Spring SseEmitter

```java
@RestController
@RequestMapping("/api/v1/sse/staff")
public class StaffSseController {

    private final SseEmitterRegistry registry;

    @GetMapping
    public SseEmitter subscribe(@RequestParam UUID location, Authentication auth) {
        var emitter = new SseEmitter(0L);  // kein Server-Timeout
        var subscription = new Subscription(
            UUID.randomUUID(),
            TenantContext.getCurrentTenant(),
            location,
            auth.getDeviceMode(),  // oder Rolle
            emitter
        );
        registry.register(subscription);

        emitter.onCompletion(() -> registry.unregister(subscription.id()));
        emitter.onTimeout(() -> registry.unregister(subscription.id()));
        emitter.onError(e -> registry.unregister(subscription.id()));

        return emitter;
    }
}
```

### Domain-Event-Brücke

Spring `@TransactionalEventListener` reagiert auf Domain-Events und ruft `SseEmitterRegistry.broadcast(...)`:

```java
@Component
public class OrderEventToSseBridge {

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onOrderItemStatusChanged(OrderItemStatusChangedEvent event) {
        registry.broadcast(
            event.tenantId(),
            event.locationId(),
            "ORDER_ITEM_STATUS_CHANGED",
            event
        );
    }
}
```

`SseEmitterRegistry.broadcast()` iteriert verbundene Subscriber, filtert nach Tenant + Location + Modus-Filter, schickt Event auf jeden passenden Emitter.

## Client-seitig

### Browser (EventSource API)

```javascript
const evt = new EventSource('/api/v1/sse/staff?location=' + locationId, {
    withCredentials: true
});

evt.addEventListener('ORDER_ITEM_STATUS_CHANGED', e => {
    const data = JSON.parse(e.data);
    updateUI(data);
});

evt.onerror = () => {
    // EventSource versucht automatisch reconnect
};
```

### Vaadin Flow

Vaadin Flow kann SSE indirekt nutzen über Server-Push: bei Domain-Event triggert Backend Vaadin's `UI.access(...)` zur UI-Aktualisierung. Im Hintergrund nutzt Vaadin Long-Polling oder WebSocket — das ist ein Implementierungs-Detail von Vaadin und unabhängig vom hier beschriebenen SSE für externe Clients.

**Klarstellung:** SSE wie hier beschrieben gilt für die Geräte-UIs (Tablet, KDS, Self-Order, Endkunden-Statusseite), die alle Thymeleaf-/HTMX- bzw. native-App-basiert sind. Vaadin-Flow-Backoffice-Views nutzen Vaadins eigenen Server-Push-Mechanismus.

## Heartbeat / Keepalive

Server schickt alle 30 Sekunden ein `HEARTBEAT`-Event auf jeden offenen Stream. Zweck:
- Verbindungs-Liveness erkennen.
- HTTP/Proxy-Timeouts vermeiden (manche Proxies schließen idle Verbindungen nach 30-60 s).

Auf Caddy-Ebene ist `proxy_read_timeout` hoch genug konfiguriert.

## Skalierung

**MVP-Annahme:** Single-Server-Setup, alle SSE-Connections im selben JVM. `SseEmitterRegistry` ist eine In-Memory-Map. Maximal etwa 750 simultane Geräte (Wachstumshorizont) stellen für SSE keine Last dar.

**Bei Multi-Server-Setup (Phase 3+):** Erforderlich wäre ein Pub/Sub-Layer (Redis Pub/Sub oder ähnliches), damit ein Domain-Event in Server A auch SSE-Subscriber an Server B erreicht. Im MVP nicht erforderlich.

## Authentifizierung pro SSE-Stream

- **Geräte:** Bearer-Token im `Authorization`-Header. EventSource erlaubt keine Custom-Headers — daher müssen Geräte-Clients einen kurz-lebigen Subscription-Token via separatem Endpoint holen und ihn als `?token=<...>` im SSE-URL übergeben. Dieser Token ist für 1 h gültig und nur für SSE-Lese-Streams.
- **Browser-Sessions:** `withCredentials: true` schickt das Session-Cookie automatisch. Standardpfad.

## Reconnect-Verhalten

- Standard-Browser-EventSource versucht automatisch alle paar Sekunden Reconnect.
- Beim Reconnect kann `Last-Event-ID` im Header geschickt werden, um verlorene Events nachzuholen.
- **MVP:** Server unterstützt **kein** Last-Event-ID-Replay. Bei Reconnect werden nur neue Events ab diesem Moment geliefert.
- Konsequenz: Client muss bei jedem Reconnect den aktuellen Server-Zustand neu laden (z.B. `GET /api/v1/orders?status=IN_PROGRESS&location=...`), um Konsistenz herzustellen.

## Fehler-Verhalten

- Verbindungsabbruch → Client zeigt subtilen Indikator („Live-Updates unterbrochen, Reconnect…"). Bestellungen werden weiterhin lokal angezeigt aus dem letzten Lade-Zustand.
- Server kann Pflicht-Refresh erzwingen via Spezial-Event `FULL_REFRESH_REQUIRED`, woraufhin Client die Seite reload't.

## Tests

- Unit-Test: `OrderEventToSseBridge` veröffentlicht korrektes Event-Format.
- Integration-Test mit Testcontainers: zwei Subscriber, ein Domain-Event, beide bekommen Event.
- Integration-Test: Subscriber für Standort A bekommt KEIN Event für Standort B.
- Last-Test (Phase 2): 1000 simultane SSE-Connections halten ohne Speicher-Leak.
