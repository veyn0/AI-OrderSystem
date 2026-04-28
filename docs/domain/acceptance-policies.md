# Annahme-Policy für Online-Bestellungen

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Annahme-Policy regelt, wie ein Standort mit eingehenden Online-Bestellungen umgeht. Dieses Dokument spezifiziert das vollständige Modell.

## Modell

```
AcceptancePolicy
  id: UUID
  tenantId: UUID
  locationId: UUID
  rules: List<AcceptanceRule>          # geordnete Regeln, erste passende gilt
  defaultMode: AcceptanceMode          # falls keine Regel passt
  manualOverride: ManualOverride?      # temporärer Override
  updatedBy, updatedAt

AcceptanceRule
  conditionType: ENUM (ALWAYS, BY_TIME, BY_WORKLOAD, BY_DELIVERY_ETA, BY_ORDER_TYPE)
  conditionParams: JSON                # je nach conditionType
  mode: AcceptanceMode
  acceptanceWindowSeconds: Int?        # nur bei MANUAL_*

AcceptanceMode = ENUM:
  AUTO                                  # Bestellung wird automatisch akzeptiert
  MANUAL_ACCEPT                         # Bestellung wartet, wird verworfen wenn nicht aktiv akzeptiert
  MANUAL_REJECT_WINDOW                  # Bestellung wird vorläufig akzeptiert, kann im Zeitfenster abgelehnt werden

ManualOverride
  mode: AcceptanceMode
  reason: String
  startedAt: Timestamp
  validUntil: Timestamp                 # z.B. "für 3h nicht automatisch annehmen"
  setBy: UUID (User.id)
```

## Bedingungstypen

### ALWAYS
Trifft immer zu. Wird typischerweise als letzte Regel oder als einzige Regel verwendet.
```json
{ "conditionType": "ALWAYS" }
```

### BY_TIME
Trifft zu, wenn die aktuelle Tageszeit innerhalb eines konfigurierten Zeitfensters liegt.
```json
{
  "conditionType": "BY_TIME",
  "weekly": {
    "FRIDAY":   [{"start": "18:00", "end": "23:00"}],
    "SATURDAY": [{"start": "12:00", "end": "23:00"}]
  }
}
```

### BY_WORKLOAD
Trifft zu, wenn aktueller Auslastungs-Status einer Bedingung entspricht.
```json
{
  "conditionType": "BY_WORKLOAD",
  "operator": "GREATER_THAN_OR_EQUAL",
  "threshold": "HIGH"
}
```
**Workload-Bestimmung:** Anzahl Order in `IN_PROGRESS` für diesen Standort:
- 0 – `lowThreshold` → LOW
- `lowThreshold` – `highThreshold` → MEDIUM
- > `highThreshold` → HIGH

`lowThreshold` und `highThreshold` werden pro Standort konfiguriert (Default: 5 / 15).

Manueller Override durch Personal: Schieberegler im UI. Manueller Wert hat Vorrang vor automatischer Berechnung, läuft nach konfigurierbarer Zeit ab oder wird manuell zurückgesetzt.

### BY_DELIVERY_ETA
Trifft zu, wenn die geschätzte Lieferzeit einer Bedingung entspricht.
```json
{
  "conditionType": "BY_DELIVERY_ETA",
  "operator": "GREATER_THAN",
  "minutes": 45
}
```
**ETA-Berechnung:** Im MVP simple Schätzung:
- Zubereitungszeit-Default 20 min × Workload-Faktor (LOW: 1.0, MEDIUM: 1.3, HIGH: 1.7)
- Lieferweg-Default 15 min für DELIVERY (statisch im MVP, in Phase 3 ggf. Distanz-basiert)

### BY_ORDER_TYPE
Trifft zu, wenn die Bestellung einem bestimmten Typ entspricht.
```json
{
  "conditionType": "BY_ORDER_TYPE",
  "orderTypes": ["DELIVERY"]
}
```

## Regel-Auswertung

1. `manualOverride` wird zuerst geprüft. Falls aktiv und nicht abgelaufen → Override-Mode wird verwendet.
2. Sonst: Regeln in `rules`-Liste werden in Reihenfolge geprüft. Erste passende Regel bestimmt den Modus.
3. Falls keine Regel passt → `defaultMode`.

## Konkrete Annahme-Modi

### AUTO
Bestellung wird sofort `ACCEPTED`, kein Personal-Eingriff. SSE-Push an KDS.
Mail an Endkunde: „Bestellung angenommen, voraussichtlich fertig um <Zeit>."

### MANUAL_ACCEPT (mit Zeitfenster)
1. Bestellung wechselt auf `AWAITING_ACCEPTANCE`.
2. SSE-Push an Personal-Geräte (Acceptance-Inbox).
3. Countdown läuft (z.B. 120 s).
4. Personal akzeptiert manuell: → `ACCEPTED` → KDS.
5. Personal lehnt ab: → `REJECTED` mit Pflichtbegründung. Mail an Endkunde, Refund-Stub.
6. Timer läuft ab ohne Aktion: → `REJECTED` mit System-Begründung „Zeitüberschreitung". Refund-Stub.

**Endkunden-Sicht:** Bestätigungsseite zeigt „Bestellung wird vom Restaurant geprüft, du erhältst Bescheid."

### MANUAL_REJECT_WINDOW
1. Bestellung wechselt sofort auf `ACCEPTED` (provisorisch).
2. SSE-Push an Personal-Geräte mit Ablehn-Möglichkeit + Countdown (z.B. 120 s).
3. Personal lehnt im Zeitfenster ab: → `REJECTED` mit Begründung. Mail an Endkunde, Refund-Stub. KDS-Auftrag wird widerrufen (falls schon angefangen, manuelle Klärung).
4. Timer läuft ab: Bestellung bleibt `ACCEPTED`, KDS hat sie ohnehin schon im Workflow.

**Endkunden-Sicht:** Bestätigungsseite zeigt „Bestellung angenommen, voraussichtlich fertig um <Zeit>." (kein Hinweis auf Ablehn-Möglichkeit, das ist intern.)

## UI-Konfiguration

Im Restaurant-Admin-UI „Standort > Annahme-Policy":

```
[Modus-Editor]
+ Regel hinzufügen
  Bedingung: [Dropdown: Immer / Zeit / Auslastung / Lieferzeit / Typ]
  Parameter: [je nach Bedingung]
  Modus:    [Auto / Manuell-Annehmen / Manuell-Ablehnen-im-Zeitfenster]
  Zeitfenster: [Sekunden, nur bei MANUAL_*]

[Liste der Regeln, drag-sortierbar]

[Default-Modus]:  [Auto / Manuell-Annehmen / Manuell-Ablehnen-im-Zeitfenster]

[Manueller Override]
[ ] Aktiv
Modus: [...]
Begründung: [...]
Gültig bis: [Datum/Zeit]
```

## Beispiel-Konfigurationen

### Pizzeria mit unterschiedlicher Strategie nach Tageszeit

```
Rules:
1. BY_WORKLOAD (>= HIGH)         → MANUAL_ACCEPT (120s)
2. BY_TIME (Fr 18-22, Sa 12-23) → MANUAL_REJECT_WINDOW (60s)
3. ALWAYS                        → AUTO
defaultMode: AUTO
```

Bedeutung:
- Wenn Küche überlastet: Personal muss aktiv akzeptieren (sonst werden Bestellungen abgelehnt).
- Stoßzeiten am Wochenende: Bestellung wird sofort angenommen, aber Personal hat 60s Veto-Möglichkeit.
- Sonst: voll-automatisch.

### Imbiss mit kleiner Belegschaft

```
Rules:
1. BY_DELIVERY_ETA (> 45 min)   → MANUAL_ACCEPT (300s)
2. ALWAYS                       → AUTO
defaultMode: AUTO
```

### Restaurant mit zentraler Steuerung

```
Rules:
1. ALWAYS → MANUAL_ACCEPT (180s)
defaultMode: MANUAL_ACCEPT
```

## Auditing

Jede Policy-Änderung MUSS auditiert werden mit Diff (alter / neuer Inhalt). Manuelle Overrides werden separat als `ACCEPTANCE_OVERRIDE_SET` / `ACCEPTANCE_OVERRIDE_CLEARED` protokolliert.

Annahme-Entscheidungen pro Bestellung werden im `OrderAuditLog` als `ACCEPTANCE_DECISION` mit Details zu angewendeter Regel + Modus gespeichert.

## Erweiterungsmöglichkeiten (Phase 2+)

- **Lerneffekt-Modus:** System schlägt anhand historischer Daten neue Schwellen vor.
- **Spitzenlast-Profile:** vorbereitete Konfigurationen, die mit einem Klick aktiviert werden („Sportevent-Modus").
- **Auto-Workload-Anpassung:** automatische Aktualisierung nach Real-Zubereitungs-Zeiten.

Diese sind im MVP nicht enthalten.
