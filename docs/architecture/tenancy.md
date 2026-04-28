# Mandantentrennung (Multi-Tenancy)

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Mandantentrennung ist die wichtigste Sicherheitseigenschaft der Plattform. **Datenleaks zwischen Tenants sind ein KO-Kriterium**.

## Modell

Variante: **Shared Database, Shared Schema, Tenant-ID-Spalte plus PostgreSQL Row-Level Security (RLS).**

- Eine PostgreSQL-Datenbank für alle Tenants.
- Ein Schema (`public` oder `app`).
- Jede Tabelle, deren Zeilen zu einem Tenant gehören, hat `tenant_id UUID NOT NULL REFERENCES tenants(id)`.
- Auf jeder solchen Tabelle ist RLS aktiviert mit Policy: `tenant_id = current_setting('app.tenant_id')::uuid`.

## Defense-in-Depth: 3 Schichten

```
┌─────────────────────────────────────┐
│ 1. Anwendungsschicht (Service)      │
│    Permission- und Scope-Prüfung    │
│    via PermissionEvaluator          │
└─────────────────────────────────────┘
                ▼
┌─────────────────────────────────────┐
│ 2. Repository-Schicht                │
│    JPA-Repositories filtern explizit │
│    nach tenantId                     │
└─────────────────────────────────────┘
                ▼
┌─────────────────────────────────────┐
│ 3. Datenbank-Schicht                 │
│    PostgreSQL RLS erzwingt           │
│    tenant_id = current_setting(...)  │
└─────────────────────────────────────┘
```

Alle drei Schichten MÜSSEN aktiv sein. Eine einzelne Schicht ist KEINE ausreichende Sicherheit.

## Tenant-Kontext im Code

### Setzen des Kontexts

```java
public class TenantContext {
    private static final ThreadLocal<UUID> currentTenant = new ThreadLocal<>();

    public static void setCurrentTenant(UUID tenantId) {
        currentTenant.set(tenantId);
    }

    public static UUID getCurrentTenant() {
        return Objects.requireNonNull(currentTenant.get(),
            "Tenant context not set");
    }

    public static void clear() {
        currentTenant.remove();
    }
}
```

### Filter / Interceptor

Ein `TenantContextFilter` ist registriert für alle relevanten Endpoints:

```java
@Component
public class TenantContextFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                     HttpServletResponse res,
                                     FilterChain chain) throws ... {
        try {
            UUID tenantId = resolveTenantFromAuth(req);
            if (tenantId != null) {
                TenantContext.setCurrentTenant(tenantId);
            }
            chain.doFilter(req, res);
        } finally {
            TenantContext.clear();
        }
    }

    private UUID resolveTenantFromAuth(HttpServletRequest req) {
        // 1. Session-User → User.tenantId
        // 2. Geräte-Token → Device.tenantId
        // 3. Endkunden-Session → Customer.tenantId
        // 4. Plattform-Admin → KEIN Tenant-Context (separater Endpoint-Pfad)
    }
}
```

### Connection-Pool-Integration

Bei jedem DB-Connection-Checkout wird die Session-Variable gesetzt:

```java
@Component
public class TenantAwareConnectionInterceptor {
    @EventListener
    public void onConnectionCheckout(...) {
        UUID tenantId = TenantContext.getCurrentTenant();
        connection.createStatement().execute(
            "SET LOCAL app.tenant_id = '" + tenantId + "'"
        );
    }
}
```

Implementierung kann auch über `Hibernate MultiTenantConnectionProvider` erfolgen.

## RLS-Policies pro Tabelle

Beispiel für die `orders`-Tabelle:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_orders ON orders
    USING (tenant_id = current_setting('app.tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

**Alle Tabellen mit `tenant_id`-Spalte MÜSSEN entsprechende Policy haben.** Eine Migration darf nicht akzeptiert werden, die eine `tenant_id`-Spalte ohne RLS-Policy einführt — CI-Check.

## Tabellen mit und ohne `tenant_id`

**MIT `tenant_id` (RLS aktiv):**
- users, role_assignments
- locations, tables, table_groups
- devices, device_tokens
- master_menus, location_menus, menu_items, variants, options, categories
- orders, order_items, payments
- customers, customer_addresses
- waiter_calls
- acceptance_policies
- audit_log (mit `tenant_id` als nullable, aber RLS erlaubt `NULL` für Plattform-Events siehe unten)
- financial_audit_log

**OHNE `tenant_id` (RLS nicht aktiv, separate Permission-Logik):**
- tenants (die Tabelle der Tenants selbst)
- admins (Plattform-Admin-Konten)
- platform_config
- flyway_schema_history (DB-internes)

## Plattform-Admin-Zugriff

Plattform-Admins müssen tenant-übergreifend arbeiten können. Lösung: **Separater DB-User**.

```sql
-- Anwendungs-User (für Tenant-Operationen)
CREATE USER app WITH PASSWORD '...';
GRANT app_role TO app;

-- Admin-User (für Plattform-Operationen)
CREATE USER platform_admin WITH PASSWORD '...';
GRANT app_role TO platform_admin;
GRANT BYPASSRLS TO platform_admin;  -- darf RLS umgehen
```

Die Anwendung verwendet **zwei Connection-Pools**:
- `app-pool`: Connection-User `app`, RLS aktiv. Default für alle normalen Requests.
- `platform-pool`: Connection-User `platform_admin`, RLS umgangen. Nur für explizite Plattform-Endpoints.

```java
@PlatformOperation  // Custom Annotation
public List<Tenant> listAllTenants() {
    // führt automatisch über platform-pool aus
}
```

**Sicherheitskritisch:** Der `platform-pool` darf NICHT von Endpoints verwendet werden, die einen Tenant-Kontext haben. Statische Code-Analyse oder ArchUnit prüft das.

## Audit-Log Tenant-Kontext

`audit_log` und `financial_audit_log` haben `tenant_id` (nullable) und einen RLS-Policy:

```sql
CREATE POLICY tenant_or_platform_audit ON audit_log
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        OR (current_setting('app.is_platform', true) = 'true' AND tenant_id IS NOT NULL)
        OR tenant_id IS NULL  -- Plattform-Events sichtbar für Plattform
    );
```

Plattform-Events (`tenant_id IS NULL`) sind nur über den Platform-Pool einsehbar.

## Tenant-übergreifende Reads (verboten)

Cross-Tenant-Joins sind durch RLS automatisch verhindert. Eine Query wie

```sql
SELECT * FROM orders WHERE tenant_id IN (...)  -- mehrere Tenants
```

gibt im normalen `app`-User-Kontext nur Zeilen des aktuellen Tenants zurück, unabhängig von der WHERE-Klausel.

## Testing der Mandantentrennung

**Pflicht-Tests** (im CI):

1. **RLS-Test:** Mit Tenant-Kontext A werden Zeilen von Tenant B angefragt → 0 Treffer.
2. **Tenant-Switch-Test:** Innerhalb derselben Connection wird `app.tenant_id` gewechselt — Daten beider Tenants werden je nach Kontext sichtbar.
3. **Missing-Context-Test:** Query ohne gesetzten Tenant-Kontext schlägt fehl.
4. **Insert-Cross-Tenant-Test:** Versuch, eine Zeile mit fremder `tenant_id` einzufügen, schlägt durch RLS-Check fehl.
5. **Permission-Override-Test:** Selbst mit Permissions auf Service-Ebene scheitert ein Cross-Tenant-Zugriff durch RLS.

## Endkunden-Daten und Mandantentrennung

Endkunden-Konten gehören zu einem Tenant (`customers.tenant_id`). Eine E-Mail-Adresse kann in mehreren Tenants als Endkunde existieren — jedes Konto ist getrennt.

```sql
ALTER TABLE customers ADD CONSTRAINT customers_email_per_tenant_unique
  UNIQUE (tenant_id, email);
```

Login eines Endkunden:
1. Endkunde besucht Online-Bestellseite eines Tenants → Tenant ist aus URL bekannt.
2. Login-Versuch mit E-Mail+Passwort → System sucht in `customers` mit `tenant_id` = aktueller Tenant + `email` = eingegeben.
3. Erfolgreich: Session-Cookie mit Tenant-Bezug.

## Tenant-Erkennung aus URL

Online-Bestellseite kann unter:
- `https://<subdomain>.platform.de` (Plattform-Subdomain pro Tenant)
- `https://www.<eigene-domain>` (vom Tenant gepflegte Custom-Domain)

erreicht werden. Erkennung:
1. Custom-Domain-Mapping-Tabelle `tenant_domains (host, tenant_id)`.
2. Bei Plattform-Subdomain: `<subdomain>` zu `tenants.slug` mappen.

## Mandantentrennung bei Domain-Events

Domain-Events MÜSSEN `tenantId` enthalten:

```java
public record OrderConfirmedEvent(
    UUID tenantId,
    UUID orderId,
    LocalDateTime occurredAt
) { ... }
```

Listener prüfen Tenant-Kontext beim Verarbeiten:

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onOrderConfirmed(OrderConfirmedEvent event) {
    TenantContext.setCurrentTenant(event.tenantId());
    try {
        // ... Verarbeitung ...
    } finally {
        TenantContext.clear();
    }
}
```

## Migrations-Strategie

Schema-Migrationen via Flyway. Pflicht-Konventionen:

1. Neue tenant-bezogene Tabellen MÜSSEN `tenant_id UUID NOT NULL` enthalten.
2. Neue tenant-bezogene Tabellen MÜSSEN RLS aktivieren mit Standard-Policy.
3. Migration-Tests im CI verifizieren das.

Beispiel-Migration:
```sql
-- V012__add_loyalty_program.sql
CREATE TABLE loyalty_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    points INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE loyalty_accounts ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_loyalty ON loyalty_accounts
    USING (tenant_id = current_setting('app.tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE INDEX idx_loyalty_tenant_customer ON loyalty_accounts(tenant_id, customer_id);
```

## Zukünftige Erweiterung

Bei sehr großen Tenants kann perspektivisch (Phase 3+) eine Migration auf **DB-pro-Tenant für Premium-Kunden** erfolgen. Architektur ist über `Hibernate MultiTenantConnectionProvider` darauf vorbereitet.
