# ADR-012: Geräte-Tokens (Bearer, langlebig)

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Geräte (Bestelltablet, KDS, Self-Order-Terminal, POS, Fahrer-App) brauchen Authentifizierung für API-Zugriff. Sie sind im Restaurant fest installiert / einem Bediener zugeordnet, nicht für eine Person personalisiert.

Anforderungen:
- Pairing einmal beim Aufstellen.
- Token überlebt Geräte-Reboots, App-Updates.
- Revocation jederzeit möglich.
- Diebstahl / Verlust wird erkannt und mitigiert.

## Entscheidung

**Langlebige Bearer-Tokens** pro Gerät.

- **Erstellung:** Restaurant-User mit Permission `DEVICE_CREATE` legt Gerät an. System generiert 256-Bit-Random-Token via `SecureRandom`, hasht ihn (Argon2id) und speichert nur den Hash in `device_tokens`.
- **Anzeige:** Klartext-Token wird **einmalig** in der UI mit QR-Code angezeigt. Danach nicht mehr abrufbar.
- **Pairing:** Gerät erfasst Token (manuell oder QR). Token wird im Geräte-Storage abgelegt:
  - PWA/Web-MVP: `localStorage` (Browser-Storage des dedizierten Browsers).
  - Native Android (Phase 2): Android Keystore.
- **Verwendung:** `Authorization: Bearer <token>` bei jeder API-Anfrage.
- **Heartbeat:** Geräte schicken alle 60 s `POST /api/v1/device/heartbeat`. System aktualisiert `last_heartbeat_at`.
- **Auto-Lockout:** Geräte ohne Heartbeat seit > 7 Tagen werden auf `INACTIVE` gesetzt. Token-Auth schlägt fehl.
- **Revocation:** Restaurant-User mit `DEVICE_REVOKE` setzt `revoked_at`. Sofortige Wirkung.

## Alternativen

### Geräte als Restaurant-Benutzer behandeln
Verworfen.
- Geräte sind nicht personenbezogen.
- Permissions für Geräte sind anders strukturiert (modus-abhängig).
- User-Konten würden mit „virtuellen Mitarbeitern" für Geräte zugespammt.

### JWT für Geräte
Verworfen.
- Revocation aufwendig.
- Tokens müssten kurz-lebig sein, dann Refresh-Token-Komplexität.
- Bearer-Token mit DB-Hash erfüllt den gleichen Zweck einfacher.

### OAuth2 Client Credentials Flow
Erwogen, verworfen.
- Komplex für Geräte-Pairing.
- Bearer-Token mit DB-Hash ist äquivalent in der Wirkung, simpler.

### Public-Key-basiertes Pairing
Erwogen für Phase 2 (mehr Sicherheit), verworfen für MVP.
- Komplexität in der Implementierung.
- Bearer-Token-Sicherheit ausreichend, wenn Pflege (Revocation, Auto-Lockout) gewährleistet ist.

## Konsequenzen

### Positiv
- **Einfaches Pairing**: QR scannen, fertig.
- **Sofortige Revocation**: Hash gelöscht/gesetzt = Token tot.
- **Hash-Speicherung**: Token-Diebstahl aus DB-Backup unbrauchbar (würde noch Argon2-Kollision brauchen).
- **Auto-Lockout** schützt verlorene/vergessene Geräte.
- **Audit-Log** zeichnet Geräte-Aktivität auf — Anomalie-Erkennung möglich.

### Negativ
- **Bei Token-Diebstahl** mit aktivem Gerät: Angreifer hat bis Revocation Zugriff. PIN-Schutz (siehe [ADR-013](013-staff-pin.md)) verringert das Schadenspotential.
- **Restaurant muss Token bei Geräteaustausch revoken** — Disziplin nötig.

### Neutral
- Token-Storage im Browser-localStorage ist akzeptabel für PWA, da das Gerät dediziert ist (nicht ein gemeinsam genutzter Computer).

## Querverweise

- [`../architecture/authentication.md`](../architecture/authentication.md)
- [`../architecture/data-model.md`](../architecture/data-model.md)
- [ADR-013](013-staff-pin.md)
