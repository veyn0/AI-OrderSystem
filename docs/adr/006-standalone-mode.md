# ADR-006: Standalone-First Architektur

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform soll als **selbstausführbare JAR ohne externe Dienste lauffähig** sein. Begründungen:

- Demo / Probieren: Interessenten sollen die Plattform ohne Cloud-Setup starten können.
- Daten-Hoheit: kleine Restaurants ohne IT-Affinität haben Vorbehalte gegen Cloud-Lösungen.
- Entwicklung: Tests und lokale Iteration sollen ohne Container-Stack laufen.

Die Anforderung ist nicht trivial: Datenbank, Sessions, Mail, Payment, TSE müssen entweder eingebettet oder durch Stubs ersetzt sein.

## Entscheidung

**Standalone-First mit Spring-Profilen.**

- Profil `demo` ist Default und nutzt:
  - Eingebettetes PostgreSQL via `io.zonky.test:embedded-postgres`.
  - In-Memory Session-Store (`MapSessionRepository`).
  - **Fake-Adapter** für alle externen Dienste: `FakePaymentAdapter`, `FakeTseAdapter`, `LogMailAdapter` (→ virtuelles Postfach), `FakeNotificationAdapter`, `PdfReceiptPrinterAdapter`.
- Profil `dev` für lokale Entwicklung mit Container-Diensten (PostgreSQL, Redis), aber weiterhin Fake-Adapter.
- Profil `prod` aktiviert echte Adapter (Stripe, fiskaly, Brevo) und nativen PostgreSQL.

## Alternativen

### H2 / In-Memory-DB für Demo
Verworfen. H2 ist nicht 100 % PostgreSQL-kompatibel. RLS, JSONB-Operatoren, jOOQ-spezifische Features würden divergieren. Bugs erst in Prod entdecken — schlecht.

### Docker als Pflicht
Verworfen. Demo-Hürde zu hoch — viele Interessenten haben Docker nicht installiert oder sind unsicher mit Compose.

### Vollständig getrennte Demo- und Prod-Codebases
Verworfen. Drift zwischen Demo und Prod garantiert. Profile-Konzept hält den Code identisch.

### Auch in Prod Stubs verwenden
Verworfen. Stubs sind nicht GoBD/KassenSichV-konform. Echtkunden-Onboarding erfordert echte Adapter.

## Konsequenzen

### Positiv
- Demo-Aufruf so simpel wie `java -jar restaurant-app.jar`.
- Kein Drift Demo vs. Prod, weil dieselbe Codebase mit anderen Profilen läuft.
- Tests können wahlweise gegen Embedded oder Container-Postgres laufen.
- Standalone-Optionen für datenschutz-bewusste Restaurants verfügbar (Phase 3+).

### Negativ
- **Disziplin nötig:** Code darf nicht profile-spezifisch in der Domäne enthalten. `@Profile`-Annotationen nur in Configuration-Klassen.
- Bei Prod-Konfigurations-Tests muss verifiziert werden, dass kein Fake-Adapter aktiv bleibt → CI-Check.
- Embedded PostgreSQL hat Lizenz Apache 2.0 (zonky), muss im License-Report enthalten sein.
- Demo-Daten und Reset-Endpoints sind dezidiert gekapselt im `demo`-Modul, nur unter `@Profile("demo")` aktiv.

### Neutral
- 3 Profile bedeutet 3 Konfigurations-Sets in `application*.yml` — überschaubar.

## Querverweise

- [`../architecture/standalone-mode.md`](../architecture/standalone-mode.md)
- [`../api/ports-and-adapters.md`](../api/ports-and-adapters.md)
- [ADR-014](014-ports-adapters.md), [ADR-004](004-postgresql.md)
