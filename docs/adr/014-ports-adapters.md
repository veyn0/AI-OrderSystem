# ADR-014: Ports & Adapters für externe Dienste

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform integriert mehrere externe Dienste mit unterschiedlichen Stadien:

- **Payment** (Stripe / Mollie / Adyen) — Phase 2.
- **TSE** (fiskaly / Epson / Swissbit) — vor Echtkunden-Go-Live.
- **Mail** (Brevo / Mailjet / SES) — Phase 2.
- **SMS / Push** (Twilio / FCM) — Phase 2/3.
- **Bondrucker** (ESC/POS-Geräte) — Phase 2.
- **Buchhaltung** (DATEV) — Phase 2.

Im MVP existieren keine echten Adapter — alles muss über Stubs funktionieren. Der Domänencode darf nicht von Provider-spezifischen Bibliotheken abhängen.

## Entscheidung

**Pro externer Abhängigkeit ein Port (Interface) + mindestens zwei Adapter (Fake + Real).**

Konkrete Ports:

| Port | Modul | Fake | Real (Phase 2+) |
|---|---|---|---|
| `PaymentPort` | `payment` | `FakePaymentAdapter` | `StripePaymentAdapter` (oder Mollie etc.) |
| `TsePort` | `tse` | `FakeTseAdapter` | `FiskalyTseAdapter` (oder Epson etc.) |
| `MailPort` | `notification` | `LogMailAdapter` | `BrevoMailAdapter` (oder Mailjet etc.) |
| `NotificationPort` | `notification` | `FakeNotificationAdapter` | `TwilioSmsAdapter` + `FcmPushAdapter` |
| `ReceiptPrinterPort` | `notification` | `PdfReceiptPrinterAdapter` | `EscPosNetworkAdapter` |
| `AccountingExportPort` | `reporting` | (kein MVP-Adapter) | `DatevCsvExportAdapter` |

Adapter werden via `@ConditionalOnProperty` aktiviert. Domänencode hängt nur am Port-Interface.

## Alternativen

### Direktnutzung von Provider-SDKs in Service-Klassen
Verworfen. Domäne wäre an konkrete Bibliothek gekoppelt. Stub-Implementierung umständlich.

### Anti-Corruption-Layer ohne Port-Pattern
Ähnliches Konzept, aber lockerer. Wir nehmen das strikte Port&Adapter-Schema, weil Spring Modulith das gut unterstützt.

### Service-Provider-Interface (Java-SPI)
Erwogen, verworfen. Spring's `@Conditional`-Mechanismus passender für Profile-basierten Adapter-Wechsel.

## Konsequenzen

### Positiv
- **Domänencode ist Provider-agnostisch.**
- **Stub-Implementierung trivial** — Demo-Adapter geben sinnvolle Werte zurück.
- **Provider-Wechsel später möglich** ohne Domänen-Änderungen.
- **Tests verwenden Fake-Adapter** — schnelle und deterministische Tests.

### Negativ
- **Mehr Klassen** (Port + Fake + Real).
- **Disziplin nötig**: Domäne darf KEINE direkten Imports der Provider-SDKs haben. ArchUnit-Tests durchsetzen.
- **Bei Provider-Wechsel**: Adapter-Implementierung neu, aber das ist akzeptabel — der Wechsel passiert selten.

### Neutral
- Ports sind in `<modul>/api/`-Sub-Package, Adapter in `<modul>/internal/`.

## Querverweise

- [`../api/ports-and-adapters.md`](../api/ports-and-adapters.md)
- [`../architecture/standalone-mode.md`](../architecture/standalone-mode.md)
- [ADR-001](001-hexagonal-architecture.md), [ADR-006](006-standalone-mode.md)
