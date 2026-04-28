# ADR-011: Server-Sessions Web, OAuth2+PKCE Mobile

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform hat verschiedene Klienten:

1. **Web-Browser**: Plattform-Admin, Restaurant-Backoffice, öffentliche Online-Bestellseite, Endkunden-Konto-UI, Demo-UI.
2. **Geräte (Tablet, KDS, Self-Order, etc.)**: Token-Auth ohne Session.
3. **Native Mobile-App** (Android, Phase 2): eingeloggter Restaurant-User, der unterwegs die App nutzt.

Auth-Optionen: Server-Session, JWT, OAuth2 (verschiedene Flows).

## Entscheidung

- **Web (Browser):** Server-Side Sessions via Spring Session.
  - Cookies HttpOnly, Secure, SameSite=Lax.
  - Session-Backing-Store: In-Memory (Demo), Redis (Dev/Prod).
  - 8 h Inaktivitäts-Timeout.

- **Geräte:** langlebiger Bearer-Token (siehe [ADR-012](012-device-tokens.md)).

- **Native Mobile (Phase 2):** OAuth2 Authorization Code Flow + PKCE mit Spring Authorization Server.

## Alternativen

### JWT für Web
Verworfen.
- Sessions im Browser via JWT in localStorage / sessionStorage = XSS-Risiko (Token klaubar).
- HttpOnly-Cookie mit JWT: dann ist es im Endeffekt eine Session.
- JWT-Revocation ist mühsam (kein zentraler Store, ggf. JTI-Blacklist).
- Server-Session mit Spring Session ist einfacher und etablierter.

### OAuth2 für Web-Browser
Erwogen für eine konsistente Lösung, verworfen.
- Browser-Login mit Authorization Server zusätzliches Round-Trip.
- Komplexer als E-Mail+Passwort+Cookie.
- Mehr Komplexität ohne Mehrwert für interne Web-Klienten.

OAuth2 könnte später für SSO mit externen Providern (Google, Microsoft) ergänzt werden — Phase 2+ als Add-on.

### JWT für native Mobile statt OAuth2
Verworfen.
- OAuth2 + PKCE ist Industrie-Standard für native Mobile-Apps.
- Refresh-Token-Rotation eingebaut, sichere Storage in Android Keystore.
- Spring Authorization Server unterstützt es nativ.

### Session-Cookies auch für Mobile
Verworfen.
- Native Mobile App handhabt Cookies anders als Browser.
- OAuth2-Standard ist für Mobile etabliert.

## Konsequenzen

### Positiv
- **Web ist einfach**: Spring Security + Spring Session, Standard-Pattern.
- **Cookies HttpOnly+SameSite=Lax** sind XSS- und CSRF-resistent.
- **Geräte und Mobile sind getrennt** — jeder Klient mit passendem Auth-Mechanismus.
- **Spring Authorization Server** ist Open-Source und Spring-Boot-integriert.
- **Refresh-Token-Rotation** schützt vor Token-Theft.

### Negativ
- **Drei Auth-Mechanismen** im Code (Sessions, Geräte-Tokens, OAuth2). Mehr Komplexität als One-Size-fits-all.
- **Mobile-Phase-2-Vorbereitung** muss schon im MVP angedacht werden (z.B. OAuth-Endpoints können später aktiviert werden ohne Code-Refactoring).

### Neutral
- Spring Session unterstützt Multiple Backing Stores, was Profile-Switching erlaubt.

## Querverweise

- [`../architecture/authentication.md`](../architecture/authentication.md)
- [ADR-012](012-device-tokens.md), [ADR-013](013-staff-pin.md)
