# Domain-Modell

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument beschreibt das Domänen-Modell auf konzeptueller Ebene. Konkrete Datenbank-Schemata stehen in [`../architecture/data-model.md`](../architecture/data-model.md).

## Aggregate-Übersicht

Aggregate (Domain-Driven-Design-Sinn) sind Cluster von Entitäten, die als Einheit konsistent gehalten werden. Folgende Aggregate sind definiert:

| Aggregate Root | Enthält | Verantwortliches Modul |
|---|---|---|
| Tenant | Restaurant-Stammdaten, Lizenz | `tenant` |
| User | Benutzerkonto, Rollen-Zuweisungen | `tenant` |
| Location | Standort, Tische, Annahme-Policy | `tenant` |
| Device | Geräte-Registrierung, Token, Modi | `device` |
| MasterMenu | Stamm-Speisekarte, Kategorien, Artikel, Varianten, Optionen | `menu` |
| LocationMenu | Standort-Overrides | `menu` |
| Order | Bestellung, Bestellpositionen, Zahlung, Lifecycle | `order` |
| Customer | Endkunden-Konto, Adressen | `customer` |
| AcceptancePolicy | Annahme-Regelwerk eines Standorts | `order` |
| AuditEntry | Audit-Log-Eintrag (Append-Only) | `auth` |

**Aggregat-Regel:** Cross-Aggregate-Referenzen sind nur über IDs erlaubt, nicht über direkte Objekt-Referenzen. Beispiel: `Order` referenziert `Customer` über `customerId`, nicht über ein `customer`-Feld.

## Entitäten im Detail

### Tenant (Restaurant)

```
Tenant
  id: UUID
  name: String                           # "Pizzeria Mario"
  legalName: String                      # "Mario Rossi GmbH"
  taxId: String?                         # USt-ID, optional
  registrationNumber: String?            # Handelsregister, optional
  ownerContact: ContactInfo
  status: ENUM (ACTIVE, SUSPENDED, DELETED)
  createdAt, updatedAt: Timestamp
  deletedAt: Timestamp?                  # Soft-Delete

ContactInfo
  fullName, email, phone, postalAddress

Tenant.license: License
License
  baseSeatCount: Int                     # Inkl. Standorte
  baseDeviceCount: Int                   # Inkl. Geräte gesamt
  extraLocationFee: Money?               # Pro zusätzlichen Standort
  extraDeviceFee: Money?
  validFrom, validUntil: Timestamp
```

**Geschäftsregeln:**
- Ein `Tenant` MUSS bei Anlage mindestens einen Standort bekommen (durch das Onboarding).
- Bei `status = SUSPENDED`: alle Logins des Tenants sind gesperrt, Daten bleiben erhalten.
- Bei `status = DELETED`: Soft-Delete, Daten bleiben mit `deletedAt` markiert. Endgültiges Löschen erfolgt nach Aufbewahrungsfristen über separaten Job.

### User (Restaurant-Benutzer)

```
User
  id: UUID
  tenantId: UUID                         # Pflicht, eindeutige Tenant-Zuordnung
  email: String                          # UNIQUE pro tenantId
  passwordHash: String                   # Argon2id
  fullName: String
  pin: String?                           # Hash der 4-6-stelligen Mitarbeiter-PIN, optional
  status: ENUM (ACTIVE, INVITED, LOCKED, DELETED)
  totpSecret: String?                    # Vorbereitet für 2FA in Phase 2, im MVP NULL
  lastLoginAt: Timestamp?
  failedLoginAttempts: Int               # Reset bei erfolgreichem Login
  lockedUntil: Timestamp?
  createdAt, updatedAt: Timestamp
  deletedAt: Timestamp?

User.roleAssignments: List<RoleAssignment>
RoleAssignment
  id: UUID
  role: ENUM (OWNER, MANAGER, LOCATION_LEAD, STAFF)
  scope: Scope
  grantedBy: UUID (User.id)
  grantedAt: Timestamp

Scope
  type: ENUM (TENANT_WIDE, LOCATION_LIST)
  locationIds: List<UUID>?               # nur bei LOCATION_LIST
```

**Geschäftsregeln:**
- Mindestens ein `User` mit Rolle `OWNER` (TENANT_WIDE) MUSS pro Tenant existieren.
- Beim Versuch, den letzten Owner zu entfernen oder zu degradieren, MUSS der Vorgang abgelehnt werden.
- `email` ist innerhalb eines Tenants UNIQUE; gleiche E-Mail kann in verschiedenen Tenants existieren — aber pro Person/Login NUR EIN Tenant (siehe [ADR-005](../adr/005-multi-tenancy.md)).
- Mehrere Rollen-Zuweisungen pro User sind erlaubt (additiv, mit verschiedenen Scopes).

### Location (Standort)

```
Location
  id: UUID
  tenantId: UUID
  name: String                           # "Standort Berlin-Mitte"
  legalAddress: PostalAddress
  contact: ContactInfo
  openingHours: OpeningHoursSchedule
  taxAddress: PostalAddress?             # Falls abweichend von legalAddress
  defaultDeliveryRadiusKm: Double?       # MVP: PLZ-Liste primär
  deliveryZipCodes: List<String>         # Erlaubte PLZs für Lieferung
  status: ENUM (ACTIVE, INACTIVE, DELETED)
  createdAt, updatedAt, deletedAt

OpeningHoursSchedule
  weekly: Map<DayOfWeek, List<TimeRange>>
  exceptions: List<DateRange + (closed | overrideHours)>

Location.acceptancePolicy: AcceptancePolicy   # 1:1
Location.tables: List<Table>
Location.devices: List<Device>
```

**Geschäftsregeln:**
- Ein `Location` MUSS mindestens eine Annahme-Policy haben (Default: AutoAnnahme).
- Lieferzonen-Definition: MVP **nur PLZ-basiert**. Polygone oder Geocoding-Radius in Phase 2.

### Table (Tisch)

```
Table
  id: UUID
  tenantId: UUID
  locationId: UUID
  number: Int                            # standardmäßig inkrementell
  displayLabel: String?                  # optional Override (z.B. "T-Theke-1")
  tags: List<String>                     # ["drinnen", "fenster"]
  positionX, positionY: Double?          # für Tischplan
  qrToken: String                        # permanenter Token für Tisch-QR
  status: ENUM (ACTIVE, MERGED, INACTIVE)
  mergedIntoGroupId: UUID?
  createdAt, updatedAt
```

**Geschäftsregeln:**
- Tisch-Nummern MÜSSEN innerhalb eines Standorts UNIQUE sein.
- `qrToken` wird beim Anlegen einmalig generiert (256-Bit zufällig) und ist permanent gültig. Aktionen über den Token sind rate-limited (siehe [Use Case S08](use-cases.md#uc-s08-tisch-bestellung-und-kellner-ruf-vom-gast-smartphone)).
- Tische in einer aktiven `TableGroup` sind im Status `MERGED`.

### TableGroup (Tisch-Zusammenlegung)

```
TableGroup
  id: UUID
  tenantId: UUID
  locationId: UUID
  tableIds: List<UUID>                   # mind. 2
  activeFrom: Timestamp
  activeUntil: Timestamp?                # NULL = aktiv
  createdBy: UUID (User.id)
```

**Geschäftsregeln:**
- Eine aktive `Order` an einem der Tische einer Gruppe behält ihre Tisch-Referenz, aber das Personal-UI zeigt sie als zur Gruppe gehörig an.

### Device (Gerät)

```
Device
  id: UUID
  tenantId: UUID
  locationId: UUID
  name: String                           # "Tablet Theke 1"
  availableModes: List<DeviceMode>       # Welche Modi sind freigeschaltet
  currentMode: DeviceMode                # Aktiver Modus
  modeSwitchPolicy: ENUM (NO_CONFIRM, MANAGER_PIN)
  staffPinPolicy: ENUM (DISABLED, OPTIONAL, REQUIRED)
  tokenHash: String                      # bcrypt/Argon2id-Hash des Tokens
  lastHeartbeatAt: Timestamp?
  status: ENUM (ACTIVE, REVOKED, INACTIVE)
  createdAt, lastTokenRotationAt: Timestamp

DeviceMode = ENUM:
  ORDER_TABLET            # Tablet zur Tisch-Bestellaufnahme durch Personal
  KITCHEN_DISPLAY         # Küchenanzeige
  POS_TERMINAL            # Phase 2
  STORAGE_SCANNER         # Phase 3
  DELIVERY_DRIVER_APP     # Phase 3
  SELF_ORDER_TERMINAL     # Festes Tablet für Self-Order
```

**Geschäftsregeln:**
- Bei Token-Anlage wird der Klartext-Token EINMALIG zurückgegeben (UI zeigt + QR-Code), in DB nur Hash.
- Heartbeat alle 60 s. Bei `now() - lastHeartbeatAt > 7 Tage`: automatisch `status = INACTIVE`. Manuell wieder aktivierbar.
- Modus-Wechsel ohne offene Vorgänge erlaubt (offene Bestellungen sind server-side, nicht ans Gerät gebunden).
- Bei `modeSwitchPolicy = MANAGER_PIN`: System verlangt vor Wechsel die PIN eines Users mit `Standortleiter`-Rolle oder höher für diesen Standort.

### MasterMenu (Stamm-Speisekarte)

```
MasterMenu
  id: UUID
  tenantId: UUID
  categories: List<Category>
  items: List<MenuItem>

Category
  id: UUID
  name: String                           # "Pizza"
  displayOrder: Int

MenuItem
  id: UUID
  tenantId: UUID
  categoryId: UUID
  name: String
  description: String?
  basePrice: Money                       # In Cent, EUR
  taxRateForOnSite: Decimal              # 19.0
  taxRateForTakeawayDelivery: Decimal    # 7.0
  allergens: List<AllergenCode>
  additives: List<AdditiveCode>
  variants: List<Variant>
  options: List<Option>
  availability: Availability?            # Zeitfenster-Verfügbarkeit
  imageUrl: String?
  status: ENUM (ACTIVE, INACTIVE, DELETED)

Variant
  id: UUID
  name: String                           # "Klein"
  priceDelta: Money                      # +/- ggü. basePrice
  default: Boolean

Option
  id: UUID
  name: String                           # "Extra Käse"
  priceDelta: Money
  group: String?                         # Optional: Gruppierung ("Beläge", "Soßen")

Availability
  weekly: Map<DayOfWeek, List<TimeRange>>
  validFrom, validUntil: Date?

AllergenCode = ENUM (LMIV-Codes A-N und Zusatzstoffe 1-10)
```

**Geschäftsregeln:**
- Pro `MenuItem` MUSS mindestens eine `Variant` existieren (auch wenn nur eine Standardgröße — Implementierung kann eine `default`-Variante mit `priceDelta = 0` als Konvention nutzen).
- Allergene und Zusatzstoffe sind LMIV-Pflichtangabe für Online-Verkauf. UI MUSS bei fehlender Pflege einen Hinweis „Allergene auf Anfrage" generieren (rechtlich grenzwertig, aber praktikabel im MVP — siehe [`compliance/lmiv-impressum-agb.md`](../compliance/lmiv-impressum-agb.md)).

### LocationMenu (Standort-Speisekarte)

```
LocationMenu
  id: UUID
  tenantId: UUID
  locationId: UUID                       # 1:1 mit Location
  itemOverrides: List<MenuItemOverride>
  ownItems: List<MenuItem>               # Standort-eigene Artikel

MenuItemOverride
  masterItemId: UUID                     # Referenz auf MasterMenu.MenuItem
  disabled: Boolean
  priceOverride: Money?                  # NULL = keine Überschreibung
  variantPriceOverrides: Map<UUID, Money>?
  optionPriceOverrides: Map<UUID, Money>?
```

**Geschäftsregeln:**
- Effektive Speisekarte eines Standorts = Stamm-Items minus deaktivierte + Preisüberschreibungen + Standort-eigene Items.
- Standort-eigene Items haben dieselbe Struktur wie Stamm-Items, sind aber nur an diesem Standort verfügbar.

### Order (Bestellung)

```
Order
  id: UUID
  tenantId: UUID
  locationId: UUID
  orderType: ENUM (ON_SITE, PICKUP, DELIVERY)
  orderMode: ENUM (SELF_ORDER_INSTANT, OPEN_TAB)
  status: OrderStatus                    # siehe order-lifecycle.md
  tableId: UUID?                         # bei ON_SITE
  customerId: UUID?                      # bei online: Endkunde, bei vor-Ort: NULL
  customerSnapshot: CustomerSnapshot?    # Gast-Bestellung ohne Konto
  staffUserId: UUID?                     # Mitarbeiter, wenn am Gerät identifiziert
  deviceId: UUID?                        # Gerät, von dem die Bestellung kam
  acceptanceDecision: AcceptanceDecision?# Auto/Manuell + Zeitpunkt + Akteur
  estimatedReadyAt: Timestamp?           # für Abholung/Lieferung
  scheduledForAt: Timestamp?             # falls Vorbestellung
  deliveryAddress: PostalAddress?        # bei DELIVERY
  notes: String?                         # freie Bestell-Notiz
  totalGross: Money
  totalNet: Money
  totalTax: Money
  taxBreakdown: Map<TaxRate, Money>      # für Bon
  taxOverridden: Boolean
  taxOverrideReason: String?
  createdAt, confirmedAt, acceptedAt, completedAt, paidAt: Timestamp?

Order.items: List<OrderItem>
Order.payments: List<Payment>
Order.audit: List<OrderAuditEntry>      # alle Zustandsänderungen

OrderItem
  id: UUID
  orderId: UUID
  menuItemId: UUID
  variantId: UUID
  selectedOptionIds: List<UUID>
  quantity: Int
  priceSnapshot: Money                   # Preis × Menge zum Bestellzeitpunkt (inkl. Optionen)
  taxRateSnapshot: Decimal
  status: OrderItemStatus
  rejectedReason: String?                # bei REJECTED_BY_KITCHEN
  notes: String?                         # "ohne Zwiebel" wenn nicht in Optionen abbildbar

CustomerSnapshot                         # für Gast-Bestellungen ohne Konto
  fullName, email, phone, address
```

**Geschäftsregeln:**
- Eine `Order` MUSS einen `tenantId` und `locationId` haben.
- Bei `orderMode = SELF_ORDER_INSTANT` MUSS die Zahlung VOR dem Status `CONFIRMED` erfolgreich abgeschlossen sein.
- Bei `orderMode = OPEN_TAB` ist Hinzufügen weiterer `OrderItem`s erlaubt, solange `status` nicht in `READY`, `SERVED`, `CANCELLED` oder `PAID` ist.
- `priceSnapshot` ist unveränderlich (siehe [ADR-...]). Korrekturen erfordern Storno + neue Position.

### Payment (Zahlung)

```
Payment
  id: UUID
  tenantId: UUID
  orderId: UUID
  amount: Money
  method: ENUM (CASH, CARD, ONLINE, OTHER)
  status: ENUM (INITIATED, SUCCEEDED, FAILED, REFUNDED, PARTIALLY_REFUNDED)
  paymentProviderRef: String?            # Token vom PaymentPort
  refundAmount: Money?
  refundReason: String?
  initiatedAt, completedAt, refundedAt: Timestamp?
```

### Customer (Endkunde)

```
Customer
  id: UUID
  tenantId: UUID                         # Endkunde gehört einem Tenant
  email: String                          # UNIQUE (tenantId, email)
  passwordHash: String?                  # NULL bei Magic-Link-only
  fullName: String
  phone: String?
  status: ENUM (ACTIVE, ANONYMIZED)
  emailVerifiedAt: Timestamp?
  createdAt, updatedAt
  anonymizedAt: Timestamp?               # bei DSGVO-Löschung

Customer.addresses: List<CustomerAddress>

CustomerAddress
  id: UUID
  customerId: UUID
  label: String                          # "Zuhause", "Büro"
  address: PostalAddress
  isDefault: Boolean
```

**Geschäftsregeln:**
- DSGVO-Löschung anonymisiert das Konto: E-Mail wird auf `deleted-{id}@anonymous.local` gesetzt, Name auf "Gelöscht", Adressen werden entfernt. Bestellungen bleiben mit Pseudonym wegen GoBD.
- Magic-Link-Login: separates Magic-Token-Aggregat, hier nicht modelliert.

### AcceptancePolicy (Annahme-Policy)

Siehe [`acceptance-policies.md`](acceptance-policies.md).

### WaiterCall (Kellner-Ruf)

```
WaiterCall
  id: UUID
  tenantId: UUID
  locationId: UUID
  tableId: UUID
  triggeredAt: Timestamp
  triggerSource: ENUM (TABLE_QR_GUEST, GUEST_DEVICE)
  status: ENUM (OPEN, ACKNOWLEDGED, RESOLVED, IGNORED, BLOCKED)
  acknowledgedBy: UUID?                  # User.id
  resolvedAt: Timestamp?
  ipFingerprint: String                  # für Rate-Limiting & Audit
```

**Geschäftsregeln:**
- Rate-Limit: max. 1 OPEN `WaiterCall` pro `tableId` gleichzeitig; nach Resolve erst nach 3 min wieder möglich.
- Personal kann einen Tisch temporär blocken (alle weiteren Calls werden für X Stunden ignoriert).

### AuditEntry

Siehe [`../architecture/audit-log.md`](../architecture/audit-log.md).

## Beziehungen-Diagramm (textuell)

```
Tenant 1───n Location 1───n Table
       │                  │
       │                  └─n Device
       │
       1───n User 1───n RoleAssignment ── Scope ── (TENANT_WIDE | LOCATION_LIST → Location[])
       │
       1───n Customer 1───n CustomerAddress
       │
       1────1 MasterMenu 1───n Category 1───n MenuItem 1───n Variant
       │                                              1───n Option
       │
       Location 1────1 LocationMenu n───1 MenuItem (Override)
       │
       1───n Order 1───n OrderItem ───→ MenuItem
                  └───n Payment
                  └───n OrderAuditEntry
```

## Wertobjekte (Value Objects)

Wertobjekte sind unveränderlich, identitätslos, durch ihre Werte gleichgesetzt.

```
Money
  amountCents: Long
  currency: ENUM (EUR)                   # MVP nur EUR

PostalAddress
  street, houseNumber, postalCode, city, countryCode

TimeRange
  startTime, endTime: LocalTime

DateRange
  startDate, endDate: LocalDate
```

**Geschäftsregel:** `Money`-Operationen MÜSSEN auf Cents arbeiten. Float/Double für Geldbeträge ist VERBOTEN (Rundungsfehler).

## Domain-Events

Innerhalb der Anwendung werden Domain-Events über Spring-Application-Events publiziert. Module reagieren asynchron darauf.

| Event | Publisher | Subscriber |
|---|---|---|
| `TenantCreatedEvent` | `tenant` | `notification` (Welcome-Mail) |
| `OrderConfirmedEvent` | `order` | `notification`, `reporting` |
| `OrderItemRejectedByKitchenEvent` | `order` | `notification`, `customer` (Stub: virtuelles Postfach) |
| `OrderStateChangedEvent` | `order` | `reporting`, `web` (SSE-Push), `notification` |
| `OrderPaidEvent` | `order` | `tse`, `reporting` |
| `WaiterCallTriggeredEvent` | `order` | `web` (SSE-Push an Personal-Geräte) |
| `DeviceHeartbeatMissedEvent` | `device` | `notification` (Admin-Alert nach 7 Tagen) |
| `CustomerAnonymizedEvent` | `customer` | `auth` (Sessions invalidieren) |

**Regel:** Domain-Events MÜSSEN tenant-skopiert sein (`tenantId` als Pflicht-Feld), damit Subscriber den Kontext kennen.

## Invariants (System-weit)

Folgende Invarianten sind ALLE auf Datenbank-Ebene durch Constraints UND in der Anwendungsschicht durchzusetzen:

1. **Mandantentrennung:** Jede Datenzeile mit `tenant_id` darf nur durch Sessions mit gleichem Tenant-Kontext gelesen/geschrieben werden (RLS).
2. **Owner-Mindestbestand:** Jeder aktive Tenant hat mindestens einen User mit Rolle `OWNER`.
3. **Order-Lifecycle:** State-Transitionen folgen strikt dem Diagramm in [`order-lifecycle.md`](order-lifecycle.md). Keine Sprünge.
4. **Preis-Snapshot:** `OrderItem.priceSnapshot` ist unveränderlich nach erstem Speichern.
5. **Payment vor Confirm bei SELF_ORDER_INSTANT:** Bestellung darf nicht in `CONFIRMED` ohne erfolgreiches `Payment.SUCCEEDED` übergehen.
6. **Audit-Append-Only:** `audit_log` und `financial_audit_log` erlauben nur INSERT.
7. **Soft-Delete:** Entitäten mit Geschäftsbedeutung (User, Customer, Location, MenuItem) verwenden `deletedAt`. Hartes DELETE nur durch Retention-Jobs.
