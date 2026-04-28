# DSGVO-Umsetzung

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28
> **Hinweis:** Dieses Dokument ist eine technische Spezifikation, KEINE Rechtsberatung. Vor Echtkunden-Go-Live MUSS eine externe Datenschutz-Beratung (Datenschutzbeauftragter) konsultiert werden.

## Rollen-Modell (DSGVO Art. 4)

```
Endkunde (Betroffener)
    ↑ erteilt Einwilligung zur Datenverarbeitung
    ↓
Restaurant (Verantwortlicher, Art. 4 Nr. 7)
    ↑ AVV (Art. 28)
    ↓
Plattform-Owner (Auftragsverarbeiter, Art. 4 Nr. 8)
```

- **Endkunde:** natürliche Person, die online bestellt.
- **Restaurant (Tenant):** entscheidet über Zwecke und Mittel der Datenverarbeitung — z.B. „ich verkaufe Pizza an meine Endkunden und brauche dafür Bestelldaten".
- **Plattform-Owner:** verarbeitet im Auftrag des Restaurants die Endkundendaten — stellt nur die technische Plattform.

## Personenbezogene Daten — Inventar

### Endkunden
- E-Mail-Adresse
- Vor- und Nachname
- Telefonnummer
- Lieferadresse(n)
- Bestellhistorie (mit Zeitpunkt, Inhalt, Beträgen)
- Tisch-Bestellungen mit IP-Fingerprint
- Login-Historie (Zeitpunkt, IP)
- IP-Adressen bei Online-Bestellungen
- Magic-Link- und Passwort-Reset-Tokens

### Restaurant-Mitarbeiter
- E-Mail-Adresse
- Vor- und Nachname
- Telefonnummer (optional)
- Login-Historie
- Aktivitäten im System (Audit-Log)
- PIN-Hashes

### Restaurant-Inhaber zusätzlich
- Vollständige Kontaktdaten
- Postanschrift (legal_name, address)
- USt-ID, Handelsregister (sind aber juristische Daten, nicht primär personenbezogen)

### Plattform-Admins
- E-Mail-Adresse
- Vor- und Nachname
- Login-Historie
- Audit-Log-Aktionen

## Rechtsgrundlagen (Art. 6 DSGVO)

| Verarbeitung | Rechtsgrundlage |
|---|---|
| Bestellabwicklung Endkunde | Art. 6 (1) b — Vertragserfüllung |
| Bestellhistorie | Art. 6 (1) b + Art. 6 (1) c (steuerliche Aufbewahrung) |
| Newsletter (Phase 2) | Art. 6 (1) a — Einwilligung |
| Bonitätsprüfung (NICHT vorhanden) | n/a |
| Login-Logs / IP-Adressen | Art. 6 (1) f — berechtigtes Interesse (Sicherheit) |
| Mitarbeiter-Daten | Art. 6 (1) b + § 26 BDSG (Beschäftigtendatenschutz) |
| Plattform-Admin-Daten | Art. 6 (1) b — Vertragsdurchführung mit Plattform-Owner |

## Maßnahmen pro Betroffenenrecht (Art. 12-22)

### Auskunftsrecht (Art. 15)

**Endkunde:**
- Im Endkunden-Konto-UI: „Meine Daten herunterladen" (`GET /api/v1/customer/me/data-export`).
- Antwort: ZIP mit JSON + CSV der gespeicherten Daten:
  - Konto-Stammdaten
  - Adressen
  - Bestellhistorie (alle Bestellungen mit Items, Zeiten, Beträgen)
  - Login-Historie der letzten 90 Tage (aus Audit-Log)
- Bereitstellung innerhalb 30 Tagen.

**Restaurant-Mitarbeiter:** auf Antrag manuell, durch Restaurant-Inhaber mit Tools-Unterstützung.

### Recht auf Berichtigung (Art. 16)

- Endkunde kann eigenes Konto bearbeiten (Name, Adressen, Telefon).
- E-Mail-Änderung mit Bestätigung der neuen Adresse.

### Recht auf Löschung / „Vergessenwerden" (Art. 17)

**Wichtige Einschränkung:** Wegen GoBD (10 Jahre Aufbewahrungspflicht) darf eine **vollständige Löschung von Bestellhistorie nicht erfolgen**. Stattdessen: **Anonymisierung**.

**Anonymisierungs-Workflow:**

1. Endkunde-Anfrage (im Konto-UI „Konto löschen") oder per Mail.
2. Anwendung führt aus:
   - `customer.email = 'deleted-' || customer.id || '@anonymous.local'`
   - `customer.full_name = 'Gelöscht'`
   - `customer.phone = NULL`
   - `customer.password_hash = NULL`
   - `customer.status = 'ANONYMIZED'`
   - `customer.anonymized_at = now()`
   - Alle `customer_addresses` werden gelöscht.
   - Bestellhistorie bleibt erhalten, referenziert aber den anonymisierten Customer.
   - In `Order.customer_snapshot` (bei Gast-Bestellungen) werden die Daten anonymisiert.
3. Audit-Log-Eintrag `CUSTOMER_ANONYMIZED`.
4. Sessions des Customers werden invalidiert.
5. Bestätigungs-Mail an alte Adresse (vor Anonymisierung gemerkt) wird optional unterdrückt — User hat ja gerade gelöscht.

**Verbleibend in Audit-Log:** Aktionen des Customers behalten ihre Customer-ID, was rückverfolgbar wäre, falls man wüsste, welche Customer-ID welcher Person gehörte. Da der Customer-Datensatz aber anonymisiert ist, ist dies in der Praxis pseudo-anonymisiert.

### Recht auf Einschränkung der Verarbeitung (Art. 18)

- Nicht häufig anfallend in dieser Domäne.
- Auf Anfrage manuell durch Plattform-Owner umzusetzen — Account auf `LOCKED` setzen, weitere Verarbeitung pausieren.

### Recht auf Datenübertragbarkeit (Art. 20)

- Identisch mit Auskunftsrecht: ZIP-Export mit JSON+CSV.
- Daten in maschinenlesbarem, strukturiertem Format.

### Widerspruchsrecht (Art. 21)

- Bei Newsletter (Phase 2): Opt-Out-Link in jeder E-Mail.
- Bei Tracking/Profiling: nicht im Scope (kein Tracking).

### Automatisierte Entscheidungen (Art. 22)

- Automatische Annahme/Ablehnung von Bestellungen ist eine automatisierte Entscheidung.
- Allerdings ohne erhebliche Auswirkungen auf Endkunden (es ist nur eine Bestellung, kein Kreditentscheid).
- Trotzdem: Datenschutzerklärung erwähnt automatische Annahme-Logik transparent.

## Datenschutz-by-Design / by-Default (Art. 25)

| Prinzip | Umsetzung |
|---|---|
| Datenminimierung | Endkunden-Konto: Pflichtfelder minimal (Name, E-Mail, Telefon). Adresse erst bei Lieferbestellung. |
| Pseudonymisierung | Anonymisierte Customer behalten Pseudo-ID. |
| Mandantentrennung | RLS, Defense-in-Depth. Endkundendaten von Tenant A sind für Tenant B unsichtbar. |
| Verschlüsselung at-rest | LUKS, GPG für Backups. |
| Verschlüsselung in-transit | TLS 1.2+. |
| Minimale Speicherdauer | Sessions 8 h, Magic-Links 30 min, Allgemeine Audit-Logs 90 Tage. |

## Auftragsverarbeitungsvertrag (AVV)

### Pflicht (Art. 28)

Tenant und Plattform-Owner MÜSSEN vor Echtkunden-Datenverarbeitung einen AVV abschließen.

### Implementierung

**Onboarding-Flow:**
1. Plattform-Owner stellt AVV-Template als PDF bereit.
2. Tenant lädt PDF herunter, druckt, unterzeichnet.
3. Tenant lädt unterzeichnetes PDF im Backoffice hoch.
4. Plattform-Owner gegenzeichnet (manuell, off-system).
5. Beide Parteien haben Kopie.

**Tenant-Status:** Im Backoffice ist „AVV unterzeichnet: ja/nein". Ohne unterzeichneten AVV: Restaurant kann Online-Bestellfunktion nicht aktivieren (Pflicht-Sperrung Pre-Echtkunden).

### AVV-Inhalt (Stichpunkte)

- Gegenstand und Dauer der Verarbeitung
- Art und Zweck (Restaurantverwaltung, Online-Bestellung)
- Art der personenbezogenen Daten (siehe Inventar)
- Kategorien Betroffener
- Pflichten und Rechte des Verantwortlichen
- TOMs (Technische und Organisatorische Maßnahmen) als Anlage
- Subunternehmer-Liste (z.B. Stripe, fiskaly, Brevo, Hosting)
- Genehmigung von Subunternehmer-Wechseln

### TOMs (Anlage zum AVV)

Übersicht der technischen und organisatorischen Maßnahmen:
- Zugangskontrolle (LUKS, SSH-Härtung, fail2ban)
- Zugriffskontrolle (Rollen, Permissions, RLS)
- Weitergabekontrolle (TLS, GPG)
- Eingabekontrolle (Audit-Log)
- Verfügbarkeitskontrolle (Backups, RAID)
- Trennungskontrolle (Mandantentrennung via RLS)

Diese TOMs werden mit jedem AVV als Anlage geliefert. Aktualisierte Versionen werden Tenants zur Kenntnis gebracht.

## Subunternehmer

Im MVP keine echten externen Dienstleister (alle Adapter sind Fakes). Vor Echtkunden-Go-Live werden eingebunden:

| Anbieter | Verarbeitete Daten | Sitz | Vertrag |
|---|---|---|---|
| Stripe / Mollie / Adyen | Zahlungsdaten | EU oder USA mit SCC | DSGVO-konform per Provider-DPA |
| fiskaly / Epson | Bestelldaten (für TSE) | EU | DPA mit Provider |
| Brevo / Mailjet | E-Mail-Adressen, E-Mail-Inhalte | EU | DPA |
| Hosting (Eigener Server) | alle | DE | n/a (Owner ist Hoster) |

**Bei jeder Subunternehmer-Änderung:** Tenants müssen informiert werden, ggf. Widerspruchs-Möglichkeit.

## Datenschutzerklärung

Pro Tenant individuell, generiert aus:
- Pflicht-Stammdaten des Tenants (Inhaber-Adresse, Kontakt).
- Plattform-Standard-Bausteine (Nennung der Subunternehmer).
- Tenant-spezifischen Anpassungen (z.B. eigene Zusatz-Verarbeitungen — Newsletter etc.).

Generator-Komponente in Phase 2. Im MVP: Tenant pflegt manuell.

## Verzeichnis von Verarbeitungstätigkeiten (Art. 30)

Pflicht für Plattform-Owner. **Verantwortung des Plattform-Owners**, NICHT der einzelnen Tenants (die Tenants haben ein eigenes Verzeichnis für ihre Kundendaten).

Inhalt:
- Name + Kontakt Plattform-Owner
- Zweck der Verarbeitung
- Kategorien Betroffener
- Kategorien personenbezogener Daten
- Empfänger (Subunternehmer)
- Drittlandtransfers
- Löschfristen
- TOMs

Wird als statisches Dokument außerhalb der Anwendung gepflegt.

## Datenschutz-Folgenabschätzung (DSFA, Art. 35)

Voraussichtlich NICHT erforderlich, da:
- Keine systematische Profilerstellung.
- Keine besonderen Kategorien personenbezogener Daten (keine Gesundheits-, Religions-Daten).
- Kein automatisierter Bonitäts-Score.

Vor Echtkunden-Go-Live in Datenschutz-Beratung formal verifizieren.

## Daten-Pannen-Meldung (Art. 33-34)

Bei Datenleck:
1. **Erkennung** durch Monitoring oder externen Hinweis.
2. **Bewertung** durch Plattform-Owner: Schwere, Anzahl Betroffener, Risiko.
3. **Meldung an Aufsichtsbehörde** binnen 72 h, falls Risiko für Betroffene besteht.
4. **Information an Betroffene** ggf. (bei hohem Risiko).
5. **Postmortem** intern.

Der Plattform-Owner hat eine Meldung-Vorlage vorbereitet (außerhalb dieser Doku).

## Cookies und Tracking

- **Im MVP:** keine Tracking-Cookies, kein Analytics, kein Werbe-Tracking.
- **Session-Cookies:** technisch notwendig (DSGVO Art. 6 (1) f), kein Cookie-Banner erforderlich.
- **Phase 2:** falls Analytics (z.B. Plausible Analytics — datenschutzfreundlich) eingeführt wird → Cookie-Banner / Consent-Mechanismus.

## Hinweise für die UI

- Bei Endkunden-Registrierung: Link zur Datenschutzerklärung des Restaurants, Link zur Plattform-eigenen Datenschutzerklärung.
- Bei Bestellabschluss: erneuter Hinweis und Pflicht-Akzeptanz der AGB.
- „Konto löschen" als prominenter Punkt im Endkunden-Konto.
- „Daten herunterladen" als prominenter Punkt im Endkunden-Konto.

## Aufbewahrungsfristen-Übersicht

| Daten-Typ | Aufbewahrung | Begründung |
|---|---|---|
| Aktive Endkunden-Konten | unbegrenzt | berechtigtes Interesse, bis Löschanfrage |
| Bestellungen | 10 Jahre | GoBD |
| Audit-Log allgemein | 90 Tage | Sicherheits-Diagnose |
| Audit-Log finanziell | 10 Jahre | GoBD |
| Login-Logs | 90 Tage (Teil des allgemeinen Audit-Logs) | Sicherheits-Diagnose |
| Magic-Links | bis zur Verwendung oder 30 min Ablauf | Funktional |
| Sessions | bis Logout oder 8 h Inaktivität | Funktional |
| Backups | 30 Tage täglich, 12 Monate monatlich | Disaster Recovery |
| AVV-Verträge | 3 Jahre nach Vertragsende (oder länger nach lokalem Recht) | Vertragsdokumentation |
