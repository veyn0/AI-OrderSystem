# Datenmodell

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument enthält die finalen Tabellen-Definitionen. SQL-Snippets sind PostgreSQL-Dialekt.

## Konventionen

- **Primärschlüssel:** `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`. Niemals Auto-Increment-Integer.
- **Geld:** als `BIGINT` in Cents. Niemals `NUMERIC` oder `FLOAT`.
- **Zeitstempel:** `TIMESTAMPTZ` (mit Zeitzone). Niemals `TIMESTAMP` (ohne).
- **Tenant-Isolation:** Jede Tabelle mit Tenant-Bezug hat `tenant_id UUID NOT NULL REFERENCES tenants(id)` und RLS-Policy.
- **Soft-Delete:** Tabellen mit Geschäftsbedeutung haben `deleted_at TIMESTAMPTZ NULL`.
- **Created/Updated:** `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at` per Trigger.
- **Indizes:** mindestens `tenant_id` plus alle FK-Spalten. Ggf. zusammengesetzte Indizes für Reporting-Queries.
- **Constraints:** UNIQUE-Constraints sind häufig pro Tenant: `UNIQUE (tenant_id, ...)`.
- **Naming:** Tabellen plural snake_case (`menu_items`), Spalten snake_case (`tenant_id`).

## Plattform-Tabellen (kein RLS)

### `tenants`

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    legal_name VARCHAR(200) NOT NULL,
    tax_id VARCHAR(50) NULL,
    registration_number VARCHAR(50) NULL,
    owner_full_name VARCHAR(200) NOT NULL,
    owner_email VARCHAR(255) NOT NULL,
    owner_phone VARCHAR(50) NULL,
    owner_address_street VARCHAR(200) NOT NULL,
    owner_address_house_number VARCHAR(20) NOT NULL,
    owner_address_postal_code VARCHAR(10) NOT NULL,
    owner_address_city VARCHAR(100) NOT NULL,
    owner_address_country_code CHAR(2) NOT NULL DEFAULT 'DE',
    status tenant_status_enum NOT NULL DEFAULT 'ACTIVE',
    pending_setup BOOLEAN NOT NULL DEFAULT true,
    slug VARCHAR(100) UNIQUE NOT NULL,           -- für Subdomain
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE TYPE tenant_status_enum AS ENUM ('ACTIVE', 'SUSPENDED', 'DELETED');

CREATE INDEX idx_tenants_status ON tenants (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_tenants_slug ON tenants (slug) WHERE deleted_at IS NULL;
```

### `tenant_licenses`

```sql
CREATE TABLE tenant_licenses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    base_location_count INT NOT NULL,
    base_device_count INT NOT NULL,
    extra_location_fee_cents BIGINT NULL,
    extra_device_fee_cents BIGINT NULL,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_licenses_tenant ON tenant_licenses (tenant_id);
```

### `tenant_setup_tokens`

```sql
CREATE TABLE tenant_setup_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    consumed_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `tenant_domains`

```sql
CREATE TABLE tenant_domains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    host VARCHAR(255) UNIQUE NOT NULL,           -- z.B. "www.pizzeria-mario.de"
    is_primary BOOLEAN NOT NULL DEFAULT false,
    verified_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `admins`

```sql
CREATE TABLE admins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(200) NOT NULL,
    role admin_role_enum NOT NULL,
    status user_status_enum NOT NULL DEFAULT 'ACTIVE',
    failed_login_attempts INT NOT NULL DEFAULT 0,
    locked_until TIMESTAMPTZ NULL,
    last_login_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE TYPE admin_role_enum AS ENUM ('SUPER_ADMIN', 'SUPPORT_ADMIN', 'BILLING_ADMIN');
CREATE TYPE user_status_enum AS ENUM ('ACTIVE', 'INVITED', 'LOCKED', 'DELETED');
```

### `platform_config`

```sql
CREATE TABLE platform_config (
    key VARCHAR(100) PRIMARY KEY,
    value JSONB NOT NULL,
    updated_by UUID NULL REFERENCES admins(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Mandanten-Tabellen (mit RLS)

### `users`

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NULL,             -- NULL bei INVITED
    full_name VARCHAR(200) NOT NULL,
    pin_hash VARCHAR(255) NULL,                  -- 4-6-stellige PIN, Argon2
    status user_status_enum NOT NULL DEFAULT 'INVITED',
    totp_secret VARCHAR(64) NULL,                -- Phase 2
    totp_enabled BOOLEAN NOT NULL DEFAULT false,
    failed_login_attempts INT NOT NULL DEFAULT 0,
    locked_until TIMESTAMPTZ NULL,
    last_login_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL,

    UNIQUE (tenant_id, email)
);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE INDEX idx_users_tenant_email ON users (tenant_id, email) WHERE deleted_at IS NULL;
```

### `role_assignments`

```sql
CREATE TABLE role_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role role_enum NOT NULL,
    scope_type scope_type_enum NOT NULL,
    granted_by UUID NULL REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (tenant_id, user_id, role, scope_type)
);

CREATE TYPE role_enum AS ENUM ('OWNER', 'MANAGER', 'LOCATION_LEAD', 'STAFF');
CREATE TYPE scope_type_enum AS ENUM ('TENANT_WIDE', 'LOCATION_LIST');

CREATE TABLE role_assignment_locations (
    role_assignment_id UUID NOT NULL REFERENCES role_assignments(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    PRIMARY KEY (role_assignment_id, location_id)
);

ALTER TABLE role_assignments ENABLE ROW LEVEL SECURITY;
-- ... RLS Policy ...

CREATE INDEX idx_role_user ON role_assignments (tenant_id, user_id);
```

**Constraint per Trigger:** Bei `scope_type = TENANT_WIDE` darf kein Eintrag in `role_assignment_locations` existieren. Bei `LOCATION_LIST` muss mind. einer existieren.

### `locations`

```sql
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(200) NOT NULL,
    legal_address_street VARCHAR(200) NOT NULL,
    legal_address_house_number VARCHAR(20) NOT NULL,
    legal_address_postal_code VARCHAR(10) NOT NULL,
    legal_address_city VARCHAR(100) NOT NULL,
    legal_address_country_code CHAR(2) NOT NULL DEFAULT 'DE',
    contact_email VARCHAR(255) NULL,
    contact_phone VARCHAR(50) NULL,
    delivery_zip_codes TEXT[] NOT NULL DEFAULT '{}',
    delivery_radius_km NUMERIC(5,2) NULL,        -- für Phase 2
    status location_status_enum NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE TYPE location_status_enum AS ENUM ('ACTIVE', 'INACTIVE', 'DELETED');

ALTER TABLE locations ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON locations USING (...);
```

### `location_opening_hours`

```sql
CREATE TABLE location_opening_hours (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    day_of_week SMALLINT NOT NULL CHECK (day_of_week BETWEEN 1 AND 7),
    open_time TIME NOT NULL,
    close_time TIME NOT NULL,

    CHECK (open_time < close_time)
);

CREATE TABLE location_opening_exceptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    exception_date DATE NOT NULL,
    closed BOOLEAN NOT NULL DEFAULT true,
    open_time TIME NULL,
    close_time TIME NULL,
    reason VARCHAR(200) NULL
);
```

### `tables`

```sql
CREATE TABLE tables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    number INT NOT NULL,
    display_label VARCHAR(50) NULL,
    tags TEXT[] NOT NULL DEFAULT '{}',
    position_x NUMERIC(10,2) NULL,
    position_y NUMERIC(10,2) NULL,
    qr_token VARCHAR(255) UNIQUE NOT NULL,
    status table_status_enum NOT NULL DEFAULT 'ACTIVE',
    merged_into_group_id UUID NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (tenant_id, location_id, number)
);

CREATE TYPE table_status_enum AS ENUM ('ACTIVE', 'MERGED', 'INACTIVE');
```

### `table_groups`

```sql
CREATE TABLE table_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    active_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    active_until TIMESTAMPTZ NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE table_group_members (
    table_group_id UUID NOT NULL REFERENCES table_groups(id) ON DELETE CASCADE,
    table_id UUID NOT NULL REFERENCES tables(id),
    PRIMARY KEY (table_group_id, table_id)
);
```

### `devices`

```sql
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    name VARCHAR(200) NOT NULL,
    available_modes device_mode_enum[] NOT NULL,
    current_mode device_mode_enum NOT NULL,
    mode_switch_policy mode_switch_policy_enum NOT NULL DEFAULT 'NO_CONFIRM',
    staff_pin_policy staff_pin_policy_enum NOT NULL DEFAULT 'OPTIONAL',
    last_heartbeat_at TIMESTAMPTZ NULL,
    status device_status_enum NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TYPE device_mode_enum AS ENUM (
    'ORDER_TABLET', 'KITCHEN_DISPLAY', 'POS_TERMINAL',
    'STORAGE_SCANNER', 'DELIVERY_DRIVER_APP', 'SELF_ORDER_TERMINAL'
);
CREATE TYPE mode_switch_policy_enum AS ENUM ('NO_CONFIRM', 'MANAGER_PIN');
CREATE TYPE staff_pin_policy_enum AS ENUM ('DISABLED', 'OPTIONAL', 'REQUIRED');
CREATE TYPE device_status_enum AS ENUM ('ACTIVE', 'REVOKED', 'INACTIVE');
```

### `device_tokens`

```sql
CREATE TABLE device_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    device_id UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    issued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ NULL,

    UNIQUE (token_hash)
);

CREATE INDEX idx_device_tokens_active ON device_tokens (token_hash) WHERE revoked_at IS NULL;
```

### `categories`, `menu_items`, `variants`, `options`

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    display_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    category_id UUID NOT NULL REFERENCES categories(id),
    location_id UUID NULL REFERENCES locations(id), -- NULL = Stamm-Item, sonst Standort-eigenes
    name VARCHAR(200) NOT NULL,
    description TEXT NULL,
    base_price_cents BIGINT NOT NULL,
    tax_rate_on_site NUMERIC(5,2) NOT NULL DEFAULT 19.00,
    tax_rate_takeaway_delivery NUMERIC(5,2) NOT NULL DEFAULT 7.00,
    image_url VARCHAR(500) NULL,
    status menu_item_status_enum NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE TYPE menu_item_status_enum AS ENUM ('ACTIVE', 'INACTIVE', 'DELETED');

CREATE TABLE variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    price_delta_cents BIGINT NOT NULL DEFAULT 0,
    is_default BOOLEAN NOT NULL DEFAULT false,
    display_order INT NOT NULL DEFAULT 0
);

CREATE TABLE options (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    price_delta_cents BIGINT NOT NULL DEFAULT 0,
    option_group VARCHAR(100) NULL,
    display_order INT NOT NULL DEFAULT 0
);

CREATE TABLE menu_item_allergens (
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    allergen_code VARCHAR(10) NOT NULL,
    PRIMARY KEY (menu_item_id, allergen_code)
);

CREATE TABLE menu_item_additives (
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    additive_code VARCHAR(10) NOT NULL,
    PRIMARY KEY (menu_item_id, additive_code)
);

CREATE TABLE menu_item_availability (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    day_of_week SMALLINT NOT NULL CHECK (day_of_week BETWEEN 1 AND 7),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL
);
```

### `location_menu_overrides`

```sql
CREATE TABLE location_menu_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    master_menu_item_id UUID NOT NULL REFERENCES menu_items(id),
    disabled BOOLEAN NOT NULL DEFAULT false,
    price_override_cents BIGINT NULL,

    UNIQUE (tenant_id, location_id, master_menu_item_id)
);

CREATE TABLE location_variant_price_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_menu_override_id UUID NOT NULL REFERENCES location_menu_overrides(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES variants(id),
    price_delta_cents BIGINT NOT NULL,
    UNIQUE (location_menu_override_id, variant_id)
);

CREATE TABLE location_option_price_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_menu_override_id UUID NOT NULL REFERENCES location_menu_overrides(id) ON DELETE CASCADE,
    option_id UUID NOT NULL REFERENCES options(id),
    price_delta_cents BIGINT NOT NULL,
    UNIQUE (location_menu_override_id, option_id)
);
```

### `acceptance_policies`

```sql
CREATE TABLE acceptance_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID UNIQUE NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    rules JSONB NOT NULL DEFAULT '[]',           -- siehe acceptance-policies.md
    default_mode acceptance_mode_enum NOT NULL DEFAULT 'AUTO',
    workload_low_threshold INT NOT NULL DEFAULT 5,
    workload_high_threshold INT NOT NULL DEFAULT 15,
    manual_workload_override workload_enum NULL,
    manual_workload_override_until TIMESTAMPTZ NULL,
    manual_mode_override JSONB NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_by UUID NULL REFERENCES users(id)
);

CREATE TYPE acceptance_mode_enum AS ENUM ('AUTO', 'MANUAL_ACCEPT', 'MANUAL_REJECT_WINDOW');
CREATE TYPE workload_enum AS ENUM ('LOW', 'MEDIUM', 'HIGH');
```

### `customers`, `customer_addresses`

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NULL,             -- NULL bei Magic-Link-only
    full_name VARCHAR(200) NOT NULL,
    phone VARCHAR(50) NULL,
    status customer_status_enum NOT NULL DEFAULT 'ACTIVE',
    email_verified_at TIMESTAMPTZ NULL,
    failed_login_attempts INT NOT NULL DEFAULT 0,
    locked_until TIMESTAMPTZ NULL,
    last_login_at TIMESTAMPTZ NULL,
    anonymized_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (tenant_id, email)
);

CREATE TYPE customer_status_enum AS ENUM ('ACTIVE', 'ANONYMIZED');

CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    label VARCHAR(50) NOT NULL,
    address_street VARCHAR(200) NOT NULL,
    address_house_number VARCHAR(20) NOT NULL,
    address_postal_code VARCHAR(10) NOT NULL,
    address_city VARCHAR(100) NOT NULL,
    address_country_code CHAR(2) NOT NULL DEFAULT 'DE',
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE magic_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    used_at TIMESTAMPTZ NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `orders`, `order_items`, `payments`

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    table_id UUID NULL REFERENCES tables(id),
    table_group_id UUID NULL REFERENCES table_groups(id),
    customer_id UUID NULL REFERENCES customers(id),
    customer_snapshot JSONB NULL,                -- Gast-Bestellung
    staff_user_id UUID NULL REFERENCES users(id),
    device_id UUID NULL REFERENCES devices(id),
    order_type order_type_enum NOT NULL,
    order_mode order_mode_enum NOT NULL,
    status order_status_enum NOT NULL,
    acceptance_decision JSONB NULL,
    estimated_ready_at TIMESTAMPTZ NULL,
    scheduled_for_at TIMESTAMPTZ NULL,
    delivery_address JSONB NULL,
    notes TEXT NULL,
    total_gross_cents BIGINT NOT NULL DEFAULT 0,
    total_net_cents BIGINT NOT NULL DEFAULT 0,
    total_tax_cents BIGINT NOT NULL DEFAULT 0,
    tax_breakdown JSONB NOT NULL DEFAULT '{}',
    tax_overridden BOOLEAN NOT NULL DEFAULT false,
    tax_override_reason TEXT NULL,
    tse_signature_id UUID NULL,
    version INT NOT NULL DEFAULT 0,              -- Optimistic Locking
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at TIMESTAMPTZ NULL,
    accepted_at TIMESTAMPTZ NULL,
    in_progress_at TIMESTAMPTZ NULL,
    ready_at TIMESTAMPTZ NULL,
    completed_at TIMESTAMPTZ NULL,
    paid_at TIMESTAMPTZ NULL,
    cancelled_at TIMESTAMPTZ NULL,
    rejected_at TIMESTAMPTZ NULL
);

CREATE TYPE order_type_enum AS ENUM ('ON_SITE', 'PICKUP', 'DELIVERY');
CREATE TYPE order_mode_enum AS ENUM ('SELF_ORDER_INSTANT', 'OPEN_TAB');
CREATE TYPE order_status_enum AS ENUM (
    'DRAFT', 'CONFIRMED', 'AWAITING_ACCEPTANCE', 'ACCEPTED',
    'IN_PROGRESS', 'READY', 'SERVED', 'PICKED_UP',
    'OUT_FOR_DELIVERY', 'DELIVERED', 'PAID',
    'CANCELLED', 'REJECTED', 'NEEDS_RESOLUTION'
);

CREATE INDEX idx_orders_tenant_location_status ON orders (tenant_id, location_id, status);
CREATE INDEX idx_orders_table ON orders (tenant_id, table_id, status) WHERE status NOT IN ('PAID', 'CANCELLED', 'REJECTED');
CREATE INDEX idx_orders_created ON orders (tenant_id, created_at DESC);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id UUID NOT NULL REFERENCES menu_items(id),
    variant_id UUID NOT NULL REFERENCES variants(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    selected_options JSONB NOT NULL DEFAULT '[]',  -- Liste der Option-IDs
    price_snapshot_cents BIGINT NOT NULL,
    tax_rate_snapshot NUMERIC(5,2) NOT NULL,
    status order_item_status_enum NOT NULL,
    rejected_reason TEXT NULL,
    notes TEXT NULL,
    sequence_number INT NOT NULL,                  -- Reihenfolge in Order
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TYPE order_item_status_enum AS ENUM (
    'REQUESTED', 'ACCEPTED', 'IN_PREPARATION', 'READY',
    'SERVED', 'PICKED_UP', 'DELIVERED',
    'CANCELLED', 'REJECTED_BY_KITCHEN'
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    order_id UUID NOT NULL REFERENCES orders(id),
    amount_cents BIGINT NOT NULL,
    method payment_method_enum NOT NULL,
    status payment_status_enum NOT NULL DEFAULT 'INITIATED',
    payment_provider_ref VARCHAR(500) NULL,
    refund_amount_cents BIGINT NULL,
    refund_reason TEXT NULL,
    initiated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ NULL,
    refunded_at TIMESTAMPTZ NULL
);

CREATE TYPE payment_method_enum AS ENUM ('CASH', 'CARD', 'ONLINE', 'OTHER');
CREATE TYPE payment_status_enum AS ENUM (
    'INITIATED', 'SUCCEEDED', 'FAILED', 'REFUNDED', 'PARTIALLY_REFUNDED'
);
```

### `waiter_calls`

```sql
CREATE TABLE waiter_calls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    table_id UUID NOT NULL REFERENCES tables(id),
    triggered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    trigger_source waiter_call_source_enum NOT NULL,
    status waiter_call_status_enum NOT NULL DEFAULT 'OPEN',
    acknowledged_by UUID NULL REFERENCES users(id),
    resolved_at TIMESTAMPTZ NULL,
    ip_fingerprint VARCHAR(64) NULL
);

CREATE TYPE waiter_call_source_enum AS ENUM ('TABLE_QR_GUEST', 'GUEST_DEVICE');
CREATE TYPE waiter_call_status_enum AS ENUM ('OPEN', 'ACKNOWLEDGED', 'RESOLVED', 'IGNORED', 'BLOCKED');

CREATE TABLE table_call_blocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    table_id UUID NOT NULL REFERENCES tables(id),
    blocked_until TIMESTAMPTZ NOT NULL,
    blocked_by UUID NOT NULL REFERENCES users(id),
    reason TEXT NULL
);
```

### `tse_signatures`

```sql
CREATE TABLE tse_signatures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    order_id UUID NOT NULL REFERENCES orders(id),
    signature TEXT NOT NULL,
    transaction_counter BIGINT NOT NULL,
    transaction_started_at TIMESTAMPTZ NOT NULL,
    transaction_finished_at TIMESTAMPTZ NOT NULL,
    process_data TEXT NOT NULL,
    serial_number VARCHAR(255) NOT NULL,
    public_key TEXT NOT NULL,
    is_fake BOOLEAN NOT NULL DEFAULT false,      -- TRUE bei FakeTseAdapter
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audit-Tabellen

Definiert in [`audit-log.md`](audit-log.md).

## RLS auf Mandanten-Tabellen

Pro Mandanten-Tabelle gleiches Pattern:

```sql
ALTER TABLE <table_name> ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON <table_name>
    USING (tenant_id = current_setting('app.tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

## Indizes (Standard-Set)

Für alle Mandanten-Tabellen:
- `tenant_id` als erster Index-Bestandteil
- Häufig zusammen mit `location_id`, `status`, oder Zeit-Spalten

Beispiele für Reporting:
```sql
CREATE INDEX idx_orders_reporting ON orders (tenant_id, location_id, paid_at) WHERE status = 'PAID';
CREATE INDEX idx_order_items_reporting ON order_items (tenant_id, order_id, menu_item_id);
```

## Update-Trigger für `updated_at`

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Pro Tabelle:
CREATE TRIGGER set_updated_at_<table_name>
BEFORE UPDATE ON <table_name>
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

## Migrations-Konvention

Flyway-Dateien in `db/migration`:
- `V001__init.sql` — Basis-Schema
- `V002__tenants_users.sql`
- `V003__locations_tables.sql`
- ... usw.

**Pflichten:**
- Idempotente Migrationen (mit `IF NOT EXISTS`).
- Reversibilität nicht erforderlich (Flyway-Konvention: forward only).
- Daten-Migrationen separat als `V*__data_*` mit Bedacht auf Performance.
- Tests im CI führen Migration auf leerer DB UND auf Seed-DB aus.
