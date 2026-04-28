# ADR-018: Tisch-QR-Code Anti-Missbrauch

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Im MVP unterstützt die Plattform **Tisch-Bestellungen vom Gast-Smartphone** über einen QR-Code-Aufkleber am Tisch. Der QR enthält einen Link zu `<restaurant>/t/<tableToken>`, was eine kontextualisierte Online-Bestellseite öffnet, die die Bestellung diesem Tisch zuordnet.

**Missbrauchs-Szenarien:**

1. Person fotografiert QR ab, geht nach Hause, bestellt von dort und schiebt Kosten an einen aktuellen Tischgast.
2. Bot schickt aus Spaß tausende Kellner-Rufe.
3. Konkurrenz-Restaurant erstellt 50 Bestellungen, bricht sie ab → Stress-Operations.
4. Person sitzt am Tisch, bestellt, rennt weg, ohne zu zahlen.

Wie schützen?

## Entscheidung

**Permanenter QR-Token + Rate-Limiting + Personal-Sicht + Block-Funktion.**

- **Tisch-QR ist permanent** (`tables.qr_token`, einmal generiert, nicht rotiert).
- **Rate-Limit pro Tisch:** maximale Anzahl Kellner-Rufe (z.B. 1 pro 3 min), maximale Anzahl Bestell-Initiations.
- **Personal-Sicht:** Bestelltablet hat „Aktive Tisch-Bestellungen"-View, wo alle laufenden Tisch-Bestellungen + offene Kellner-Rufe sichtbar sind.
- **Block-Funktion:** Personal kann einen Tisch temporär für Tisch-QR-Aktionen blocken (`table_call_blocks`), z.B. für 1 h.
- **IP-Fingerprint-Logging** bei Kellner-Rufen, um Missbrauch nachvollziehen zu können.

**KEIN Geofence-Check.** **KEINE rotierenden QR-Sticker.**

## Alternativen

### Geofencing
Verworfen.
- Browser-Geolocation ist unzuverlässig (User kann verweigern, Indoor-GPS schwach).
- Standort-Spoofing trivial.
- UX-Hindernis für legitime Nutzer.

### Rotierende QR-Codes
Verworfen.
- Logistik: Aufkleber müssten regelmäßig getauscht werden.
- Hardware-Lösung mit eInk-Display am Tisch wäre teuer.
- Selbst rotierende Codes können fotografiert werden.

### NFC am Tisch (statt QR)
Erwogen, verworfen für MVP.
- Hardware-Investitionen für Restaurant.
- Nicht alle Smartphones haben NFC oder es ist zugänglich genug.

### Kein Tisch-QR-Bestellen, nur Personal über Tablet
Erwogen, abgelehnt — eine Hauptattraktion der Plattform für moderne Restaurants entfällt.

### Pre-Authorization mit Anzahlung beim Self-Order
Erwogen, verworfen für MVP.
- UX-Hürde.
- Komplexer Refund-Flow bei Stornos.
- Phase 2 erwägen für SELF_ORDER_INSTANT-Modus.

## Konsequenzen

### Positiv
- **Einfach zu implementieren** — Tabellen + Rate-Limit-Filter.
- **Keine Hardware-Investitionen** für Restaurants.
- **Personal hat Kontrolle** — sieht aktive Bestellungen, kann eingreifen.
- **Audit-Spuren**: IP, Zeitstempel — auch wenn Anonymität teilweise besteht, gibt es Diagnose-Material.

### Negativ
- **Restrisiko Stalking-Bestellungen aus der Ferne**: Persona, die QR abfotografiert, kann technisch von außerhalb bestellen. Mitigation:
  - Bestelltablet zeigt aktive Tisch-Bestellungen → Personal sieht „Tisch 7 hat eine offene Bestellung", wenn dort niemand sitzt → Verdacht-Indiz.
  - Bestellungen werden auf Tisch attribuiert → echter Schaden ist begrenzt (Tisch zahlt am Ende, Personal erkennt Diskrepanz).
- **Restrisiko Bot-Spam**: Rate-Limit dämpft, aber ein verteilter Angriff könnte trotzdem stören. Mitigation: Captcha bei verdächtigem Verhalten (Phase 2).

### Neutral
- Tisch-QR-Token wird bei `TABLE_DELETE` invalidiert; bei Tisch-Tausch ggf. neuer Token nötig.

## Operative Hinweise für Restaurants

- QR-Aufkleber sollten an gut sichtbarer, aber nicht abreißbarer Stelle (z.B. unter Glas eingeklebt).
- Personal trainiert auf „aktive Tisch-Bestellungen"-Sicht.
- Bei Vorfall: Tisch temporär blocken, Vorfall in Audit-Log dokumentieren.

## Phase 2 / 3 Erweiterungen

- **Captcha** bei wiederholten verdächtigen Aktionen.
- **Pre-Authorization** für SELF_ORDER_INSTANT.
- **NFC-Option** für Tische, die das wünschen.
- **Tisch-Heatmap der Aktivität** im Reporting (welche Tische haben am meisten Aktivität).

## Querverweise

- [`../domain/use-cases.md`](../domain/use-cases.md) (Tisch-Bestellung Use Cases)
- [`../api/api-design-principles.md`](../api/api-design-principles.md) (Rate-Limiting)
- [`../risks.md`](../risks.md)
