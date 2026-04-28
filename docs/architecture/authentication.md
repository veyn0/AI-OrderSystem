# Authentifizierung

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Plattform unterstützt **vier Authentifizierungs-Pfade**:

1. **Plattform-Admins** — E-Mail + Passwort, server-side Session
2. **Restaurant-Benutzer** — E-Mail + Passwort, server-side Session
3. **Endkunden** — E-Mail + Passwort ODER Magic Link, server-side Session
4. **Geräte** — langlebiger Bearer-Token

Native Mobile (Phase 2) nutzt OAuth2 Authorization Code + PKCE.

## Allgemeine Sicherheits-Standards

| Aspekt | Standard |
|---|---|
| Passwort-Hashing | Argon2id (Standard-Spring-Security-Encoder) |
| Argon2-Parameter | memory=64MB, iterations=3, parallelism=2 (review) |
| Session-Cookie | HttpOnly, Secure (in Prod), SameSite=Lax |
| Session-Timeout | 8 Stunden inaktiv |
| TLS | 1.2+ Pflicht in Prod, kein TLS in Demo (HTTP localhost) |
| Brute-Force | Lockout nach 10 Fehlversuchen, exponentiell verlängert |

## Plattform-Admin-Login

**Endpoint:** `/admin/login` (separat von Restaurant-Login).

**Ablauf:**
1. Admin gibt E-Mail + Passwort ein.
2. System prüft `admins`-Tabelle, hasht eingegebenes Passwort, vergleicht.
3. Bei Erfolg: Session mit Marker `principalType=PLATFORM_ADMIN`.
4. Tenant-Kontext: KEINER (Plattform-Endpoints).
5. Zugriff auf `/admin/**` Routen.

**Rate-Limiting:** maximal 10 Login-Versuche pro IP / Stunde.

## Restaurant-Benutzer-Login

**Endpoint:** `/login` (Tenant-aware).

**Ablauf:**
1. Benutzer öffnet `https://<tenant>.platform.de/login` ODER `https://www.<custom-domain>/login`.
2. System ermittelt `tenantId` aus URL.
3. Benutzer gibt E-Mail + Passwort ein.
4. System sucht in `users` mit `tenant_id` und `email`.
5. Bei Erfolg: Session mit Marker `principalType=RESTAURANT_USER`, `tenantId`, `userId`.
6. Tenant-Kontext: `tenantId`.

**E-Mail-Eindeutigkeit:** Innerhalb eines Tenants UNIQUE; gleiche E-Mail kann in unterschiedlichen Tenants existieren — jedes ist eigenes Konto.

## Endkunden-Login

### Variante A: E-Mail + Passwort
1. Endkunde öffnet `<tenant>/customer/login`.
2. E-Mail + Passwort.
3. System sucht in `customers` mit `(tenant_id, email)`.
4. Bei Erfolg: Customer-Session, getrennter Cookie-Pfad `/customer`.

### Variante B: Magic Link
1. Endkunde gibt nur E-Mail ein.
2. System erzeugt Einmal-Token (256-Bit, 30 min Gültigkeit), speichert in `magic_links`.
3. System sendet Link per Mail (Stub: virtuelles Postfach).
4. Endkunde klickt Link → Token wird konsumiert, Session erstellt.

**Tabelle `magic_links`:**
```
id, tenant_id, customer_id, token_hash, used_at, expires_at, created_at
```

Sicherheit:
- Token nur als Hash gespeichert.
- Bei Klick: ein Klick verbraucht Token, weitere Klicks → 400.
- Rate-Limit: max. 5 Magic Links pro Customer / Stunde.

## Geräte-Authentifizierung

### Token-Erstellung
1. Restaurant-Benutzer mit `DEVICE_CREATE` legt Gerät an.
2. System generiert 256-Bit-Random-Token (`SecureRandom`).
3. Hash (Argon2id) wird in `device_tokens` gespeichert.
4. Klartext-Token wird **einmalig** in der UI angezeigt (mit QR-Code).

### Pairing
1. Gerät hat Eingabefeld für Token oder QR-Scanner.
2. Gerät speichert Token im Android Keystore (Phase 2 Native) bzw. localStorage (PWA, MVP).
3. Erste API-Anfrage mit `Authorization: Bearer <token>`.

### Authentifizierung pro Request
1. Gerät sendet `Authorization: Bearer <token>` bei jedem Request.
2. Server hasht Token, sucht in `device_tokens`.
3. Bei Erfolg: Tenant-Kontext aus `device.tenant_id`, Permissions sind die fixen Geräte-Permissions (siehe unten).

### Heartbeat
1. Gerät sendet `POST /api/v1/device/heartbeat` alle 60 s.
2. System aktualisiert `device.last_heartbeat_at`.
3. Job (täglich): Geräte ohne Heartbeat seit > 7 Tage → `status = INACTIVE`. Token-Auth schlägt dann fehl mit 401.

### Revocation
1. Restaurant-Benutzer mit `DEVICE_REVOKE` widerruft Token.
2. `device_tokens.revoked_at = now()`.
3. Nächste Auth-Request → 401.

### Geräte-Permissions
Geräte haben fixe Permissions abhängig vom **aktuellen Modus** (`device.current_mode`):

| Modus | Permissions |
|---|---|
| ORDER_TABLET | ORDER_CREATE, ORDER_MODIFY, ORDER_CHARGE (im Standort-Scope), ORDER_CANCEL_PRE_KITCHEN, WAITER_CALL_VIEW |
| KITCHEN_DISPLAY | nur Read auf Order, Item-Status setzen |
| SELF_ORDER_TERMINAL | ORDER_CREATE (eingeschränkt: nur SELF_ORDER_INSTANT) |
| POS_TERMINAL (Phase 2) | ORDER_CHARGE, ORDER_TAX_OVERRIDE |
| STORAGE_SCANNER (Phase 3) | Lager-Permissions |
| DELIVERY_DRIVER_APP (Phase 3) | Lieferungs-Permissions |

Wenn ein Mitarbeiter sich am Gerät identifiziert (PIN), dann werden zusätzlich seine User-Permissions berücksichtigt — der **stärkere** Wert gilt.

## Mitarbeiter-PIN am Gerät

**Definition:** 4-6-stellige PIN, getrennt vom Login-Passwort.

**Speicherung:** `users.pin` enthält Argon2-Hash der PIN.

**Verwendung am Gerät:**
1. Gerät zeigt PIN-Eingabe (abhängig von `device.staff_pin_policy`).
2. Eingegebene PIN wird gehasht und gegen alle aktiven User dieses Tenants verglichen, deren Standort-Scope den Gerät-Standort enthält.
3. Bei Match: `staffUserId` ist gesetzt, Aktionen werden auf den User attribuiert.
4. Auto-Lock nach 5 min Inaktivität (zurück zu PIN-Eingabe).

**Sicherheits-Anforderung:** PIN ist NICHT geeignet für Web-Login. Sie ist nur am Gerät gültig, weil dort der Geräte-Token bereits authentifiziert hat.

**Brute-Force:** Pro Gerät max. 5 PIN-Fehlversuche → Gerät-Lockout 5 min.

## Manager-PIN-Mechanismus

Ein Mitarbeiter kann am Gerät eine Aktion auslösen, die seine eigene Permission nicht erlaubt. Das System fordert dann eine Manager-PIN-Bestätigung.

**Ablauf:**
1. Mitarbeiter klickt z.B. „Storno" bei `IN_PROGRESS`-Item.
2. Backend antwortet mit `403 PERMISSION_REQUIRED, requires=ORDER_CANCEL_POST_KITCHEN`.
3. Frontend zeigt Modal „Manager-PIN".
4. Manager gibt PIN ein → API-Aufruf `POST /api/v1/auth/manager-confirm` mit Aktion und PIN.
5. Server prüft: PIN gehört zu einem User mit der erforderlichen Permission im Standort-Scope.
6. Bei Erfolg: Server gibt einen einmal-gültigen Confirm-Token zurück.
7. Frontend wiederholt ursprünglichen Request mit Confirm-Token.
8. Audit-Log attribuiert Aktion auf Manager mit `initiatedByUserId=<staff>`.

**Confirm-Token:** Lebenszeit 60 s, einmal verbrauchbar, an die spezifische Aktion gebunden.

## Session-Management

**Spring Session** mit Profil-spezifischem Backing Store:

| Profil | Backing Store |
|---|---|
| demo | In-Memory (`MapSessionRepository`) |
| dev | Redis (Container) |
| prod | Redis |

**Session-Daten:**
- Principal-Typ
- Principal-ID (User/Customer/Admin)
- Tenant-Kontext (NULL für Plattform-Admin)
- Berechtigungs-Cache
- IP, User-Agent (für Audit)

**Logout:** Cookie wird invalidiert UND Session-Eintrag im Store gelöscht. Sofortiger Effekt.

**Concurrent Sessions:** Erlaubt (User kann an mehreren Geräten gleichzeitig eingeloggt sein).

## Passwort-Validierung

| Anforderung | Wert |
|---|---|
| Minimale Länge | 10 Zeichen |
| Min. Buchstaben | 1 |
| Min. Ziffern | 1 |
| Min. Sonderzeichen | 1 |
| Max. Länge | 256 Zeichen |
| Verboten | Top-100 Most-Common-Passwords (Bibliothek `passay` oder eigene Liste) |

**Bei Passwortwechsel:** Altes Passwort darf nicht wiederverwendet werden (letzte 5 Hashes in `password_history`).

## Passwort-Vergessen

1. User fordert Reset auf `/login/forgot`.
2. Server prüft: existiert E-Mail in `users` für diesen Tenant?
3. **Antwort ist immer "E-Mail wurde gesendet, falls Konto existiert"** — keine Auskunft über Existenz.
4. Falls existent: Reset-Token (256-Bit, 30 min Gültigkeit) erzeugen, Mail mit Link.
5. Klick auf Link → neues Passwort setzen → Token verbrauchen.

## 2FA (Phase 2)

Vorbereitet, im MVP NICHT aktiv.

**Datenmodell:**
- `users.totp_secret VARCHAR(32) NULL` — verschlüsselt mit App-Key (Datenbank-Verschlüsselung at-rest reicht).
- `users.totp_enabled BOOLEAN DEFAULT false`.
- `users.recovery_codes TEXT[]` — Hashes der Recovery-Codes.

**Aktivierung Phase 2:**
1. User generiert TOTP-Geheimnis im Profil.
2. Scannt QR-Code mit Authenticator-App.
3. Bestätigt mit aktuellem TOTP-Code.
4. `totp_enabled = true`.
5. Bei nächstem Login zusätzlich TOTP-Code abgefragt.

## OAuth2 für Mobile (Phase 2)

Vorbereitet im MVP, aktiv ab Phase 2.

**Konfiguration:**
- Authorization Server: Spring Authorization Server (eigenständiger Modul-Pfad oder eigene Library).
- Resource Server: gleiche App, mit `spring-boot-starter-oauth2-resource-server`.

**Client-Konfiguration für Android-App:**
- Client-ID: `restaurant-app-android`
- Client-Type: Public (PKCE)
- Redirect-URI: `de.deinrestaurant.app://oauth-callback`
- Scopes: `read:orders`, `write:orders`, `read:menu`, etc.

**Flow:**
1. App öffnet System-Browser zu `/oauth2/authorize?...&code_challenge=...`.
2. User loggt sich ein (Restaurant-User-Login).
3. Authorization Code an Redirect-URI.
4. App tauscht Code gegen Access-Token + Refresh-Token (`/oauth2/token`).
5. Access-Token (15 min) in Memory, Refresh-Token (30 d) im Android Keystore.
6. Refresh-Token-Rotation: bei jedem Refresh wird neuer Refresh-Token ausgegeben, alter verbraucht.

## Auth-Flows als Tabelle

| Akteur | Auth-Mechanismus | Session-Typ | Tenant-Kontext |
|---|---|---|---|
| Plattform-Admin | E-Mail+Passwort | Server-Session | KEINER |
| Restaurant-User | E-Mail+Passwort | Server-Session | tenantId aus User |
| Endkunde | E-Mail+Passwort oder Magic Link | Server-Session (separater Cookie-Pfad) | tenantId aus URL+Customer |
| Gerät | Bearer-Token | Stateless (per Request) | tenantId aus Device |
| Mobile (Phase 2) | OAuth2 + PKCE | Bearer Access-Token | tenantId aus Token-Claim |

## Audit-Anforderungen

Folgende Auth-Ereignisse MÜSSEN im Audit-Log landen:
- Login erfolgreich (Akteur, IP, User-Agent)
- Login fehlgeschlagen (E-Mail, IP, User-Agent, Grund)
- Lockout
- Logout
- Passwort geändert
- Reset-Mail angefordert
- Reset durchgeführt
- Magic-Link-Versand
- Magic-Link-Verbrauch
- Manager-PIN-Bestätigung (mit Initiator und Aktion)
- 2FA-Aktivierung/Deaktivierung (Phase 2)
- Token-Erstellung/Revocation
- Impersonation Start/Ende
