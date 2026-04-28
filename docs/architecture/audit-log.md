# Audit-Log

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Audit-Logs sind essentiell für Sicherheits-Diagnose, DSGVO-Auskunftspflicht, GoBD und spätere TSE-Anbindung.

## Zwei Audit-Log-Tabellen

Aus regulatorischen Gründen sind zwei separate Tabellen mit unterschiedlicher Aufbewahrung erforderlich.

### `audit_log` (90 Tage Retention)

Allgemeine Sicherheits-Events:
- Auth-Events (Login, Logout, Failed Logins, Lockouts)
- Permission-Änderungen, Rollen-Zuweisungen
- Speisekarten-Änderungen mit Diff
- Geräte-Token-Lifecycle
- Annahme-Policy-Änderungen
- Tenant-Lifecycle (Create, Suspend, Reactivate, Delete)
- Plattform-Admin-Aktionen
- Impersonation Start/Ende

### `financial_audit_log` (10 Jahre Retention, GoBD)

Geldwert-relevante Events:
- Order-Lifecycle-Transitionen
- OrderItem-Lifecycle
- Stornos (mit Begründung)
- Steuersatz-Overrides
- Refunds
- Zahlungs-Vorgänge
- TSE-Signaturen
- Annahme-/Ablehnungs-Entscheidungen mit Beträgen

## Tabellen-Schema

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    tenant_id UUID NULL REFERENCES tenants(id),     -- NULL bei Plattform-Events
    actor_type ACTOR_TYPE_ENUM NOT NULL,
    actor_id UUID NULL,                              -- User/Admin/Customer/Device-ID
    actor_ip INET NULL,
    actor_user_agent TEXT NULL,
    impersonated_by UUID NULL,                       -- bei Admin-Impersonation
    initiated_by_user UUID NULL,                     -- bei Manager-PIN-Override
    action AUDIT_ACTION_ENUM NOT NULL,
    target_type TEXT NULL,                           -- z.B. "User", "Order", "MenuItem"
    target_id UUID NULL,
    payload JSONB NULL,                              -- Diff oder Kontext
    correlation_id UUID NULL,                        -- für Request-Tracing
    PRIMARY KEY (id)
);

-- Append-only Constraint via Trigger
CREATE TRIGGER audit_log_append_only
BEFORE UPDATE OR DELETE ON audit_log
FOR EACH ROW EXECUTE FUNCTION raise_exception_append_only();

-- RLS
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_log_tenant_or_platform ON audit_log
    USING (
        (tenant_id = current_setting('app.tenant_id', true)::uuid)
        OR (current_setting('app.is_platform', true) = 'true')
    );

-- Indizes
CREATE INDEX idx_audit_tenant_time ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_target ON audit_log (target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_action_time ON audit_log (action, occurred_at DESC);

-- Analoge Definition für financial_audit_log
```

## Append-Only-Garantie

```sql
CREATE OR REPLACE FUNCTION raise_exception_append_only()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_log is append-only';
END;
$$ LANGUAGE plpgsql;
```

Updates und Deletes auf Audit-Tabellen werden auf DB-Ebene unterbunden. Das gilt auch für den `platform_admin`-DB-User. Einzige Ausnahme: Retention-Pruning-Job (eigener DB-User mit speziellem Recht).

## Akteur-Typen

```
ACTOR_TYPE_ENUM:
  PLATFORM_ADMIN
  RESTAURANT_USER
  CUSTOMER
  DEVICE
  SYSTEM                      -- automatische System-Aktionen (Cronjobs, Events)
  ADMIN_IMPERSONATING         -- Plattform-Admin agiert als Restaurant-User
```

## Action-Enum (Auswahl)

```
AUDIT_ACTION_ENUM (audit_log):
  -- Auth
  LOGIN_SUCCEEDED, LOGIN_FAILED, LOGOUT, LOCKOUT,
  PASSWORD_CHANGED, PASSWORD_RESET_REQUESTED, PASSWORD_RESET_COMPLETED,
  MAGIC_LINK_SENT, MAGIC_LINK_CONSUMED,
  MANAGER_PIN_CONFIRMED,

  -- Tenant
  TENANT_CREATED, TENANT_SUSPENDED, TENANT_REACTIVATED, TENANT_DELETED,
  LICENSE_CHANGED,

  -- Users
  USER_INVITED, USER_ACTIVATED, USER_REMOVED, USER_ROLE_CHANGED,
  OWNERSHIP_TRANSFERRED,

  -- Locations & Tables
  LOCATION_CREATED, LOCATION_UPDATED, LOCATION_DELETED,
  TABLE_CREATED, TABLE_UPDATED, TABLE_GROUP_CREATED, TABLE_GROUP_DISSOLVED,

  -- Devices
  DEVICE_CREATED, DEVICE_TOKEN_REVOKED, DEVICE_TOKEN_ROTATED,
  DEVICE_MODE_SWITCHED,

  -- Menu
  MENU_ITEM_CREATED, MENU_ITEM_UPDATED, MENU_ITEM_DELETED,
  LOCATION_MENU_OVERRIDE_CHANGED,

  -- Acceptance Policy
  ACCEPTANCE_POLICY_CHANGED, ACCEPTANCE_OVERRIDE_SET, ACCEPTANCE_OVERRIDE_CLEARED,

  -- Customer
  CUSTOMER_REGISTERED, CUSTOMER_ANONYMIZED,

  -- Waiter Calls
  WAITER_CALL_TRIGGERED, WAITER_CALL_RESOLVED, WAITER_CALL_BLOCKED,

  -- Platform-Admin
  IMPERSONATION_STARTED, IMPERSONATION_ENDED

FINANCIAL_AUDIT_ACTION_ENUM (financial_audit_log):
  -- Order Lifecycle
  ORDER_CREATED, ORDER_CONFIRMED, ORDER_ACCEPTED, ORDER_REJECTED,
  ORDER_IN_PROGRESS, ORDER_READY, ORDER_SERVED, ORDER_PICKED_UP,
  ORDER_OUT_FOR_DELIVERY, ORDER_DELIVERED, ORDER_PAID,
  ORDER_CANCELLED, ORDER_NEEDS_RESOLUTION, ORDER_RESOLVED,

  -- OrderItems
  ORDER_ITEM_ADDED, ORDER_ITEM_REQUESTED, ORDER_ITEM_ACCEPTED,
  ORDER_ITEM_REJECTED_BY_KITCHEN, ORDER_ITEM_IN_PREPARATION,
  ORDER_ITEM_READY, ORDER_ITEM_CANCELLED,

  -- Tax & Pricing
  ORDER_TAX_OVERRIDDEN, ORDER_DISCOUNT_APPLIED,

  -- Payments
  PAYMENT_INITIATED, PAYMENT_SUCCEEDED, PAYMENT_FAILED,
  PAYMENT_REFUNDED_FULL, PAYMENT_REFUNDED_PARTIAL,

  -- TSE
  TSE_SIGNATURE_CREATED, TSE_SIGNATURE_FAILED,

  -- Acceptance
  ACCEPTANCE_DECISION_AUTO, ACCEPTANCE_DECISION_MANUAL_ACCEPT,
  ACCEPTANCE_DECISION_MANUAL_REJECT, ACCEPTANCE_DECISION_TIMEOUT
```

## Payload-Schema

`payload` ist JSONB mit aktionspezifischer Struktur. Beispiele:

**MENU_ITEM_UPDATED:**
```json
{
  "menuItemId": "...",
  "before": {"name": "Pizza Margherita", "basePrice": 850},
  "after":  {"name": "Pizza Margherita", "basePrice": 950},
  "diff":   ["basePrice"]
}
```

**ORDER_PAID:**
```json
{
  "orderId": "...",
  "paymentMethod": "CASH",
  "amountGross": 4250,
  "amountNet": 3958,
  "amountTax": 292,
  "taxBreakdown": {"7.0": 158, "19.0": 134},
  "tseSignatureRef": "...",
  "receiptId": "..."
}
```

**ORDER_TAX_OVERRIDDEN:**
```json
{
  "orderId": "...",
  "itemId": "...",
  "originalTaxRate": 7.0,
  "newTaxRate": 19.0,
  "reason": "Gast wechselt von Lieferung zu Vor-Ort"
}
```

**IMPERSONATION_STARTED:**
```json
{
  "adminId": "...",
  "targetTenantId": "...",
  "targetUserId": "...",
  "reason": "Support-Anfrage Ticket #12345"
}
```

## Schreib-Pfad

Ein zentraler `AuditLogger`-Service im `auth`-Modul kapselt alle Schreibvorgänge:

```java
public interface AuditLogger {
    void log(AuditEntry entry);
    void logFinancial(FinancialAuditEntry entry);
}
```

Aufrufer:
- Service-Methoden direkt (für sichere, transaktionale Logs).
- `@TransactionalEventListener(phase = AFTER_COMMIT)` für Domain-Events.

**Pflichtinvariante:** Audit-Logs werden in **derselben Transaktion** geschrieben wie die zu auditierende Aktion (für nicht-Event-basierte). Sonst Inkonsistenz möglich.

## Lese-Pfad

`AuditQueryService` mit folgenden Methoden:

```java
List<AuditEntry> findByTenant(UUID tenantId, AuditQuery query);
List<FinancialAuditEntry> findFinancialByOrder(UUID orderId);
List<AuditEntry> findPlatformEvents(AuditQuery query);   // nur PlatformAdmin
```

`AuditQuery`:
- Zeitraum
- Akteur-Typ / Akteur-ID
- Action-Filter
- Target-Typ / Target-ID
- Pagination

**Performance:** Listen-Queries müssen Pagination nutzen. Indexe sind oben definiert.

## UI-Sicht

**Tenant-Audit-View:** Eingeloggter User mit `AUDIT_VIEW`-Permission sieht Audit-Log seines Tenants. Standortleiter sieht nur Aktionen seiner Standorte (Filter durch RLS bzw. Service-Layer).

**Plattform-Audit-View:** SuperAdmin/SupportAdmin sieht Plattform-weite Logs.

**Order-Detail:** Bei Order-Detail-Seite ist die Order-Audit-Historie eingebettet (wer hat wann was gemacht).

## DSGVO-Auskunft

Bei Auskunftsersuchen eines Endkunden:
1. Suchen `customer_id` aus E-Mail.
2. Audit-Einträge mit `actor_type=CUSTOMER, actor_id=<id>` exportieren.
3. Alle Bestellungen des Customers + zugehörige Financial-Audit-Einträge exportieren.

## DSGVO-Löschung

Bei Löschung eines Endkundens:
1. Customer-Datensatz wird anonymisiert (siehe [`../compliance/gdpr.md`](../compliance/gdpr.md)).
2. Audit-Einträge bleiben mit Pseudonym-ID erhalten (GoBD verlangt Aufbewahrung).
3. Audit-Einträge mit `actor_type=CUSTOMER` setzen `actor_id` auf den anonymisierten Customer-Datensatz.

## Retention-Job

```
Daily Cron Job:
  DELETE FROM audit_log WHERE occurred_at < now() - INTERVAL '90 days';
  DELETE FROM financial_audit_log WHERE occurred_at < now() - INTERVAL '10 years';
```

Der Cron-Job läuft mit dem speziellen `audit_pruner` DB-User, der als einziger DELETE-Rechte auf Audit-Tabellen hat.

**Pflicht:** Retention-Job läuft nur in Prod. In Demo/Dev-Profilen ist er deaktiviert.

## Tampering-Erkennung (optional, Phase 2)

Bei Bedarf kann ein Hash-Chain-Mechanismus aufgesetzt werden:
- Jeder Audit-Eintrag hat `previous_hash` und `entry_hash`.
- `entry_hash = SHA-256(payload, occurred_at, previous_hash)`.
- Manipulation eines Eintrags bricht die Kette.

Im MVP nicht implementiert. Append-Only-Trigger gibt grundlegenden Schutz.

## Test-Anforderungen

- Bei jeder neuen Service-Methode mit auditierbarer Wirkung: Test, dass Audit-Eintrag korrekt geschrieben wird.
- Test, dass Append-Only-Trigger UPDATE und DELETE blockiert.
- Test, dass RLS auf Audit-Log-Tabellen funktioniert (Cross-Tenant-Read schlägt fehl).
- Test, dass Retention-Job nur abgelaufene Einträge löscht.
