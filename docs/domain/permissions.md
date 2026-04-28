# Permissions und Rollen

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Dieses Dokument ist die verbindliche Quelle für alle Berechtigungen und Rollen-Templates. Die Implementierung MUSS:

1. Permissions als fixe Enum-Werte im Code definieren (keine Datenbank-Konfiguration).
2. Rollen-Templates als fixe Konfiguration im Code definieren.
3. Jede Permission-Prüfung im Service-Layer und zusätzlich, wo möglich, auf DB-Ebene (RLS).

## Permission-Hierarchie

Es gibt zwei Permission-Klassen:

- **Plattform-Permissions** (`PLATFORM_*`) — gelten für Plattform-Admin-Konten, scope-frei.
- **Mandanten-Permissions** — gelten innerhalb eines Tenants, mit Scope (TENANT_WIDE oder LOCATION_LIST).

## Plattform-Permissions

| Permission | Bedeutung |
|---|---|
| `PLATFORM_TENANT_VIEW` | Tenants einsehen (Liste, Detail, Stammdaten) |
| `PLATFORM_TENANT_CREATE` | Tenant anlegen |
| `PLATFORM_TENANT_SUSPEND` | Tenant sperren / reaktivieren |
| `PLATFORM_TENANT_DELETE` | Tenant löschen (Soft-Delete) |
| `PLATFORM_TENANT_LICENSE_EDIT` | Lizenzpaket bearbeiten |
| `PLATFORM_TENANT_IMPERSONATE` | Als Tenant-Inhaber einloggen (Support) |
| `PLATFORM_USAGE_VIEW` | Plattform-weite Statistiken einsehen |
| `PLATFORM_CONFIG_EDIT` | Plattform-Defaults bearbeiten |
| `PLATFORM_ADMIN_MANAGE` | Andere Admin-Konten verwalten |
| `PLATFORM_AUDIT_VIEW` | Plattform-Audit-Log einsehen |

## Plattform-Rollen-Templates

```
SuperAdmin = {
  PLATFORM_TENANT_VIEW,
  PLATFORM_TENANT_CREATE,
  PLATFORM_TENANT_SUSPEND,
  PLATFORM_TENANT_DELETE,
  PLATFORM_TENANT_LICENSE_EDIT,
  PLATFORM_TENANT_IMPERSONATE,
  PLATFORM_USAGE_VIEW,
  PLATFORM_CONFIG_EDIT,
  PLATFORM_ADMIN_MANAGE,
  PLATFORM_AUDIT_VIEW
}

SupportAdmin = {
  PLATFORM_TENANT_VIEW,
  PLATFORM_TENANT_IMPERSONATE,
  PLATFORM_USAGE_VIEW,
  PLATFORM_AUDIT_VIEW
}

BillingAdmin = {
  PLATFORM_TENANT_VIEW,
  PLATFORM_TENANT_CREATE,
  PLATFORM_TENANT_LICENSE_EDIT,
  PLATFORM_USAGE_VIEW
}
```

**Pflichtinvariante:** Es MUSS mindestens ein aktiver `SuperAdmin`-Account existieren. Letzten SuperAdmin zu degradieren oder zu löschen ist verboten.

## Mandanten-Permissions

### Tenant-Verwaltung
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `TENANT_DELETE` | TENANT_WIDE | Restaurant löschen |
| `TENANT_OWNERSHIP_TRANSFER` | TENANT_WIDE | Inhaberschaft übertragen |
| `TENANT_SETTINGS_EDIT` | TENANT_WIDE | Tenant-Stammdaten bearbeiten |

### Benutzer-Verwaltung
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `USER_INVITE` | TENANT_WIDE oder LOCATION_LIST | Benutzer einladen |
| `USER_ROLE_EDIT` | TENANT_WIDE oder LOCATION_LIST | Rollen anderer Benutzer ändern |
| `USER_REMOVE` | TENANT_WIDE oder LOCATION_LIST | Benutzer entfernen |

### Standort-Verwaltung
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `LOCATION_CREATE` | TENANT_WIDE | Standort anlegen |
| `LOCATION_EDIT` | LOCATION_LIST | Standort bearbeiten |
| `LOCATION_DELETE` | TENANT_WIDE | Standort löschen |
| `TABLE_MANAGE` | LOCATION_LIST | Tische und Tischplan |
| `DEVICE_CREATE` | LOCATION_LIST | Gerät anlegen, Token erzeugen |
| `DEVICE_REVOKE` | LOCATION_LIST | Geräte-Token widerrufen |

### Speisekarten
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `MENU_MASTER_EDIT` | TENANT_WIDE | Stamm-Speisekarte bearbeiten |
| `MENU_LOCATION_EDIT` | LOCATION_LIST | Standort-Speisekarte (Override) |

### Bestellungen
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `ORDER_CREATE` | LOCATION_LIST | Bestellung anlegen |
| `ORDER_MODIFY` | LOCATION_LIST | Items hinzufügen/ändern |
| `ORDER_CANCEL_PRE_KITCHEN` | LOCATION_LIST | Storno vor Versand |
| `ORDER_CANCEL_POST_KITCHEN` | LOCATION_LIST | Storno nach IN_PROGRESS |
| `ORDER_CHARGE` | LOCATION_LIST | Bestellung abrechnen |
| `ORDER_TAX_OVERRIDE` | LOCATION_LIST | Steuersatz manuell überschreiben |

### Online-Bestellungen
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `ONLINE_ORDER_ACCEPT_REJECT` | LOCATION_LIST | Online-Bestellungen manuell annehmen/ablehnen |
| `ACCEPTANCE_POLICY_EDIT` | LOCATION_LIST | Annahme-Policy konfigurieren |

### Kellner-Rufe
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `WAITER_CALL_VIEW` | LOCATION_LIST | Eingehende Kellner-Rufe sehen |
| `WAITER_CALL_BLOCK` | LOCATION_LIST | Tisch temporär blocken |

### Reporting und Audit
| Permission | Scope-Art | Bedeutung |
|---|---|---|
| `REPORT_VIEW` | TENANT_WIDE oder LOCATION_LIST | Reporting einsehen |
| `REPORT_DRILLDOWN` | TENANT_WIDE oder LOCATION_LIST | Drill-Down auf Bestellebene |
| `AUDIT_VIEW` | TENANT_WIDE oder LOCATION_LIST | Audit-Log einsehen |

## Mandanten-Rollen-Templates

### Inhaber (Owner)

**Scope:** ausschließlich TENANT_WIDE.
**Permissions:** ALLE Mandanten-Permissions.

```
Owner = {
  TENANT_DELETE,
  TENANT_OWNERSHIP_TRANSFER,
  TENANT_SETTINGS_EDIT,
  USER_INVITE, USER_ROLE_EDIT, USER_REMOVE,
  LOCATION_CREATE, LOCATION_EDIT, LOCATION_DELETE,
  TABLE_MANAGE,
  DEVICE_CREATE, DEVICE_REVOKE,
  MENU_MASTER_EDIT, MENU_LOCATION_EDIT,
  ORDER_CREATE, ORDER_MODIFY,
  ORDER_CANCEL_PRE_KITCHEN, ORDER_CANCEL_POST_KITCHEN,
  ORDER_CHARGE, ORDER_TAX_OVERRIDE,
  ONLINE_ORDER_ACCEPT_REJECT, ACCEPTANCE_POLICY_EDIT,
  WAITER_CALL_VIEW, WAITER_CALL_BLOCK,
  REPORT_VIEW, REPORT_DRILLDOWN, AUDIT_VIEW
}
```

**Pflichtinvariante:** Jeder aktive Tenant MUSS mindestens einen User mit `Owner`-Rolle haben.

### Geschäftsführer (Manager)

**Scope:** ausschließlich TENANT_WIDE.
**Permissions:** alle außer `TENANT_DELETE`, `TENANT_OWNERSHIP_TRANSFER`. Außerdem:
- Darf KEINE Inhaber oder Geschäftsführer entfernen (`USER_REMOVE` wirkt nur auf Standortleiter und Mitarbeiter).

```
Manager = Owner \ {
  TENANT_DELETE,
  TENANT_OWNERSHIP_TRANSFER
}
```

**Sonderregel `USER_REMOVE`:** Im Code MUSS geprüft werden, dass das Ziel keine Owner- oder Manager-Rolle hat. Diese Geschäftsregel ist nicht durch Permission-Set abbildbar.

### Standortleiter (LocationLead)

**Scope:** LOCATION_LIST (1..n Standorte).
**Permissions:**

```
LocationLead = {
  LOCATION_EDIT,
  TABLE_MANAGE,
  DEVICE_CREATE, DEVICE_REVOKE,
  MENU_LOCATION_EDIT,
  ORDER_CREATE, ORDER_MODIFY,
  ORDER_CANCEL_PRE_KITCHEN, ORDER_CANCEL_POST_KITCHEN,
  ORDER_CHARGE, ORDER_TAX_OVERRIDE,
  ONLINE_ORDER_ACCEPT_REJECT, ACCEPTANCE_POLICY_EDIT,
  WAITER_CALL_VIEW, WAITER_CALL_BLOCK,
  REPORT_VIEW, REPORT_DRILLDOWN, AUDIT_VIEW,
  USER_INVITE     # eingeschränkt: nur Staff in eigenem Scope
}
```

**Sonderregel für `USER_INVITE`:** Standortleiter darf nur `Staff`-Rollen einladen, und nur für Standorte aus seinem eigenen Scope.

### Mitarbeiter (Staff)

**Scope:** LOCATION_LIST (1..n Standorte).
**Permissions:**

```
Staff = {
  ORDER_CREATE, ORDER_MODIFY,
  ORDER_CANCEL_PRE_KITCHEN,
  ORDER_CHARGE,
  WAITER_CALL_VIEW
}
```

**Hinweis:** Staff hat KEIN `ORDER_CANCEL_POST_KITCHEN`. Stornos nach IN_PROGRESS erfordern Manager-PIN-Bestätigung am Gerät.

## Manager-PIN-Mechanismus

Manche Aktionen sind für Staff verboten, müssen aber im Tagesbetrieb pragmatisch ermöglicht werden. Der Manager-PIN-Mechanismus löst das ohne Sitzungswechsel:

1. Staff triggert verbotene Aktion am Gerät.
2. UI zeigt PIN-Modal: „Diese Aktion erfordert Manager-Bestätigung."
3. Manager (Owner/Manager/LocationLead) gibt seine PIN am Gerät ein.
4. System validiert PIN gegen aktive User dieses Tenants mit der erforderlichen Permission.
5. Aktion wird ausgeführt; Audit-Log attribuiert die Aktion auf den Manager mit `initiatedByUserId=<staff>` als Kontext.

**Permissions, die per Manager-PIN aufgerufen werden können:**
- `ORDER_CANCEL_POST_KITCHEN`
- `ORDER_TAX_OVERRIDE`
- Modus-Wechsel am Gerät (falls `modeSwitchPolicy = MANAGER_PIN`)

**Implementierungs-Anforderung:** Der Manager-PIN-Mechanismus DARF NICHT als Login-Bypass missbraucht werden. Er gilt ausschließlich pro einzelner Aktion und führt zu KEINER neuen Session.

## Permission-Auswertung

### Bei Service-Calls
```
Pseudo-Code:
function checkPermission(user, permission, context):
    if user is impersonated by admin:
        actor = admin
    else:
        actor = user

    for assignment in user.roleAssignments:
        if permission in assignment.role.permissions:
            if assignment.scope.type == TENANT_WIDE:
                return GRANTED
            elif assignment.scope.type == LOCATION_LIST:
                if context.locationId in assignment.scope.locationIds:
                    return GRANTED
    return DENIED
```

### Bei DB-Zugriff
- RLS-Policies erzwingen `tenant_id`-Filter automatisch.
- Application-Schicht prüft zusätzlich Permission und Scope.
- **Defense in Depth:** Ohne BEIDE Schichten ist die Mandantentrennung nicht garantiert.

## Permission-Mapping zu Use Cases

Eine vollständige Matrix befindet sich in [`use-cases.md`](use-cases.md). Stichprobe:

| Use Case | Permission(s) | Notiz |
|---|---|---|
| UC-A01 (Tenant anlegen) | `PLATFORM_TENANT_CREATE` | Plattform-Scope |
| UC-R10 (Stamm-Speisekarte) | `MENU_MASTER_EDIT` | nur TENANT_WIDE |
| UC-R11 (Standort-Speisekarte) | `MENU_LOCATION_EDIT` | LOCATION_LIST |
| UC-B05 (An Küche senden) | `ORDER_CREATE` | LOCATION_LIST |
| UC-B06 (Storno nach IN_PROGRESS) | `ORDER_CANCEL_POST_KITCHEN` (via Manager-PIN für Staff) | LOCATION_LIST |
| UC-B09 (Abrechnen) | `ORDER_CHARGE` | LOCATION_LIST |
| UC-B09-Steuer-Override | `ORDER_TAX_OVERRIDE` | LOCATION_LIST |
| UC-O02 (Online annehmen) | `ONLINE_ORDER_ACCEPT_REJECT` | LOCATION_LIST |
| UC-A05 (Impersonate) | `PLATFORM_TENANT_IMPERSONATE` | Plattform-Scope |

## Zukunft (Phase 2+)

- 2FA-Erzwingung pro Tenant (Permission `TENANT_REQUIRE_2FA`).
- Frei konfigurierbare Rollen für Tenants mit Pro-Lizenz (Phase 3).
- Audit-Log-Export-Permission (Phase 2).
