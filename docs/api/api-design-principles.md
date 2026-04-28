# API-Design-Prinzipien

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Plattform exponiert eine REST-API für Geräte (MVP) und native Mobile (Phase 2). Dokumentation via OpenAPI 3.x, automatisch generiert über `springdoc-openapi`.

## Allgemeine Prinzipien

- **Versionierung im Pfad:** `/api/v1/...`. Major-Version-Änderungen erfordern parallele Pfade (`/v1/`, `/v2/`).
- **JSON only:** `Content-Type: application/json`, `Accept: application/json`. Keine XML-Unterstützung.
- **HTTPS-only in Prod:** HTTP nur im Demo (localhost).
- **camelCase** für Feld-Namen in JSON-Bodies. URL-Pfade in **kebab-case** für Mehrwort-Begriffe (`/api/v1/order-items` nicht `/api/v1/orderItems`).
- **Plural-URLs für Sammlungen:** `/api/v1/orders`, `/api/v1/menu-items`, NICHT `/api/v1/order`.
- **Pflicht-Auth auf allen Endpoints außer einer engen Ausnahme-Liste** (`/api/v1/health`, `/api/v1/login`, `/api/v1/customer/register`, Webhook-Endpoints).

## Auth-Header

| Akteur | Header |
|---|---|
| Gerät | `Authorization: Bearer <device-token>` |
| Web-Session (Browser) | Cookie `SESSION=...` (auto via Browser) |
| Mobile (Phase 2) | `Authorization: Bearer <oauth-access-token>` |

## Pfad-Konventionen

```
/api/v1/                           ← API-Root
  health                           ← Liveness-Check (kein Auth)
  health/ready                     ← Readiness (kein Auth, prüft DB-Verbindung)
  auth/
    login                          ← POST E-Mail+Passwort
    logout                         ← POST
    manager-confirm                ← POST PIN-Bestätigung für Override-Aktion
  device/
    heartbeat                      ← POST
    me                             ← GET aktuelle Geräte-Info
  tenants/{tenantId}               ← Plattform-Admin-Endpoints
  locations/{locationId}/...
  menu/
    master/                        ← Stamm-Speisekarte
    locations/{locationId}/        ← effektive Standort-Speisekarte
  orders/
    {orderId}/
      items/
      transitions/                 ← State-Transition-Endpoints
      charge                       ← POST Bestellung abrechnen
  customers/
    me/                            ← eingeloggter Endkunde
  reporting/
    kpi
    drilldown
  audit/
  sse/
    staff
    customer/orders/{orderId}
  payments/webhook                 ← Webhook von Payment-Provider (Phase 2)
```

## HTTP-Status-Codes

| Code | Verwendung |
|---|---|
| 200 OK | Erfolgreicher GET, PUT, oder POST mit Body-Antwort |
| 201 Created | Erfolgreicher POST mit Resource-Erstellung |
| 204 No Content | Erfolgreicher DELETE oder PUT ohne Body |
| 400 Bad Request | Validierungsfehler (siehe Body) |
| 401 Unauthorized | Auth fehlt oder ungültig |
| 403 Forbidden | Auth ok, aber Permission fehlt |
| 404 Not Found | Resource existiert nicht oder ist nicht im Tenant-Scope sichtbar |
| 409 Conflict | Konflikt (z.B. Optimistic-Lock-Versionsfehler, doppelte E-Mail) |
| 422 Unprocessable Entity | Semantische Validierung scheitert (z.B. ungültige State-Transition) |
| 429 Too Many Requests | Rate-Limit überschritten |
| 500 Internal Server Error | Unerwartet — Logging-Eintrag erforderlich |

**WICHTIG:** Tenant-Cross-Reads geben 404 zurück, NICHT 403 — andernfalls wäre die Existenz von Resources in fremden Tenants ableitbar.

## Fehler-Antwort-Format

Einheitliches Fehler-Schema (RFC 7807-light):

```json
{
  "code": "ORDER_INVALID_TRANSITION",
  "message": "Cannot transition from PAID to IN_PROGRESS",
  "details": {
    "currentStatus": "PAID",
    "attemptedStatus": "IN_PROGRESS"
  },
  "traceId": "abc-123",
  "timestamp": "2026-04-28T12:34:56Z"
}
```

`code` ist eine stabile Maschinen-Kennung. `message` ist menschenlesbar (Englisch im API; UI lokalisiert). `traceId` ermöglicht Backend-Logsuche.

## Pagination

Listen-Endpoints unterstützen Pagination via Query-Parameter:

```
GET /api/v1/orders?page=0&size=20&sort=createdAt,DESC
```

**Antwort:**
```json
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 145,
  "totalPages": 8
}
```

Standard-`size` = 20, Maximum 200.

## Filterung

Listen-Endpoints unterstützen Filter via Query-Parameter, dokumentiert im OpenAPI:

```
GET /api/v1/orders?location=<id>&status=IN_PROGRESS&from=2026-04-28T00:00:00Z
```

Mehrere Werte für gleichen Filter: Komma-separiert (`status=IN_PROGRESS,READY`).

## Idempotenz

**Pflicht für mutierende Endpoints**, vorbereitend für späteren Offline-Modus:

Client sendet bei POST/PUT einen `Idempotency-Key`-Header (UUID, vom Client generiert):

```
POST /api/v1/orders
Idempotency-Key: 7c3b6e4a-...
```

Server speichert pro Key + Endpoint die erste Antwort 24 h. Wiederholter Aufruf mit gleichem Key gibt gespeicherte Antwort zurück.

**Idempotenz-Speicher:** Tabelle `idempotency_keys (key, endpoint, request_hash, response_status, response_body, created_at)`.

**Verhalten:**
- Gleicher Key + gleicher Request-Body → gespeicherte Antwort.
- Gleicher Key + abweichender Request-Body → 422 mit Code `IDEMPOTENCY_KEY_MISMATCH`.
- Im MVP: Pflicht für `POST /orders`, `POST /orders/{id}/items`, `POST /orders/{id}/charge`. Empfohlen für alle weiteren mutierenden Endpoints.

## State-Transitions

State-Transitions werden NICHT als generisches `PATCH /orders/{id}` mit Status-Feld umgesetzt. Stattdessen jeweils eigener Endpoint pro Aktion:

```
POST /api/v1/orders/{id}/transitions/send-to-kitchen
POST /api/v1/orders/{id}/transitions/cancel        Body: {"reason": "..."}
POST /api/v1/orders/{id}/transitions/serve
POST /api/v1/orders/{id}/charge                    Body: {"method": "CASH", "amountReceived": ...}
```

**Vorteile:**
- Klare Semantik pro Endpoint.
- Eigene Validierung pro Aktion.
- Eigene Permission-Prüfung.
- OpenAPI-Doku ist verständlicher.

## Concurrency / Optimistic Locking

Mutierende Endpoints, die auf bestehende Aggregate wirken, akzeptieren ein `If-Match: <version>` Header. Bei abweichender Version → 409 mit Code `VERSION_CONFLICT`.

```
PUT /api/v1/menu-items/{id}
If-Match: 17
```

Server-seitig per JPA-`@Version`-Feld gemappt.

## Datums- und Zeit-Format

- **Alles als ISO-8601 UTC mit Z**: `2026-04-28T12:34:56Z`.
- Zeitzonen-Konvertierung in UI passiert client-seitig.
- Datum-only: `2026-04-28`.
- Zeit-only: `12:34:56`.

## Geld

Zentrale Konvention:
- `*Cents` Suffix in Feldnamen: `priceCents`, `totalGrossCents`.
- Wert: ganzzahlig, `Long`.
- Währung als zusätzliches Feld `currency: "EUR"` wo Mehrwert.

```json
{
  "priceCents": 950,
  "currency": "EUR"
}
```

NICHT:
```json
{ "price": 9.50 }     ← VERBOTEN (Float)
{ "price": "9.50" }   ← VERBOTEN (String-Geld)
```

## Validierung

Client-Inputs werden mit Jakarta Bean Validation validiert (`@NotNull`, `@Size`, `@Email` etc.). Bei Verletzung → 400 mit detail-strukturierter Antwort:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": {
    "errors": [
      {"field": "email", "code": "INVALID_FORMAT"},
      {"field": "items[0].quantity", "code": "MUST_BE_POSITIVE"}
    ]
  }
}
```

## Rate-Limiting

Pro IP / Authentifizierung definierte Limits, durchgesetzt auf Caddy-Ebene und/oder Spring-Filter:

| Endpoint-Klasse | Limit |
|---|---|
| `/api/v1/auth/login` | 10 / Stunde / IP |
| `/api/v1/customer/register` | 5 / Stunde / IP |
| `/api/v1/sse/*` | 1 Verbindung / Auth-Principal gleichzeitig (sanft, Reconnect erlaubt) |
| Kellner-Ruf (Tisch-QR) | 1 / 3 min / Tisch |
| Sonstige Schreib-Endpoints | 100 / Minute / Auth-Principal |

Bei Überschreitung: 429, mit `Retry-After`-Header.

## CORS

API ist by default **same-origin only**. Native Mobile (Phase 2) kommt vom App-Native-Code, nicht über Browser → keine CORS-Anforderung.

Falls externe Browser-Clients in Zukunft erforderlich sind, wird CORS über explizite Origin-Whitelist freigeschaltet.

## OpenAPI / Doku

- `/v3/api-docs` (JSON OpenAPI 3.x).
- `/swagger-ui` (UI im Demo/Dev, NICHT in Prod aktiv).
- Jeder Controller-Method hat `@Operation`, `@ApiResponses`, `@Parameter`-Annotationen.
- DTOs haben `@Schema(description=...)`.

## API-Beispiel-Antworten

### `GET /api/v1/orders/{id}`

```json
{
  "id": "8f3a-...",
  "tenantId": "...",
  "locationId": "...",
  "tableId": "...",
  "orderType": "ON_SITE",
  "orderMode": "OPEN_TAB",
  "status": "IN_PROGRESS",
  "items": [
    {
      "id": "ab12-...",
      "menuItemId": "...",
      "menuItemName": "Pizza Margherita",
      "variantId": "...",
      "variantName": "Mittel",
      "quantity": 1,
      "selectedOptions": [
        {"id": "...", "name": "Extra Käse"}
      ],
      "priceSnapshotCents": 1100,
      "taxRateSnapshot": 19.0,
      "status": "IN_PREPARATION",
      "notes": null
    }
  ],
  "totalGrossCents": 1100,
  "totalNetCents": 924,
  "totalTaxCents": 176,
  "taxBreakdown": {"19.0": 176},
  "createdAt": "2026-04-28T12:00:00Z",
  "version": 3
}
```

### `POST /api/v1/orders`

Request:
```json
{
  "locationId": "...",
  "orderType": "ON_SITE",
  "orderMode": "OPEN_TAB",
  "tableId": "..."
}
```

Header: `Idempotency-Key: <uuid>`

Response 201:
```json
{ "id": "8f3a-...", "status": "DRAFT", "version": 0, ... }
```

## Versionierungs-Politik

- **Patches/Bugfixes:** keine Pfad-Änderung.
- **Backwards-compatible Erweiterungen:** Neue optionale Felder, neue Endpoints — bleibt v1.
- **Breaking Changes:** parallele v2-Pfade. v1 bleibt mindestens 12 Monate erhalten, danach Deprecation.
- **Deprecation:** mit `Deprecation: true` Response-Header ankündigen, OpenAPI markieren.

## Konventionen verwerfen

- Keine `actions=...`-Query-Parameter im REST-Stil.
- Keine HTTP-Methoden wie `PROPFIND` o.ä.
- Kein Hypermedia-API (HATEOAS) — Overkill für unsere Klienten.
- Kein gRPC, kein Protobuf — JSON ist gesetzt.
