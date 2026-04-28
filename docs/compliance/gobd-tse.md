# GoBD und KassenSichV / TSE

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28
> **Hinweis:** Dieses Dokument ist eine technische Spezifikation, KEINE Steuer- oder Rechtsberatung. Vor Echtkunden-Go-Live MUSS ein Steuerberater konsultiert werden.

## Was geregelt ist

**GoBD** = Grundsätze zur ordnungsmäßigen Führung und Aufbewahrung von Büchern. Vom BMF erlassen, regelt steuerrelevante Datenaufzeichnung und -aufbewahrung.

**KassenSichV** = Kassensicherungsverordnung. Verlangt seit 2020 in Deutschland, dass elektronische Aufzeichnungssysteme (sprich: Kassen) eine **zertifizierte Technische Sicherheitseinrichtung (TSE)** verwenden, die jeden Bargeschäfts-Vorgang signiert.

**AO § 146a** = Abgabenordnung-Paragraph, der Aufzeichnungspflichten elektronischer Kassen verlangt.

## GoBD-Anforderungen

### Unveränderbarkeit
Aufgezeichnete Geschäftsvorfälle dürfen nicht im Nachhinein verändert werden, ohne dass die Änderung nachvollziehbar bleibt.

**Umsetzung in der Plattform:**
- `OrderItem.priceSnapshot` ist unveränderlich nach erstem Speichern.
- Stornierte Items werden nicht gelöscht, sondern auf `CANCELLED` mit Begründung gesetzt.
- Steuersatz-Overrides werden auditiert, nie still überschrieben.
- `financial_audit_log` ist append-only auf DB-Ebene.
- TSE-Signatur fixiert die Order zum Zeitpunkt PAID.

### Vollständigkeit
Alle Geschäftsvorfälle MÜSSEN aufgezeichnet werden — keine „Schubladen-Bestellungen" ohne Beleg.

**Umsetzung:**
- Jede Order, die in `PAID` übergeht, erzeugt zwingend einen TSE-Signature-Eintrag.
- Bestellungen können nicht „verschwinden" — Soft-Delete erfolgt nicht für Order, sondern Status `CANCELLED`/`REJECTED` mit Begründung.

### Richtigkeit
Aufzeichnungen müssen sachlich korrekt sein.

**Umsetzung:**
- Steuersätze werden automatisch aus Bestelltyp und Artikel-Konfiguration ermittelt.
- Manuelle Override (`ORDER_TAX_OVERRIDE`) erfordert Permission und Begründung.
- Geld-Beträge in Cent, keine Float-Rundungsfehler.

### Zeitgerechtheit
Aufzeichnung MUSS zeitnah zum Geschäftsvorfall erfolgen.

**Umsetzung:**
- Order-Lifecycle-Transitionen werden mit Timestamps gesetzt, sobald sie passieren.
- TSE-Signatur erfolgt synchron beim PAID-Übergang.

### Ordnung
Aufzeichnungen müssen geordnet und systematisch sein.

**Umsetzung:**
- Datenmodell-Struktur ist dokumentiert.
- Aggregate-Grenzen sind klar.
- Audit-Log ist chronologisch sortierbar.

### Aufbewahrung
**10 Jahre** für steuerrelevante Daten.

**Umsetzung:**
- `financial_audit_log` mit 10-Jahres-Retention-Job.
- Backups werden so betrieben, dass auch nach Disaster die Daten verfügbar bleiben.
- Bei Tenant-Löschung: Soft-Delete, Daten bleiben mit `deletedAt` markiert über die 10-Jahres-Frist.

### Datenzugriff durch Finanzbehörden (Z1, Z2, Z3)
- **Z1:** Lese-Zugriff durch Prüfer auf das Kassensystem.
- **Z2:** Anforderung maschinell auswertbarer Daten.
- **Z3:** Datenträgerüberlassung (Standard).

**Umsetzung:**
- Restaurant-Inhaber kann auf Anfrage Daten als CSV/JSON exportieren (DATEV-Export Phase 2).
- Plattform-Owner stellt im Bedarfsfall (Steuerprüfung) IT-Zugriff für berechtigte Prüfer bereit (Z1).

## TSE / KassenSichV-Anforderungen

### Zertifizierte TSE
Pflicht für jede Kasse, die in Deutschland Bargeschäfte aufzeichnet. Die TSE signiert jeden Vorgang mit einem privatem Schlüssel; die Signatur ist Teil des Belegs.

**Eine TSE besteht aus:**
- Sicherheitsmodul (signiert)
- Speichermedium (speichert Aufzeichnungen)
- Schnittstelle für Kasse

### Belegausgabepflicht
Jede Bestellung MUSS einen Beleg erzeugen — auf Papier oder elektronisch (Endkunde stimmt zu).

**Umsetzung:**
- `ReceiptPrinterPort` druckt Beleg bei jedem PAID-Übergang.
- Im MVP: PDF im virtuellen Postfach (Demo), in Prod ESC/POS oder PDF per E-Mail an Endkunden.

### Beleg-Mindestinhalt (KassenSichV § 6)

Der Beleg MUSS enthalten:
1. Vollständiger Name + Anschrift des leistenden Unternehmers (Tenant).
2. Datum der Belegausstellung.
3. Zeitpunkt des Vorgangsbeginns + Vorgangsende.
4. Menge und Art der gelieferten Gegenstände / Leistungsumfang.
5. Transaktionsnummer.
6. Entgelt + Steuerbeträge in einer Summe sowie aufgeschlüsselt nach Steuersätzen.
7. Seriennummer der Kasse.
8. Seriennummer der TSE.
9. **Prüfwert (Signatur) der TSE.**
10. Signaturzähler der TSE.

**Umsetzung in `PdfReceiptPrinterAdapter`:**
- Generiert PDF, das alle obigen Punkte enthält.
- Im Demo: Hinweis „Demo-Beleg, nicht KassenSichV-konform".
- In Prod (mit echter TSE): voll-konformer Beleg.

### Signatur-Inkrement
Die TSE führt einen monoton steigenden Counter. Lücken in der Sequenz wären verdächtig.

**Umsetzung:**
- `tse_signatures.transaction_counter` wird vom TSE-Provider gesetzt (im FakeAdapter simuliert).
- Auswertung im Audit zeigt eventuelle Lücken (Phase 2).

### TSE-Ausfall
Falls die TSE ausfällt:
- Bestellungen MÜSSEN trotzdem entgegengenommen und bedient werden.
- Aufzeichnung erfolgt manuell (Papier).
- Sobald TSE wieder läuft: Nacherfassung.

**Umsetzung:**
- Im MVP: kein TSE-Ausfall-Modus implementiert (FakeTseAdapter ist immer verfügbar).
- Vor Echtkunden-Go-Live: Behandlung von TSE-Provider-Fehlern, Fallback-Pfad.

## Architektur in der Plattform

### TsePort

Detail siehe [`../api/ports-and-adapters.md`](../api/ports-and-adapters.md).

```java
public interface TsePort {
    TseSignatureResult signTransaction(SignTransactionRequest request);
    TseStatus getStatus();
    TseExport exportRecords(LocalDate from, LocalDate to);
}
```

### Aufruf-Zeitpunkt
Beim Übergang `Order` von `SERVED|PICKED_UP|DELIVERED` → `PAID`:

1. `OrderService.chargeOrder(...)` validiert, ruft `PaymentPort` für Zahlung.
2. Bei Zahlungserfolg: `TsePort.signTransaction(...)` synchron.
3. Signatur wird in `tse_signatures` gespeichert.
4. `Order.tse_signature_id` wird gesetzt, Order auf `PAID`.
5. `ReceiptPrinterPort.printReceipt(...)` mit TSE-Daten.
6. `OrderPaidEvent` wird publiziert (asynchrones Reporting etc.).

**Pflicht:** Wenn TSE-Signatur fehlschlägt, MUSS Order NICHT in `PAID` übergehen. Stattdessen Fehlerseite an Personal mit Hinweis und manuellem Workaround.

### Daten in `tse_signatures`

```sql
CREATE TABLE tse_signatures (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    order_id UUID NOT NULL,
    signature TEXT NOT NULL,                 -- base64 der TSE-Signatur
    transaction_counter BIGINT NOT NULL,
    transaction_started_at TIMESTAMPTZ NOT NULL,
    transaction_finished_at TIMESTAMPTZ NOT NULL,
    process_data TEXT NOT NULL,              -- strukturierter Inhalt
    serial_number VARCHAR(255) NOT NULL,
    public_key TEXT NOT NULL,
    is_fake BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`is_fake` markiert Signaturen vom Stub. UI im Backoffice zeigt diese mit Warnhinweis.

### TSE-Provider-Auswahl

**Im MVP:** keine Entscheidung. Stub aktiv.

**Vor Echtkunden-Go-Live:** Auswahl. Optionen:

| Provider | Typ | Vorteile | Nachteile |
|---|---|---|---|
| **fiskaly** | Cloud-TSE | Reine REST-API, keine Hardware, schnelle Integration | Monatliche Kosten pro Restaurant; Internet-Abhängigkeit |
| **Epson TM-m30 mit eingebauter TSE** | Hardware-TSE im Bondrucker | Einmalige Kosten, lokale Signierung | Hardware muss vom Restaurant gekauft + verwaltet werden |
| **Swissbit USB-TSE** | USB-Stick | Günstig | Verwaltungs-Overhead |

**Architektur ist so vorbereitet**, dass mehrere Adapter koexistieren und der Tenant pro Standort wählen kann.

## Steuersatz-Logik

Deutschland kennt für Gastronomie aktuell zwei Steuersätze:
- **7 % USt** (Speisen außer Haus, Lieferung, Abholung)
- **19 % USt** (Speisen vor Ort, Getränke, Alkohol)

**Umsetzung:**
- Pro `MenuItem`: zwei Steuersätze (`tax_rate_on_site`, `tax_rate_takeaway_delivery`).
- Bei Bestellanlage wird basierend auf `Order.orderType` der passende Satz gewählt.
- Snapshot pro `OrderItem.tax_rate_snapshot`.
- Manueller Override mit Permission `ORDER_TAX_OVERRIDE` und Pflicht-Begründung.

**Sonderfall Getränke außer Haus:** in DE 19 % auch außer Haus. Daher pro MenuItem flexibel konfigurierbar — bei Cola sind beide Sätze 19 %.

**Sonderfall „eigentlich Lieferung, aber Gast bleibt":** Personal kann Steuer-Override durchführen.

## Belegformat (PDF)

Bei `PdfReceiptPrinterAdapter` enthält das PDF (im Demo, später im Prod identisch):

```
+----------------------------------------+
| Pizzeria Mario                         |
| Hauptstraße 12, 10115 Berlin           |
| USt-ID: DE123456789                    |
+----------------------------------------+
| Datum:    28.04.2026 13:24             |
| Beleg-Nr: 2026-04-28-00123             |
| TX-Nr:    8f3a-...                     |
+----------------------------------------+
| 1x Pizza Margherita Mittel       11,00 |
|    + Extra Käse                  +1,00 |
| 1x Cola 0,5l                      3,50 |
+----------------------------------------+
| Zwischensumme                    15,50 |
| MwSt 19% auf 15,50 EUR            2,47 |
| Brutto-Summe                     15,50 |
+----------------------------------------+
| Zahlung: Bar                     15,50 |
| Erhalten:                        20,00 |
| Rückgeld:                         4,50 |
+----------------------------------------+
| TSE-Seriennummer:    SN-12345          |
| TSE-Counter:         98765             |
| TSE-Signatur:        eyJh...           |
| Vorgang gestartet:   13:23:45          |
| Vorgang beendet:     13:24:01          |
+----------------------------------------+
| Vielen Dank für Ihren Besuch!          |
+----------------------------------------+
```

Im Demo wird zusätzlich ein „DEMO-MODUS" Wasserzeichen eingeblendet.

## Aufzeichnungs- und Aufbewahrungs-Pflicht im Detail

| Datentyp | GoBD-Pflicht | Aufbewahrung | Umsetzung |
|---|---|---|---|
| Bestelldaten (Order) | ✓ | 10 Jahre | `orders` Soft-Delete, nie hart |
| Belege | ✓ | 10 Jahre | PDF-Archiv in Phase 2 |
| TSE-Signaturen | ✓ | 10 Jahre | `tse_signatures` |
| Stornos mit Begründung | ✓ | 10 Jahre | `financial_audit_log` |
| Steuer-Overrides | ✓ | 10 Jahre | `financial_audit_log` |
| Konfigurations-Änderungen Steuersatz | ✓ | 10 Jahre | `audit_log` (genauer prüfen) |

**Hinweis zur Konfiguration:** Änderungen am Steuersatz eines `MenuItem` sind potenziell steuerrelevant (Wechsel von 7 % zu 19 % wäre auffällig). Diese Änderungen werden im allgemeinen Audit-Log protokolliert (90 Tage). **Vor Echtkunden-Go-Live evaluieren**, ob diese in `financial_audit_log` gehören.

## DATEV-Export (Phase 2)

`AccountingExportPort.generateDatevExport(tenantId, from, to)`:
- Erzeugt CSV im DATEV-Format.
- Eine Zeile pro Buchung (Bestellung, ggf. aufgesplittet nach Steuersatz).
- Konten gemäß Standardkontenrahmen SKR03 / SKR04 (konfigurierbar).

Im MVP nicht implementiert. Stub wirft `UnsupportedOperationException`.

## Risiko-Hinweis

**Plattform mit Fake-TSE darf NICHT für reale Bargeschäfte in DE eingesetzt werden.** Das wäre eine Ordnungswidrigkeit (Bußgeld) und im Schadensfall (Steuerprüfung) deutlich schlimmer.

**Daher:** Echtkunden-Onboarding ist erst zulässig, nachdem ein echter, BSI-zertifizierter TSE-Adapter integriert ist.
