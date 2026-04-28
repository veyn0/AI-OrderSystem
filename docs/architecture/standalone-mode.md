# Standalone-Mode (Demo-Profil)

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Plattform MUSS als selbstausführbare JAR ohne externe Dienste lauffähig sein. Dies ist eine fundamentale Architektur-Anforderung, kein optionales Feature.

## Anforderung

```
$ java -jar restaurant-app.jar
$ # → Anwendung läuft, alle MVP-Use-Cases ausführbar
$ # → http://localhost:8080
```

Voraussetzungen:
- Java 21 LTS Runtime auf dem Host installiert.
- Schreibzugriff im aktuellen Arbeitsverzeichnis.
- Port 8080 frei.

Keine weiteren Voraussetzungen. Insbesondere KEINE:
- PostgreSQL-Installation
- Redis-Installation
- Docker
- Internetverbindung
- API-Keys, Credentials, externe Konten

## Spring-Profil-System

Drei Profile, exakt entsprechend [ADR-006](../adr/006-standalone-mode.md):

| Profil | Zweck | Aktivierung |
|---|---|---|
| `demo` | Standalone-Default | Default ohne weitere Aktivierung |
| `dev` | Lokale Entwicklung mit Container-Diensten | `-Dspring.profiles.active=dev` |
| `prod` | Produktivbetrieb | `-Dspring.profiles.active=prod` |

**Aktivierungs-Logik:**
```
spring.profiles.default=demo
```
Das bewirkt: ohne explizite Profil-Setzung läuft `demo`.

## Komponenten-Konfiguration je Profil

| Komponente | Demo | Dev | Prod |
|---|---|---|---|
| Datenbank | zonky embedded PostgreSQL | Container PostgreSQL | Native PostgreSQL |
| Session-Store | In-Memory (`MapSessionRepository`) | Redis Container | Redis |
| Mail-Adapter | LogMail → virtuelles Postfach | LogMail | echter Adapter (Brevo/Mailjet) |
| Payment-Adapter | FakePayment | FakePayment | echter Adapter (Stripe/Mollie) |
| TSE-Adapter | FakeTSE | FakeTSE | echter Adapter (fiskaly) |
| Notification-Adapter | FakeNotification | FakeNotification | echter Adapter (Twilio/FCM) |
| Receipt-Printer-Adapter | PdfReceiptPrinter | PdfReceiptPrinter | echter Adapter (ESC/POS) |
| Demo-Daten | beim Start geladen | optional via Flag | NIEMALS |
| Demo-Reset-Endpoint | aktiv | aktiv | DEAKTIVIERT |
| Zeit-Sprung-Funktion | aktiv | aktiv | DEAKTIVIERT |
| TLS | KEIN (HTTP localhost) | optional | Pflicht |
| Plattform-Admin-Default | Ja, vorbelegt | Optional | NEIN, manuell anlegen |

## Embedded PostgreSQL (Demo-Profil)

Verwendet wird `io.zonky.test:embedded-postgres` (Apache 2.0). Diese Bibliothek startet eine echte PostgreSQL-Instanz aus einem heruntergeladenen Binary, lokal verzeichnis-basiert. Vorteile:
- Funktional 100 % identisch zu nativer PostgreSQL — RLS, jOOQ, Funktionen, Trigger, alles.
- Kein In-Memory-Engine wie H2, der semantische Unterschiede hätte.

**Datenverzeichnis:** `./data/embedded-pg` (relativ zum JAR-Aufruf-Verzeichnis).
- Beim ersten Start: Verzeichnis wird angelegt, PostgreSQL initialisiert, Flyway-Migrationen laufen, Demo-Daten werden geladen.
- Bei Folgestarts: Datenverzeichnis wird wiederverwendet, Daten persistieren über Neustarts hinweg.

**Konfiguration:**
```yaml
spring:
  datasource:
    embedded:
      enabled: true
      data-dir: ${user.dir}/data/embedded-pg
```

**Lebenszyklus:** Spring-Bean steuert Start/Stop. Beim Anwendungs-Shutdown wird PostgreSQL sauber heruntergefahren.

## Demo-Daten-Loader

Komponente: `de.deinrestaurant.demo.DemoDataLoader`, aktiv nur unter `@Profile("demo")`.

**Wird beim Start ausgeführt**, wenn die DB leer ist (Erkennung: `tenants`-Tabelle hat 0 Zeilen).

**Geladene Daten:**

### Plattform-Admin
- E-Mail: `admin@demo.local`
- Passwort: `admin`
- Rolle: `SuperAdmin`

### Test-Tenant „Pizzeria Demo"
- Slug: `demo`
- 1 Inhaber: `inhaber@demo.local`, Passwort `demo`
- 1 Geschäftsführer: `manager@demo.local`, Passwort `demo`
- 2 Standorte:
  - **„Berlin Mitte"** mit 5 Tischen, 3 Geräten (Bestelltablet, KDS-Display, Self-Order-Terminal)
  - **„Berlin Kreuzberg"** mit 4 Tischen, 2 Geräten

### Speisekarte
- Kategorien: Pizza, Pasta, Salate, Getränke, Desserts
- 24 Artikel, jeweils mit Allergenen, Varianten (Klein/Mittel/Groß), Optionen (Beläge, Soßen)
- 2 Standort-Overrides als Beispiel (z.B. Pizza Margherita 50 Cent günstiger in Kreuzberg)

### Geräte mit fixen Tokens (für Tests bequem)
- Tablet Berlin Mitte 1: Token `demo-tablet-bm-1`
- KDS Berlin Mitte: Token `demo-kds-bm`
- Self-Order Berlin Mitte: Token `demo-self-bm`
- Tablet Berlin Kreuzberg 1: Token `demo-tablet-bk-1`
- KDS Berlin Kreuzberg: Token `demo-kds-bk`

**Hinweis:** Diese fixen Demo-Tokens funktionieren NUR im Demo-Profil. In `dev`/`prod` werden Tokens generiert.

### Test-Endkunden
- 3 registrierte Endkunden mit jeweils mehreren Adressen.
- 1 Test-Endkunde mit langer Bestellhistorie (für Reporting-Demo).

### Bestellhistorie
- 50 historische Bestellungen über die letzten 14 Tage, gemischte Status, gemischte Bestelltypen — sodass Reporting beim ersten Login gefüllt ist.

### Annahme-Policy
- Berlin Mitte: AutoAnnahme.
- Berlin Kreuzberg: BY_TIME-Regel: Fr/Sa 18-23 Uhr ManuellAblehnen-im-Zeitfenster, sonst Auto.

## Demo-Reset-Endpoint

`POST /demo/reset` (nur im Demo-Profil aktiv):

1. Server lehnt ab, wenn Profil != demo: HTTP 404.
2. Bestätigungs-Header erforderlich: `X-Confirm: yes-i-want-to-reset`. Sonst 400.
3. Alle Tabellen werden geleert (außer Flyway-History).
4. `DemoDataLoader.load()` wird erneut ausgeführt.
5. Antwort: 200 mit Zusammenfassung („Datenbank zurückgesetzt, X Tenants/Y Bestellungen geladen.").

**Im UI:** Eigene Demo-Seite `/demo` mit großem Reset-Button mit Bestätigungs-Modal.

## Zeit-Sprung-Funktion

Im Demo-Profil ist die Anwendungszeit über einen `Clock`-Bean steuerbar:

```java
@Bean
public Clock systemClock(@Value("${app.clock-mode:system}") String mode) {
    if ("offset".equals(mode)) {
        return offsetableClock();   // im Demo bereitgestellt
    }
    return Clock.systemDefaultZone();
}
```

UI-Aktion: `POST /demo/time-jump?days=7` springt um 7 Tage nach vorne. Folgen:
- Reporting für die letzten 30 Tage zeigt aktualisierte Daten.
- Synthetische Bestellungen für den übersprungenen Zeitraum werden eingefügt.

In `prod`: `Clock.systemDefaultZone()` ist hart verdrahtet. Im Code DARF NICHT `LocalDateTime.now()` verwendet werden — stattdessen immer `clock.instant()` über injizierten `Clock`.

## Virtuelles Postfach

Das virtuelle Postfach ersetzt im Demo-Profil:
- E-Mails (alle MailPort-Aufrufe schreiben dorthin)
- Push-Notifications
- SMS-Nachrichten
- Bondrucke (PDFs)

**Implementation:**
- Tabelle `virtual_inbox_messages` (nur im Demo-Profil migriert):

```sql
CREATE TABLE virtual_inbox_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NULL,
    message_type VARCHAR(50) NOT NULL,           -- EMAIL, PUSH, SMS, RECEIPT
    recipient VARCHAR(500) NOT NULL,
    subject VARCHAR(500) NULL,
    body TEXT NULL,
    attachment_path VARCHAR(500) NULL,           -- für PDF-Belege
    metadata JSONB NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**UI:** `/demo/inbox`
- Liste aller Nachrichten, sortiert nach Zeitstempel.
- Filter nach Typ, Empfänger.
- Detail-Modal mit Inhalt.
- Bei Receipt: PDF-Vorschau.

## Embedded Caddy (optional, Phase 2)

Im MVP nicht. In Phase 2 KÖNNTE für vereinfachten Standalone-Betrieb ein eingebetteter HTTP-Server mit TLS-Selbstzertifikat ergänzt werden. Aktuell HTTP-only auf localhost im Demo.

## Konfiguration via `application.yml`

Strukturierter Aufbau:

```yaml
# application.yml (gemeinsam)
spring:
  profiles:
    default: demo

# application-demo.yml
spring:
  datasource:
    embedded:
      enabled: true
  session:
    store-type: in-memory
app:
  payment:
    adapter: fake
  tse:
    adapter: fake
  mail:
    adapter: log
  notification:
    adapter: fake
  receipt-printer:
    adapter: pdf
  demo:
    reset-enabled: true
    time-jump-enabled: true

# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/restaurant
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  session:
    store-type: redis
  redis:
    host: ${REDIS_HOST}
app:
  payment:
    adapter: stripe
  tse:
    adapter: fiskaly
  mail:
    adapter: brevo
  notification:
    adapter: twilio
  receipt-printer:
    adapter: escpos
```

## Adapter-Konfiguration (Bedingungsbasiert)

```java
@Configuration
public class PaymentAdapterConfiguration {
    @Bean
    @ConditionalOnProperty(name = "app.payment.adapter", havingValue = "fake")
    public PaymentPort fakePaymentAdapter() {
        return new FakePaymentAdapter();
    }

    @Bean
    @ConditionalOnProperty(name = "app.payment.adapter", havingValue = "stripe")
    public PaymentPort stripePaymentAdapter(StripeConfig config) {
        return new StripePaymentAdapter(config);
    }
}
```

## Profil-Verifikation im Test

Architektur-Test im CI:
```java
@Test
void prodProfileMustNotHaveFakeAdapters() {
    var ctx = new SpringApplicationBuilder(RestaurantApplication.class)
        .profiles("prod")
        .web(WebApplicationType.NONE)
        .properties(...)
        .run();

    assertThat(ctx.getBean(PaymentPort.class))
        .isNotInstanceOf(FakePaymentAdapter.class);
    // gleiches für TSE, Mail, Notification, ReceiptPrinter
}

@Test
void demoEndpointMustNotBeAvailableInProd() {
    // RestAssured: POST /demo/reset → 404 in prod
}
```

## Onboarding für Entwickler / Probierende

Eine `STANDALONE.md` im Repo-Root mit:
1. Java 21 installieren.
2. JAR herunterladen / aus Quellen bauen.
3. `java -jar restaurant-app.jar`.
4. `http://localhost:8080` öffnen.
5. Mit `admin@demo.local / admin` einloggen für Plattform-UI.
6. Mit `inhaber@demo.local / demo` einloggen für Restaurant-UI.
7. Tisch-Bestellung simulieren: `http://localhost:8080/t/<demo-table-token>`.
8. Reset jederzeit über `/demo/reset` oder Demo-Seite.

## Nicht im Standalone-Mode

- Echte Zahlungen
- Echte TSE-Signaturen mit BSI-zertifiziertem Modul
- Echter E-Mail-Versand
- Echte Bondrucker-Anbindung
- Plattform-übergreifender Email-Empfang

Diese werden alle simuliert, klar gekennzeichnet im UI mit Demo-Hinweisen.
