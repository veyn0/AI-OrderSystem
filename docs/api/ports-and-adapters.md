# Ports und Adapter

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Externe Abhängigkeiten sind hinter Ports (Interfaces) gekapselt. Pro Port existieren mindestens zwei Implementierungen: ein **Fake-Adapter** für Demo/Test, ein **Echter Adapter** für Produktion.

## Übersicht

| Port | Modul | Fake-Adapter | Echter Adapter (Phase 2+) |
|---|---|---|---|
| `PaymentPort` | `payment` | `FakePaymentAdapter` | Stripe / Mollie / Adyen |
| `TsePort` | `tse` | `FakeTseAdapter` | fiskaly / Epson-TSE / Swissbit |
| `MailPort` | `notification` | `LogMailAdapter` | Brevo / Mailjet / SES |
| `NotificationPort` | `notification` | `FakeNotificationAdapter` | Twilio (SMS) / FCM (Push) |
| `ReceiptPrinterPort` | `notification` | `PdfReceiptPrinterAdapter` | ESC/POS-Network/USB |
| `AccountingExportPort` | `reporting` | (kein MVP-Adapter) | DATEV-Export-Generator |

## PaymentPort

```java
public interface PaymentPort {

    /**
     * Initiiert eine Zahlung. Gibt eine URL zurück, zu der der Endkunde
     * weitergeleitet werden soll, um die Zahlung abzuschließen.
     */
    PaymentSession initiatePayment(InitiatePaymentRequest request);

    /**
     * Verarbeitet einen Webhook-Callback vom Provider.
     * Aktualisiert den Payment-Status und publiziert ggf. Domain-Events.
     */
    void handleWebhook(WebhookPayload payload);

    /**
     * Initiiert einen Refund.
     */
    RefundResult refund(UUID paymentId, long amountCents, String reason);
}

public record InitiatePaymentRequest(
    UUID tenantId,
    UUID orderId,
    long amountCents,
    String currency,
    URI returnUrl,
    URI webhookUrl,
    Map<String, String> metadata
) {}

public record PaymentSession(
    String providerPaymentId,
    URI redirectUrl,
    Instant expiresAt
) {}

public record RefundResult(
    UUID paymentId,
    boolean succeeded,
    String providerRefundId,
    String failureReason
) {}
```

### FakePaymentAdapter

- Speichert Sessions in-memory (oder kleiner DB-Tabelle).
- `initiatePayment` gibt eine URL zurück, die im Demo auf `/demo/fake-payment/{sessionId}` zeigt — eine simulierte Provider-Seite mit „Zahlen" / „Abbrechen" / „Fehler" Buttons.
- Verzögerung: nach 1-3 s Webhook auslösen.
- Erfolgs-Wahrscheinlichkeit: 95 % konfigurierbar via `app.payment.fake.success-rate`.
- `refund` ist immer erfolgreich.

### Echter Adapter (Phase 2)

Empfohlene Wahl: **Stripe** für Demo/MVP-Markt — sehr gute Doku, Test-Modus, APIs für hosted Checkout.

Alternative: **Mollie** — DACH-fokussiert, oft günstiger.

Konfiguration in Prod: API-Keys via Environment-Variablen, niemals im Code/Config-File.

## TsePort

Vereinfachte Schnittstelle, die der TSE-Schnittstellen-Spezifikation für Kassensysteme folgt.

```java
public interface TsePort {

    /**
     * Erstellt eine TSE-Signatur für eine Bestellung beim PAID-Übergang.
     */
    TseSignatureResult signTransaction(SignTransactionRequest request);

    /**
     * Hol-Status der TSE: ist sie aktiv, hat sie Speicher, ist sie zertifiziert?
     */
    TseStatus getStatus();

    /**
     * Export der TSE-Aufzeichnungen für externe Prüfung (Phase 2 / Echtbetrieb).
     */
    TseExport exportRecords(LocalDate from, LocalDate to);
}

public record SignTransactionRequest(
    UUID tenantId,
    UUID orderId,
    String processType,           // "Kassenbeleg-V1"
    String processData,           // strukturierter Inhalt der Buchung
    Instant transactionStartedAt,
    Instant transactionFinishedAt
) {}

public record TseSignatureResult(
    String signature,             // base64
    long transactionCounter,
    String serialNumber,
    String publicKey,
    boolean isFake                // true bei FakeTseAdapter
) {}
```

### FakeTseAdapter

- Generiert plausibel aussehende Signaturen (random base64, fixe Serial-Number, simulierter Counter).
- Setzt `isFake = true` im Result.
- Schreibt in `tse_signatures` mit `is_fake = true`.
- **UI-Hinweis Pflicht:** Auf Belegen (PDF) und im Backoffice prominent „Demo-TSE — keine BSI-Zertifizierung" einblenden.

### Echter Adapter

Optionen:
- **fiskaly Cloud-TSE** (Software-as-a-Service, REST-API): einfache Integration, monatliche Kosten.
- **Epson TM-m30 mit eingebauter TSE** (Hardware-TSE im Bondrucker): einmalige Kosten, lokale Integration.
- **Swissbit USB-TSE**: günstige Hardware-TSE per USB.

Im MVP NICHT entschieden — Auswahl bei Echtkunden-Go-Live.

## MailPort

```java
public interface MailPort {

    /**
     * Versendet eine E-Mail.
     */
    void sendMail(SendMailRequest request);
}

public record SendMailRequest(
    UUID tenantId,                // NULL bei Plattform-Mails
    String fromAddress,
    String fromName,
    String toAddress,
    String toName,
    String subject,
    String bodyHtml,
    String bodyText,              // Fallback für Text-Only-Clients
    List<MailAttachment> attachments
) {}

public record MailAttachment(
    String filename,
    String mimeType,
    byte[] content
) {}
```

### LogMailAdapter (Demo)

- Schreibt in `virtual_inbox_messages` mit `message_type = EMAIL`.
- Logt zusätzlich auf Console („Mail to user@example.com: 'Bestätigung Bestellung #123'").
- Kein Versand an reale E-Mail-Server.

### Echter Adapter

Optionen:
- **Brevo** (vormals Sendinblue): EU-Hosting, 300 Mails/Tag kostenlos, dann günstige Tarife.
- **Mailjet**: ähnlich, Preis vergleichbar.
- **AWS SES**: günstig pro Mail, aber DSGVO-Komplexität (US-Anbieter).

**Empfehlung Phase 2:** Brevo (DACH-Konformität).

## NotificationPort

Für Push-Notifications (Mobile-App) und SMS.

```java
public interface NotificationPort {

    void sendPush(SendPushRequest request);

    void sendSms(SendSmsRequest request);
}

public record SendPushRequest(
    UUID tenantId,
    String deviceToken,           // FCM-Token
    String title,
    String body,
    Map<String, String> data
) {}

public record SendSmsRequest(
    UUID tenantId,
    String toPhone,
    String body
) {}
```

### FakeNotificationAdapter (Demo)

- Schreibt in virtuelles Postfach.
- Im MVP eher Platzhalter; Push/SMS sind in Phase 2 relevant.

## ReceiptPrinterPort

```java
public interface ReceiptPrinterPort {

    /**
     * Druckt einen Beleg.
     */
    PrintResult printReceipt(PrintReceiptRequest request);
}

public record PrintReceiptRequest(
    UUID tenantId,
    UUID orderId,
    UUID locationId,
    ReceiptType type,             // CUSTOMER_RECEIPT, KITCHEN_TICKET
    ReceiptContent content
) {}

public record ReceiptContent(
    String header,                // Restaurant-Name, Adresse, USt-ID
    List<ReceiptLine> lines,
    String footer,                // TSE-Daten, Datum, etc.
    Map<String, Object> metadata
) {}

public record PrintResult(
    boolean succeeded,
    String virtualReceiptUrl      // bei PDF-Adapter
) {}
```

### PdfReceiptPrinterAdapter (Demo, MVP)

- Generiert PDF via Apache PDFBox.
- Speichert in `virtual_inbox_messages` mit `message_type = RECEIPT`, `attachment_path` zeigt auf PDF.
- UI im virtuellen Postfach zeigt PDF-Vorschau.

### Echter Adapter (Phase 2)

- ESC/POS über TCP/IP an Bondrucker im Restaurant (z.B. Epson TM-m30, Star TSP100).
- Adapter sendet ESC/POS-Bytes über Socket an konfigurierte IP:Port.
- Konfiguration pro Standort: IP, Port, Drucker-Modell, Zeichensatz.

## AccountingExportPort (Phase 2)

```java
public interface AccountingExportPort {
    DatevExport generateDatevExport(UUID tenantId, LocalDate from, LocalDate to);
}

public record DatevExport(
    byte[] csvContent,
    String filename,
    long entryCount
) {}
```

Im MVP nicht implementiert — Stub-Klasse, die `UnsupportedOperationException` wirft.

## Adapter-Konfiguration

Spring `@ConditionalOnProperty`:

```java
@Bean
@ConditionalOnProperty(name = "app.payment.adapter", havingValue = "fake")
public PaymentPort fakePayment() { return new FakePaymentAdapter(); }

@Bean
@ConditionalOnProperty(name = "app.payment.adapter", havingValue = "stripe")
public PaymentPort stripe(StripeConfig cfg) { return new StripePaymentAdapter(cfg); }
```

Profil-Default-Mapping siehe [`../architecture/standalone-mode.md`](../architecture/standalone-mode.md).

## Einbindung in der Domäne

Die Domänenlogik kennt nur den Port:

```java
@Service
public class OrderChargeService {
    private final PaymentPort payment;
    private final TsePort tse;
    private final ReceiptPrinterPort receiptPrinter;

    public ChargeResult chargeOrder(UUID orderId, ChargeRequest req) {
        // ... Validierung ...
        var paymentResult = payment.initiatePayment(...);
        if (paymentResult.succeeded()) {
            var tseResult = tse.signTransaction(...);
            var receipt = receiptPrinter.printReceipt(...);
            // ... persistieren ...
        }
        // ...
    }
}
```

Tests verwenden Mockito-Mocks oder Test-Fakes; reale Adapter laufen nur in Integrations-/E2E-Tests gegen Stage-Umgebungen.

## Webhook-Endpoints

Echte Adapter benötigen Webhooks vom Provider:

```
POST /api/v1/webhooks/payment/stripe
POST /api/v1/webhooks/tse/fiskaly
```

**Sicherheits-Anforderungen:**
- HMAC-Signatur-Validierung mit Provider-Secret.
- Idempotenz (Provider können Webhooks mehrfach senden).
- Antwort 200 nur bei erfolgreicher Verarbeitung. Bei Fehler 5xx → Provider versucht erneut.
- Webhooks werden NICHT durch Standard-Auth abgesichert (Provider kennt kein Session-Cookie); Schutz erfolgt rein über HMAC.

## Test-Strategie pro Port

- **Unit-Tests** der Domänenlogik mit Mockito-Mocks.
- **Adapter-Tests** für Fakes: einfache Verifikation des Verhaltens.
- **Adapter-Tests** für Echte Adapter: gegen Sandbox-Endpoints des Providers (Stripe Test-Mode, fiskaly Sandbox, etc.). Diese laufen im CI nur, wenn Sandbox-Credentials konfiguriert sind.
- **Contract-Tests** (Phase 2): WireMock-Stubs sichern API-Verträge ab.
