# ADR-013: Mitarbeiter-PIN am Gerät

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Geräte im Restaurant sind authentifiziert via Geräte-Token, aber nicht an eine Person gebunden. Trotzdem müssen Aktionen am Gerät einer Person zugeordnet werden für:

- Audit-Log (wer hat Bestellung X angelegt).
- Zusätzliche Permissions (z.B. Manager-Override).
- Gerichtliche Nachvollziehbarkeit bei Stornos, Steuer-Overrides.

Eingabe von E-Mail+Passwort am Tablet ist im Tagesbetrieb unzumutbar (zu langsam, fehleranfällig).

## Entscheidung

**4- bis 6-stellige PIN pro User**, separat vom Login-Passwort.

- Speicherung: `users.pin_hash` mit Argon2id-Hash.
- Eingabe am Gerät: Wenn `device.staff_pin_policy = REQUIRED` oder OPTIONAL.
- Validierung: PIN wird gehasht und gegen alle aktiven User des Tenants verglichen, deren Standort-Scope den Gerät-Standort enthält.
- Bei Match: `staffUserId` ist im Gerät-State gesetzt, Aktionen werden auf den User attribuiert.
- Auto-Lock: Gerät kehrt zur PIN-Eingabe nach 5 min Inaktivität.
- Brute-Force: max. 5 PIN-Fehlversuche pro Gerät, dann 5 min Lockout.

## Manager-PIN-Mechanismus

Spezialfall: Mitarbeiter (Staff) löst Aktion aus, die seine eigene Permission nicht erlaubt (z.B. Storno nach IN_PROGRESS).

**Ablauf:**
1. Backend antwortet 403 mit Code `PERMISSION_REQUIRED`, gibt benötigte Permission an.
2. Frontend zeigt Manager-PIN-Modal.
3. Manager (Owner / Manager / LocationLead) gibt PIN ein → `POST /api/v1/auth/manager-confirm`.
4. Server prüft: PIN gehört zu einem User mit der erforderlichen Permission im Standort-Scope.
5. Server gibt Confirm-Token (60 s gültig, einmal verbrauchbar, an Aktion gebunden).
6. Frontend wiederholt ursprünglichen Request mit Confirm-Token.
7. Audit: Aktion wird auf Manager attribuiert mit `initiatedByUserId=<staff>` als Kontext.

## Alternativen

### Voll-Login per E-Mail+Passwort am Gerät
Verworfen. Operativ unzumutbar.

### Magnetkarte / RFID
Verworfen. Hardware-Investitionen, zusätzliche Komplexität, kein klarer Mehrwert in dieser Domäne.

### Biometrie
Verworfen. Nur auf manchen Geräten verfügbar, unzuverlässig in Küchen-Umgebung mit Mehl/Fett an Fingern.

### Personal-Login per QR-Code mit User-spezifischem Token
Erwogen, verworfen. Token-Diebstahl-Risiko, PIN ist robust und Standard in Gastro.

### Gemeinsame Schicht-PIN
Verworfen. Verlust der individuellen Attribuierung — Audit-Spuren wertlos.

## Konsequenzen

### Positiv
- **Schnelle Eingabe** im Tagesbetrieb (4-6 Ziffern, 2-3 Sekunden).
- **Individuelle Attribuierung** — Audit-Log weiß, wer was gemacht hat.
- **Manager-PIN-Mechanismus** löst Override-Anforderungen ohne Sitzungswechsel.
- Standardisierte UX in Gastro-Branche, Personal kennt das.

### Negativ
- **PINs sind kurz** → niedrige Entropie. Daher PIN ausschließlich am Gerät einsetzbar (das ist bereits durch Token authentifiziert), niemals als Login-Bypass für Web.
- **Brute-Force** muss aktiv geblockt werden (5 Fehlversuche → Lockout).
- **PIN-Vergessen** → Reset über Manager / Owner.
- **PIN-Kollision** möglich (zwei User mit derselben PIN). Argon2-Hash macht das nicht trivial verifizierbar — System sucht den ersten Match. Bei Risiko: Pflicht zur eindeutigen PIN pro User-Pool.

### Neutral
- PIN-Eingabe am Bildschirm hat Privacy-Issues (Schulter-Surfing) — typisch für Gastro, akzeptabel.

## Querverweise

- [`../architecture/authentication.md`](../architecture/authentication.md)
- [`../domain/permissions.md`](../domain/permissions.md)
- [ADR-012](012-device-tokens.md)
