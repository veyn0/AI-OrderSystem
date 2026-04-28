# Bestell-Lifecycle (Zustandsmaschine)

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument spezifiziert die exakten Zustände und erlaubten Übergänge der Aggregate `Order` und `OrderItem`. Die Implementierung MUSS dieses Modell strikt einhalten — keine Sprünge, keine zusätzlichen Zustände, keine impliziten Übergänge.

## Order-Zustände

```
DRAFT
  |
  | (Self-Order: nach erfolgreicher Zahlung)
  | (Tisch:     nach "An Küche senden")
  v
CONFIRMED
  |
  | (Annahme-Policy entscheidet)
  +---> AWAITING_ACCEPTANCE  --(akzeptiert)--> ACCEPTED
  |       |                  --(timeout/abgelehnt)--> REJECTED
  |       v
  +---> ACCEPTED (provisorisch bei ManuellAblehnung)
  |       |
  |       | (Zeitfenster läuft ab oder explizit akzeptiert)
  |       v
  +-----> IN_PROGRESS
            |
            | (Items werden zubereitet; pro Item eigener Status)
            v
          READY
            |
            +-- bei ON_SITE: -----------> SERVED ---> PAID
            +-- bei PICKUP:  -----------> PICKED_UP ---> PAID  (oder PAID schon vorher)
            +-- bei DELIVERY:-----> OUT_FOR_DELIVERY
                                          |
                                          v
                                      DELIVERED ---> PAID

  Sonderzustand:
  CANCELLED          (vor IN_PROGRESS storniert)
  REJECTED           (von Standort abgelehnt; vor IN_PROGRESS)
  NEEDS_RESOLUTION   (Kitchen rejected ein Item einer bezahlten Bestellung)
```

## Erlaubte State-Transitionen (Order)

Jede Transition ist eine Zeile. Andere Übergänge sind **VERBOTEN** und MÜSSEN zur Laufzeit als IllegalStateTransitionException abgelehnt werden.

| Von | Nach | Trigger | Voraussetzung |
|---|---|---|---|
| (neu) | DRAFT | `OrderService.createOrder(...)` | — |
| DRAFT | CONFIRMED | Self-Order: PaymentSucceeded; Tisch: "An Küche" | Bei Self-Order: Payment.SUCCEEDED |
| DRAFT | CANCELLED | User/System abbricht vor Confirm | — |
| CONFIRMED | AWAITING_ACCEPTANCE | Annahme-Policy = ManuellAnnahme | Online-Bestellung |
| CONFIRMED | ACCEPTED | Annahme-Policy = AutoAnnahme oder ManuellAblehnung | — |
| CONFIRMED | REJECTED | Manuelle Ablehnung vor Annahme | Permission ONLINE_ORDER_ACCEPT_REJECT |
| AWAITING_ACCEPTANCE | ACCEPTED | Personal akzeptiert manuell | Permission |
| AWAITING_ACCEPTANCE | REJECTED | Personal lehnt ab oder Timer läuft ab | — |
| ACCEPTED | IN_PROGRESS | Erste Item-State-Transition zu IN_PREPARATION | automatisch |
| ACCEPTED | REJECTED | ManuellAblehnung-Modus: Personal lehnt im Zeitfenster ab | Permission |
| ACCEPTED | CANCELLED | Storno vor Küchenarbeit | Permission ORDER_CANCEL_PRE_KITCHEN |
| IN_PROGRESS | READY | Alle aktiven Items sind READY | automatisch |
| IN_PROGRESS | NEEDS_RESOLUTION | Item REJECTED_BY_KITCHEN bei bezahlter Order | automatisch |
| IN_PROGRESS | CANCELLED | Storno mit Manager-PIN | Permission ORDER_CANCEL_POST_KITCHEN |
| READY | SERVED | ON_SITE: Personal bestätigt "ausgeliefert" | Permission |
| READY | PICKED_UP | PICKUP: Personal bestätigt Übergabe | Permission |
| READY | OUT_FOR_DELIVERY | DELIVERY: Fahrer übernimmt | Permission, Phase 3 |
| OUT_FOR_DELIVERY | DELIVERED | Fahrer bestätigt | Phase 3 |
| SERVED \| PICKED_UP \| DELIVERED | PAID | Zahlung erfolgreich | Permission ORDER_CHARGE oder Online-Zahlung |
| NEEDS_RESOLUTION | (manuelle Auflösung → CANCELLED oder weiter) | Personal-Aktion | — |

**Spezialregel SELF_ORDER_INSTANT:**
Bei `orderMode=SELF_ORDER_INSTANT` durchläuft die Bestellung `DRAFT → CONFIRMED → ACCEPTED → IN_PROGRESS` (ggf. mit AWAITING_ACCEPTANCE dazwischen) **innerhalb derselben Service-Operation** atomar bis ACCEPTED. Es gibt für den Endkunden keine sichtbare DRAFT-Phase nach Zahlung.

**Spezialregel OPEN_TAB:**
Bei `orderMode=OPEN_TAB` ist `IN_PROGRESS` der Standardzustand für eine offene Tisch-Bestellung. **Neue Items können hinzugefügt werden, solange Order in IN_PROGRESS** (oder ACCEPTED) ist. Items haben dann eigene Lifecycle-Stufen, Order bleibt im IN_PROGRESS bis abgerechnet wird.

## OrderItem-Zustände

```
REQUESTED
  |
  +-- (initialer Versand an Küche bei OPEN_TAB nicht nötig wenn erstmalig)
  +-- bei Erweiterung einer offenen Bestellung:
  |     v
  |   ACCEPTED (Küche bestätigt) ----> IN_PREPARATION
  |   REJECTED_BY_KITCHEN
  |
  v (bei initialer Bestellung)
IN_PREPARATION
  |
  v
READY
  |
  +-- bei ON_SITE:    SERVED
  +-- bei PICKUP:     PICKED_UP
  +-- bei DELIVERY:   DELIVERED

Sonderzustände:
  CANCELLED            (manuell storniert)
  REJECTED_BY_KITCHEN  (Küche kann nicht zubereiten)
```

## Erlaubte State-Transitionen (OrderItem)

| Von | Nach | Trigger |
|---|---|---|
| (neu) | REQUESTED | Item zu OPEN_TAB hinzugefügt |
| (neu) | IN_PREPARATION | Item bei initialem Bestellen einer Order direkt versandt |
| REQUESTED | ACCEPTED | Küche bestätigt Erweiterung |
| REQUESTED | REJECTED_BY_KITCHEN | Küche lehnt Erweiterung ab (mit optionaler Begründung) |
| REQUESTED | CANCELLED | Mitarbeiter storniert vor Versand |
| ACCEPTED | IN_PREPARATION | automatisch |
| IN_PREPARATION | READY | Küche markiert fertig |
| IN_PREPARATION | CANCELLED | Manager-PIN-Storno |
| READY | SERVED \| PICKED_UP \| DELIVERED | je nach Bestelltyp |

**Regel zur Order-State-Ableitung:** Der Order-Status wird aus den OrderItem-Status abgeleitet:
- Wenn alle aktiven (nicht-cancelled, nicht-rejected) Items `READY` sind → Order `READY`.
- Wenn mind. ein Item `IN_PREPARATION` und keines `REQUESTED` → Order `IN_PROGRESS`.
- Bei `REJECTED_BY_KITCHEN` an bezahlter Order → Order `NEEDS_RESOLUTION`.

## NEEDS_RESOLUTION-Spezialfall

Wenn ein Item auf `REJECTED_BY_KITCHEN` wechselt und die zugehörige `Order` bereits `Payment.status=SUCCEEDED` hat, dann:

1. `Order.status = NEEDS_RESOLUTION`.
2. **Domain-Event** `OrderNeedsResolutionEvent` → SSE-Push an Personal-Geräte des Standorts → akustische Warnung.
3. Personal MUSS manuell entscheiden:
   - **Option A:** Komplette Stornierung der Bestellung → manueller Refund-Stub → Order auf `CANCELLED`.
   - **Option B:** Nur das eine Item streichen → Order bleibt mit Restmenge in `IN_PROGRESS`/`READY` → manueller Teil-Refund-Stub.
   - **Option C:** Item wird umbestellt (Variante/Option ändern) → neues `REQUESTED` Item.
4. Im MVP: Refund-Vorgang ist NICHT automatisiert. UI zeigt Hinweis „Manuell zurückerstatten oder Gutschrift ausstellen". Phase 2: automatischer Refund über Payment-Provider.

**Audit-Log:** `ORDER_NEEDS_RESOLUTION` und `ORDER_RESOLVED` mit gewählter Option.

## Konkurrenz und Race Conditions

- Order-State-Transitionen MÜSSEN per `SELECT ... FOR UPDATE` oder optimistischer Sperre (`@Version`) abgesichert sein.
- Wenn zwei Mitarbeiter gleichzeitig denselben Tisch öffnen und Items hinzufügen: erlaubt; jede Hinzufügung ist eine eigenständige Insert-Operation auf der Order-Items-Liste, kein Konflikt.
- Wenn zwei Mitarbeiter gleichzeitig dieselbe Bestellung abrechnen wollen: optimistische Sperre auf Order-Version. Erste gewinnt, zweite bekommt Hinweis „Bestellung wurde gerade bezahlt".

## Zeitstempel auf Order

Folgende Felder MÜSSEN bei Erreichen der jeweiligen Zustände gesetzt werden:

| Feld | Gesetzt bei Übergang nach |
|---|---|
| `createdAt` | DRAFT |
| `confirmedAt` | CONFIRMED |
| `acceptedAt` | ACCEPTED |
| `inProgressAt` | IN_PROGRESS |
| `readyAt` | READY |
| `completedAt` | SERVED / PICKED_UP / DELIVERED |
| `paidAt` | PAID |
| `cancelledAt` | CANCELLED |
| `rejectedAt` | REJECTED |

Diese sind unveränderlich nach erstem Setzen.

## Audit-Log-Pflicht

Jede State-Transition (Order und OrderItem) MUSS einen Eintrag in `financial_audit_log` erzeugen mit:
- Akteur (User/Device/System)
- Vor-Zustand, Nach-Zustand
- Begründung (bei Storno/Refund/Override Pflicht, sonst optional)
- Zeitstempel
- Manager-Override-Flag, falls zutreffend

Diese Logs sind 10 Jahre aufzubewahren (GoBD).
