# Use-Case-Katalog

> **Status:** Accepted (MVP-Scope)
> **Letzte Aktualisierung:** 2026-04-28

Dieser Katalog ist die verbindliche Spezifikation aller Anwendungsfälle im MVP. Für jeden Use Case sind **Akteur**, **Vorbedingungen**, **Hauptablauf**, **Alternativen**, **Nachbedingungen** und **Permissions** dokumentiert.

## Konventionen

- **UC-Code:** Eindeutige ID, Format `UC-X##` (X = Bereichscode, ## = laufende Nummer).
- **Akteur:** Wer triggert den Use Case.
- **Permissions:** Welche Permission(s) sind erforderlich (siehe [`permissions.md`](permissions.md)).
- **Trigger:** Was startet den Use Case (UI-Aktion, eingehender Request, geplanter Job).
- **Hauptablauf:** Schritt-für-Schritt Standardfall.
- **Alternativen / Fehlerpfade:** Was passiert bei Abweichung.
- **Nachbedingungen:** Welche Zustandsänderungen sind garantiert.
- **Audit-Log-Eintrag:** Welche Aktion wird auditiert.

## Bereichs-Codes

- **A** = Plattform-Admin
- **R** = Restaurant-Verwaltung (Onboarding, Konfiguration)
- **B** = Bestellaufnahme am Tablet
- **K** = Küche (Kitchen Display)
- **S** = Self-Order / öffentliche Online-Bestellseite
- **O** = Online-Bestellungs-Annahme (im Standort)
- **E** = Endkunden-Konto
- **X** = Querschnitt (Audit, Reset, Auth)

---

## Bereich A: Plattform-Admin

### UC-A01: Restaurant-Tenant anlegen

**Akteur:** Plattform-Admin (mit `PLATFORM_TENANT_CREATE`)
**Trigger:** Admin klickt „Neues Restaurant anlegen" im Plattform-UI.
**Vorbedingungen:** Admin ist eingeloggt.

**Hauptablauf:**
1. Admin öffnet Formular „Neuer Tenant".
2. Admin gibt ein: Name, Legal Name, Inhaber-Kontakt (Name, E-Mail, Telefon, Postanschrift), USt-ID (optional), Handelsregister (optional), Lizenzpaket-Auswahl, Ablaufzeit für Einrichtungs-Link (Default: 7 Tage).
3. Admin klickt „Anlegen".
4. System validiert: E-Mail des Inhabers ist nicht bereits Inhaber eines anderen Tenants.
5. System erzeugt:
   - `Tenant` mit `status=ACTIVE` aber `pendingSetup=true`.
   - `License`-Datensatz.
   - Einmal-Token (256-Bit) mit Ablaufdatum.
   - Default-Stamm-Speisekarte (leer, nur Kategorien-Vorlage).
6. System zeigt Admin den Einrichtungs-Link (`https://platform/setup/<token>`) zum Kopieren und Versenden.
7. Optional: System schickt Welcome-Mail mit Link über `MailPort` (im Demo: virtuelles Postfach).

**Alternativen:**
- **A1:** E-Mail bereits in anderem Tenant Inhaber → Fehlermeldung, Vorgang abbrechen.
- **A2:** Admin schließt Browser, bevor Link kopiert ist → System bietet auf Tenant-Detailseite „Einrichtungs-Link erneut anzeigen", solange Token nicht eingelöst.

**Nachbedingungen:**
- Tenant existiert mit `pendingSetup=true`.
- Einrichtungs-Token ist versandt/anzeigt.
- Audit-Log: `TENANT_CREATED` mit Tenant-Daten und Admin-ID.

**Sonderfälle:** Token-Ablauf siehe UC-R01.

---

### UC-A02: Tenant sperren / reaktivieren / soft-löschen

**Akteur:** Plattform-Admin (`PLATFORM_TENANT_SUSPEND` für Sperren/Reaktivieren, `PLATFORM_TENANT_DELETE` für Löschen)
**Trigger:** Admin klickt im Tenant-Detail auf „Sperren" / „Reaktivieren" / „Löschen".

**Hauptablauf (Sperren):**
1. Admin öffnet Tenant-Detail.
2. Admin klickt „Sperren", gibt Begründung ein.
3. System fragt: „Wirklich sperren? Alle Logins werden sofort gesperrt."
4. Admin bestätigt.
5. System setzt `Tenant.status = SUSPENDED`.
6. System invalidiert alle aktiven Sessions des Tenants.
7. System sendet Mail an Tenant-Inhaber (Stub im Demo).

**Hauptablauf (Löschen):**
1. Wie oben, aber zusätzlich Captcha/Tippbestätigung des Restaurant-Namens.
2. System setzt `Tenant.status = DELETED`, `deletedAt = now()`.
3. Daten bleiben erhalten (Soft-Delete) für Aufbewahrungsfrist (10 Jahre für finanzielle Daten).
4. Login wird unmöglich.

**Permissions:** `PLATFORM_TENANT_SUSPEND` bzw. `PLATFORM_TENANT_DELETE`
**Audit-Log:** `TENANT_SUSPENDED` / `TENANT_REACTIVATED` / `TENANT_DELETED` mit Begründung.

---

### UC-A03: Lizenzpaket zuweisen / ändern

**Akteur:** Plattform-Admin (`PLATFORM_TENANT_LICENSE_EDIT`)
**Hauptablauf:**
1. Admin öffnet Tenant-Detail → Lizenz-Tab.
2. Admin ändert Werte (Anzahl inkl. Standorte, Anzahl inkl. Geräte, Aufpreise).
3. System validiert: aktuelle Standort-/Geräteanzahl darf nicht über neuer Limite liegen, sonst Hinweis und Bestätigung erforderlich („6 Standorte aktuell, neue Limite 5 — Tenant überlizenziert, OK?").
4. Speichern.

**Audit-Log:** `LICENSE_CHANGED` mit Diff.

---

### UC-A04: Nutzungsstatistiken einsehen

**Akteur:** Plattform-Admin (`PLATFORM_USAGE_VIEW`)
**Hauptablauf:**
1. Admin öffnet „Plattform-Statistiken".
2. System zeigt: Anzahl Tenants total/aktiv, Bestellungen/Tag/Woche/Monat, größte Tenants nach Volumen, Geräteauslastung.
3. Drill-Down pro Tenant: Bestellungen, Logins, aktive Geräte.

**Permissions:** `PLATFORM_USAGE_VIEW`

---

### UC-A05: Impersonation eines Tenant-Inhabers

**Akteur:** Plattform-Admin (`PLATFORM_TENANT_IMPERSONATE`)
**Trigger:** Admin klickt auf Tenant-Detail „Als Inhaber einloggen (Support)".

**Hauptablauf:**
1. Admin gibt Pflicht-Begründung ein.
2. System fragt Bestätigung: „Du wirst als <Owner-Name> eingeloggt. Alle Aktionen werden auf dich attribuiert."
3. Admin bestätigt.
4. System startet eine Impersonations-Session: Cookie enthält Marker `impersonated_by=<adminId>` und `original_user=<ownerId>`.
5. System leitet auf Tenant-Dashboard.
6. UI zeigt prominentes rotes Banner „Impersonation aktiv — Beenden".

**Beendigung:**
1. Admin klickt im Banner „Beenden".
2. System schließt Impersonations-Session, leitet zurück ins Admin-Dashboard.

**Audit-Log:**
- `IMPERSONATION_STARTED` mit Begründung, Admin-ID, Ziel-Tenant, Ziel-User.
- Während Impersonation: jede Aktion wird als `actor_type=ADMIN_IMPERSONATING`, `actor_id=adminId`, `as_user=ownerId` protokolliert.
- `IMPERSONATION_ENDED` bei Beendigung.

---

### UC-A06: Plattform-Default-Konfiguration ändern

**Akteur:** Plattform-Admin (`PLATFORM_CONFIG_EDIT`)
**Hauptablauf:**
1. Admin öffnet „Plattform-Konfiguration".
2. Admin ändert: Default-Steuersätze, Default-Annahme-Policy, AGB-Template, Impressums-Template-Hinweise, Datenschutzerklärungs-Template.
3. Speichern.

**Hinweis:** Bestehende Tenants übernehmen Änderungen NICHT automatisch — diese sind Defaults für neue Tenants.

---

### UC-A07: Weiteres Admin-Konto anlegen

**Akteur:** Plattform-Admin (`PLATFORM_ADMIN_MANAGE`, nur `SuperAdmin`)
**Hauptablauf:**
1. Admin öffnet „Plattform-Admins".
2. „Neuer Admin": E-Mail, Name, Plattform-Rolle (`SuperAdmin` / `SupportAdmin` / `BillingAdmin`).
3. System sendet Einrichtungs-Link (analog UC-A01).

---

## Bereich R: Restaurant-Verwaltung (Onboarding und Konfiguration)

### UC-R01: Erstes Inhaber-Konto via Einrichtungs-Link aufsetzen

**Akteur:** Restaurant-Inhaber (eingeladen, noch ohne Konto)
**Trigger:** Inhaber öffnet Einrichtungs-Link in Mail.

**Hauptablauf:**
1. Inhaber klickt Link `https://platform/setup/<token>`.
2. System validiert Token (existiert, nicht abgelaufen, nicht bereits eingelöst).
3. System zeigt Setup-Formular: Passwort + Passwort-Bestätigung, Telefon, Sprache (Default DE).
4. Inhaber gibt Daten ein, klickt „Konto anlegen".
5. System erstellt:
   - `User` mit `tenantId=<aus Token>`, `email=<aus Tenant.ownerContact>`, `passwordHash`, `status=ACTIVE`.
   - `RoleAssignment` mit `role=OWNER`, `scope=TENANT_WIDE`.
6. System markiert Token als eingelöst.
7. System loggt User automatisch ein und leitet auf „Erste Schritte" Wizard:
   - Standort 1 anlegen (UC-R02)
   - Optional: AGB / Impressum / Datenschutzerklärung pflegen
   - Optional: Erste Speisekarten-Items

**Alternativen:**
- **A1:** Token abgelaufen → Hinweisseite, Inhaber soll Plattform-Owner kontaktieren.
- **A2:** Token bereits eingelöst → Hinweis „Konto bereits eingerichtet, bitte einloggen".

**Audit-Log:** `OWNER_ACCOUNT_CREATED`.

---

### UC-R02: Standort anlegen / bearbeiten

**Akteur:** Inhaber oder Geschäftsführer (`LOCATION_CREATE` für anlegen, `LOCATION_EDIT` für bearbeiten)

**Hauptablauf (Anlegen):**
1. User öffnet „Standorte" → „Neuer Standort".
2. Eingabe: Name, Adresse, Kontakt, Öffnungszeiten (wöchentlich + Ausnahmen), Steuersitz (falls abweichend), Lieferzonen-PLZs (falls Lieferung angeboten).
3. Eingabe: Initial-Annahme-Policy (Default: AutoAnnahme).
4. Speichern.
5. System prüft Lizenzlimit: aktuelle Standorte+1 ≤ Limit + Aufpreis-Erlaubnis. Falls nicht: Bestätigung „Aufpreis fällt an" oder Abbruch.
6. System erzeugt `Location` und `LocationMenu` (leer, erbt nur von Stamm).

**Bearbeiten:** Analog, ohne Lizenzprüfung außer beim Reaktivieren eines INACTIVE Standorts.

**Audit-Log:** `LOCATION_CREATED` / `LOCATION_UPDATED` mit Diff.

---

### UC-R03: Tische anlegen / bearbeiten

**Akteur:** User mit `TABLE_MANAGE` (Inhaber, Geschäftsführer, Standortleiter für eigenen Standort).

**Hauptablauf:**
1. User öffnet Standort → Tab „Tische".
2. „Neuer Tisch": Nummer (Default: nächste freie), Tags (Multi-Select aus existierenden + freie Eingabe), Position im Tischplan (X/Y).
3. Speichern.
4. System generiert `qrToken`, druckbares QR-Code-PDF wird angeboten zum Download.

**Massen-Aktion:** „Tische 1–20 anlegen" → 20 Tische mit inkrementellen Nummern.

**Audit-Log:** `TABLE_CREATED` etc.

---

### UC-R04: Tischplan editieren

**Akteur:** User mit `TABLE_MANAGE`.
**Hauptablauf:**
1. Visuelle Ansicht aller Tische des Standorts.
2. Drag-and-Drop ändert `positionX`/`positionY`.
3. Auto-Speichern oder explizit „Speichern".

---

### UC-R05: Restaurant-Benutzer einladen

**Akteur:** User mit `USER_INVITE` (Inhaber, Geschäftsführer, Standortleiter — letzterer nur für Mitarbeiter-Rolle in eigenem Scope).

**Hauptablauf:**
1. User öffnet „Benutzer" → „Einladen".
2. Eingabe: E-Mail, Name, Rollen-Auswahl (gefiltert nach eigener Berechtigung), Scope-Auswahl (TENANT_WIDE oder Standort-Liste; gefiltert nach eigener Berechtigung).
3. Speichern.
4. System erstellt `User` mit `status=INVITED`, sendet Einladungs-Mail mit Token.

**Bei Annahme:**
1. Eingeladener öffnet Link, setzt Passwort.
2. `User.status = ACTIVE`.

**Audit-Log:** `USER_INVITED`, `USER_ACTIVATED`.

**Validierung:**
- Inhaber → kann beliebige Rolle vergeben.
- Geschäftsführer → keine Inhaber, keine Geschäftsführer.
- Standortleiter → nur Mitarbeiter, nur in eigenen Standort-Scope.

---

### UC-R06: Rolle / Scope eines Benutzers ändern

**Akteur:** User mit `USER_ROLE_EDIT` (Begrenzungen wie UC-R05).

**Hauptablauf:**
1. User öffnet Benutzer-Detail.
2. Ändert Rolle und/oder Scope.
3. Speichern.
4. System invalidiert die Sessions des betroffenen Users (nächster Request → re-Login).

**Validierung:**
- Letzten Inhaber degradieren ist verboten (siehe UC-R07 für Übertragung).
- Eigene Rolle herabstufen ist erlaubt, außer letzten Inhaber-Status.

**Audit-Log:** `USER_ROLE_CHANGED` mit Diff.

---

### UC-R07: Inhaberschaft übertragen

**Akteur:** Aktueller Inhaber (`TENANT_OWNERSHIP_TRANSFER`)

**Hauptablauf:**
1. Inhaber öffnet „Einstellungen" → „Inhaberschaft übertragen".
2. Auswahl des Ziel-Users (muss existierender User des Tenants sein).
3. System fragt aktuelles Passwort zur Bestätigung.
4. System fordert Pflicht-Bestätigung „Ich verstehe, dass ich nach Übertragung keine Inhaber-Rechte mehr habe, außer der neue Inhaber gibt sie mir zurück."
5. System setzt:
   - Ziel-User erhält `OWNER`-Rolle.
   - Bisheriger Inhaber: Rolle bleibt `OWNER` ODER wird optional auf gewählte Rolle herabgestuft (UI-Wahl).
6. Mail an beide Beteiligten.

**Audit-Log:** `OWNERSHIP_TRANSFERRED`, mit alter und neuer Inhaber-ID.

---

### UC-R08: Gerät anlegen

**Akteur:** User mit `DEVICE_CREATE`.

**Hauptablauf:**
1. „Geräte" → „Neues Gerät".
2. Eingabe: Name, Standort, verfügbare Modi (Multi-Select), Initial-Modus, Modus-Wechsel-Policy, Mitarbeiter-PIN-Policy.
3. System prüft Lizenzlimit; ggf. Aufpreis-Bestätigung.
4. System erzeugt `Device` mit Token.
5. System zeigt Token als langer String UND als QR-Code zum einmaligen Scannen.
6. Pop-up Hinweis: „Token wird nur jetzt angezeigt. Pairing am Gerät durchführen."

**Pairing am Gerät:**
1. Gerät startet im „Pairing"-Modus.
2. Nutzer scannt QR oder gibt Token manuell ein.
3. Gerät speichert Token sicher (Android Keystore bzw. localStorage in PWA).
4. Erste Heartbeat-Anfrage an Server bestätigt Pairing.

**Audit-Log:** `DEVICE_CREATED` (ohne Token-Hash im Log, nur Geräte-ID).

---

### UC-R09: Geräte-Token revoken / Gerät neu pairen

**Akteur:** User mit `DEVICE_REVOKE`.

**Hauptablauf:**
1. User öffnet Geräte-Detail.
2. Klick „Token widerrufen".
3. Bestätigung „Gerät verliert sofort Zugriff".
4. `Device.status = REVOKED`, alle weiteren API-Requests des alten Tokens → 401.
5. Optional: „Neuen Token erzeugen" → wie UC-R08 ab Schritt 4.

**Audit-Log:** `DEVICE_TOKEN_REVOKED`, `DEVICE_TOKEN_ROTATED`.

---

### UC-R10: Stamm-Speisekarte pflegen

**Akteur:** User mit `MENU_MASTER_EDIT` (Inhaber, Geschäftsführer).

**Hauptablauf (neuer Artikel):**
1. „Speisekarte" → „Neuer Artikel".
2. Eingabe: Kategorie, Name, Beschreibung, Bild (Upload, optional), Basispreis, Steuersätze (vor Ort / außer Haus), Allergene (Multi-Select aus LMIV-Liste), Zusatzstoffe.
3. Varianten hinzufügen: mind. 1, jeweils Name + priceDelta.
4. Optionen hinzufügen: 0..n, jeweils Name + priceDelta + optional Gruppe.
5. Verfügbarkeits-Zeitfenster (optional).
6. Speichern.
7. **Nachgelagert:** System propagiert Änderung in alle Standort-Speisekarten innerhalb von Sekunden (Domain-Event `MasterMenuChangedEvent`).

**Bearbeiten:** Diff wird in Audit-Log gespeichert. Bestehende Bestellungen sind durch Preis-Snapshot nicht betroffen.

**Audit-Log:** `MENU_ITEM_CREATED/UPDATED/DELETED` mit Diff.

---

### UC-R11: Standort-Speisekarte anpassen

**Akteur:** Inhaber, Geschäftsführer (`MENU_LOCATION_EDIT`) ODER Standortleiter (für eigenen Standort).

**Hauptablauf:**
1. „Standorte" → einzelnen Standort öffnen → „Speisekarte".
2. UI zeigt geerbte Stamm-Speisekarte mit „Override"-Optionen pro Item:
   - Aktivieren / Deaktivieren am Standort.
   - Preis überschreiben (Standard: erbt Stamm-Preis).
   - Pro Variante / Option Preis überschreiben.
3. „Eigener Artikel" Button: legt Standort-spezifischen Artikel an (gleiche Felder wie Stamm-Item).
4. Speichern → sofortige Propagation zu Geräten.

**Audit-Log:** `LOCATION_MENU_OVERRIDE_CHANGED` mit Diff.

---

### UC-R12: Reporting einsehen

**Akteur:** User mit `REPORT_VIEW` (Inhaber, Geschäftsführer, Standortleiter).

**Hauptablauf:**
1. „Berichte" öffnen.
2. UI zeigt Top-KPIs für ausgewählten Zeitraum (Default: heute):
   - Tagesumsatz mit Vergleich Vortag / Vorwoche
   - Bestellzahlen nach Bestelltyp (Vor Ort / Abholung / Lieferung)
   - Top 10 Artikel
   - Durchschnittlicher Bonwert
   - Stornorate (Items REJECTED_BY_KITCHEN + nachträglich gecancelt) / Gesamt-Items
   - Tageszeit-Heatmap (Bestellungen pro Stunde)
   - Umsatz pro Standort (bei Multi-Standort)
3. Filter: Zeitraum, Standorte (gefiltert nach Scope), Bestelltyp, Mitarbeiter (falls PIN-Erfassung).
4. Drill-Down (`REPORT_DRILLDOWN`): Klick auf KPI → Liste der einzelnen Bestellungen.

**Performance-Anforderung:** Initial-Load < 3 s, Drill-Down < 5 s (siehe NFR).

**Audit-Log:** kein Eintrag für Read-only.

---

### UC-R13: Annahme-Policy konfigurieren

**Akteur:** User mit `ACCEPTANCE_POLICY_EDIT`.
Siehe [`acceptance-policies.md`](acceptance-policies.md) für Details des Policy-Modells.

---

## Bereich B: Bestellaufnahme am Tablet

### UC-B01: Mitarbeiter identifiziert sich am Tablet

**Akteur:** Restaurant-Mitarbeiter (Permission `ORDER_CREATE` mit Standort-Scope)
**Vorbedingungen:** Tablet ist gepairt, Modus ist `ORDER_TABLET`.

**Hauptablauf (PIN-Policy = REQUIRED):**
1. Tablet zeigt PIN-Eingabe.
2. Mitarbeiter gibt 4-6-stellige PIN ein.
3. System validiert PIN gegen `User.pin` für Users dieses Standorts.
4. Bei Erfolg: Session am Gerät startet mit `staffUserId` belegt.
5. Auto-Lock nach Inaktivität (Default: 5 min konfigurierbar).

**PIN-Policy = OPTIONAL:** Mitarbeiter kann PIN eingeben oder „Anonym fortfahren". Bei anonym: `staffUserId = NULL`.
**PIN-Policy = DISABLED:** Direkter Zugriff ohne PIN.

**Alternativen:**
- **A1:** PIN ungültig → Counter erhöhen, nach 5 Fehlversuchen Tablet-Lockout 5 min.

**Audit-Log:** `STAFF_IDENTIFIED_AT_DEVICE`.

---

### UC-B02: Tisch wählen

**Hauptablauf:**
1. Tablet zeigt Tischplan oder Liste.
2. Mitarbeiter tippt auf Tisch.
3. System prüft: Hat dieser Tisch eine offene `Order` (status NICHT in {READY, SERVED, PAID, CANCELLED})?
   - Ja: zeige bestehende Bestellung (öffnen → UC-B03 in Edit-Modus).
   - Nein: neue Bestellung anlegen (UC-B03 in New-Modus).

---

### UC-B03: Bestellung als OPEN_TAB öffnen oder erweitern

**Akteur:** Mitarbeiter mit `ORDER_CREATE`/`ORDER_MODIFY`.
**Hauptablauf:**
1. Bei neuer Bestellung: System legt `Order` an mit `status=DRAFT`, `orderType=ON_SITE`, `orderMode=OPEN_TAB`, `tableId=<gewählter Tisch>`, `staffUserId=<aus B01>`.
2. UI zeigt Speisekarten-Browser (Standort-Speisekarte des aktuellen Standorts).
3. Mitarbeiter wählt Item, dann Variante (Pflicht), dann Optionen (optional), dann Menge, dann „Hinzufügen".
4. UI zeigt aktuellen Bestellkorb.

**Spezialfall Tisch-Gruppe:** Wenn Tisch in aktiver Gruppe → UI zeigt Gruppen-Indikator, Bestellung wird der Gruppe zugeordnet.

---

### UC-B04: Items zur Bestellung hinzufügen

Sub-Use-Case von UC-B03. Detail in der UI:

1. Tab „Speisekarte" → Kategorie wählen → Item antippen.
2. Modal mit Varianten (Radio), Optionen (Checkboxes), Menge (+/-), Hinweistext (freie Texteingabe).
3. „Hinzufügen" → `OrderItem` mit `status=REQUESTED`, `priceSnapshot` berechnet aus aktueller Standort-Speisekarte.

---

### UC-B05: Bestellung an Küche senden

**Akteur:** Mitarbeiter (`ORDER_CREATE`/`ORDER_MODIFY`).
**Hauptablauf:**
1. Mitarbeiter klickt „An Küche".
2. System wechselt `Order.status` von `DRAFT` (oder bei Erweiterung: bleibt im aktuellen Status, neue Items wechseln in Lifecycle).
3. Items wechseln auf `ACCEPTED` (bei Erweiterung einer offenen Bestellung) oder `IN_PREPARATION` (bei initialer Bestellung — keine separate Acceptance nötig, Standardablauf).
4. **Domain-Event:** `OrderItemsSentToKitchenEvent` → SSE-Push an alle KDS-Geräte des Standorts.

**Spezialfall: Erweiterung einer bereits offenen Bestellung:**
- Neue Items haben Status `REQUESTED` → KDS sieht „Nachorder" mit Bestätigungs-Button.
- KDS bestätigt → `ACCEPTED` → `IN_PREPARATION`.
- KDS lehnt ab → `REJECTED_BY_KITCHEN` mit optionaler Begründung → Mitarbeiter sieht Push-Notification am Tablet.

**Audit-Log:** `ORDER_ITEMS_SENT_TO_KITCHEN`, `ORDER_ITEM_REJECTED_BY_KITCHEN`.

---

### UC-B06: Item nach Versand stornieren

**Akteur:** Mitarbeiter mit `ORDER_CANCEL_PRE_KITCHEN` (vor IN_PROGRESS) ODER Manager-PIN-Bestätigung für `ORDER_CANCEL_POST_KITCHEN`.

**Hauptablauf (vor IN_PROGRESS):**
1. Mitarbeiter wählt Item in offener Bestellung → „Stornieren".
2. Pflichtbegründung-Dropdown (Templates: „Falsche Auswahl", „Gast-Wunsch") + freier Text.
3. System setzt `OrderItem.status = CANCELLED`.

**Hauptablauf (nach IN_PROGRESS):**
1. Mitarbeiter wählt Item → „Stornieren".
2. System: „Diese Aktion erfordert Manager-PIN."
3. Manager gibt PIN ein.
4. System validiert: PIN gehört zu User mit `ORDER_CANCEL_POST_KITCHEN`-Permission für diesen Standort.
5. Storno wird auf Manager attribuiert, mit Hinweis auf initiierenden Mitarbeiter.

**Audit-Log:** `ORDER_ITEM_CANCELLED` mit Begründung, Akteur, ggf. Manager-Override-Hinweis.

---

### UC-B07: Item nach Versand ändern

**Akteur:** Mitarbeiter, ggf. Manager-PIN.

**Hauptablauf:**
1. Mitarbeiter wählt Item → „Ändern".
2. Bei `IN_PREPARATION` oder später: System fragt Manager-PIN (analog UC-B06).
3. Bei Variante/Option-Änderung: System erstellt impliziten Storno + neues `OrderItem` mit `status=REQUESTED` (durchläuft Küchen-Bestätigung wie B05).
4. Bei Mengenänderung: analog (Storno + neue Position mit korrekter Menge).

---

### UC-B08: Tisch-Wechsel oder Tisch-Zusammenlegung

**Akteur:** Mitarbeiter (`ORDER_MODIFY` + `TABLE_MANAGE` für Zusammenlegung).

**Hauptablauf (Tisch-Wechsel):**
1. Mitarbeiter öffnet offene Bestellung → „Tisch wechseln".
2. Auswahl neuer Tisch (muss frei sein oder bestätigte Zusammenlegung).
3. System ändert `Order.tableId`.

**Hauptablauf (Zusammenlegung):**
1. „Tische zusammenlegen" → Tisch-Auswahl (mind. 2).
2. System erzeugt `TableGroup`, betroffene Tische → `status=MERGED`.
3. Optional: bestehende Bestellungen der Tische werden auf einen Tisch konsolidiert (UI-Frage: „Bestellungen zusammenführen?").

**Audit-Log:** `TABLE_GROUP_CREATED`, `ORDER_TABLE_CHANGED`.

---

### UC-B09: Bestellung abrechnen

**Akteur:** Mitarbeiter mit `ORDER_CHARGE`.

**Hauptablauf:**
1. Mitarbeiter öffnet offene Bestellung → „Abrechnen".
2. System zeigt Übersicht: Items, Steuern (auto-ermittelt), Gesamtsumme.
3. Optional: Steuersatz-Override pro Item (Permission `ORDER_TAX_OVERRIDE`, mit Pflichtbegründung).
4. Mitarbeiter wählt Zahlart: Bar / Karte / (Online im MVP nur für SELF_ORDER vorgesehen).
5. Bei Bar:
   - Eingabe „Erhalten" → System berechnet Wechselgeld.
6. Bei Karte:
   - System triggert `ReceiptPrinterPort` für Beleg (Stub: PDF im virtuellen Postfach).
   - Im MVP: Kartenzahlung außerhalb Plattform (standalone Terminal), Mitarbeiter bestätigt „Zahlung erfolgreich am Terminal".
7. System erzeugt `Payment` mit `status=SUCCEEDED`.
8. **Domain-Event:** `OrderPaidEvent` → `TsePort.signTransaction()` (Stub im MVP) → Speicherung der Signatur in Order.
9. `Order.status = PAID`.
10. Bondruck via `ReceiptPrinterPort` (Stub: PDF).

**Audit-Log:** `ORDER_PAID` mit Beträgen, Zahlart, ggf. Steuer-Override.

---

### UC-B10: Modus des Tablets wechseln

**Akteur:** Anwesende Person am Gerät (mit Manager-PIN, falls Policy = MANAGER_PIN).
**Hauptablauf:**
1. Tablet-Menü → „Modus wechseln".
2. Auswahl Ziel-Modus aus `availableModes`.
3. Falls `modeSwitchPolicy = MANAGER_PIN`: PIN-Eingabe.
4. Tablet wechselt UI auf Ziel-Modus.

**Audit-Log:** `DEVICE_MODE_SWITCHED`.

---

## Bereich K: Küche / KDS

### UC-K01: Eingehende Bestellungen sehen

**Akteur:** Küchenpersonal (am KDS-Gerät).
**Hauptablauf:**
1. KDS-UI zeigt offene Bestellungen, gruppiert nach `IN_PREPARATION`.
2. SSE-Push bei neuer Bestellung → akustisches Signal + Highlight.
3. Bestellungen sortiert nach Eingangszeit oder geplanter Bereitstellung.

---

### UC-K02: Item-Status durchsetzen (Akzeptieren / Fertig / Ablehnen)

**Akteur:** Küchenpersonal.
**Hauptablauf:**
1. Item antippen.
2. Bei Status `REQUESTED` (Erweiterung): Buttons „Annehmen" / „Ablehnen mit Begründung".
3. Bei Status `IN_PREPARATION`: Button „Fertig".
4. State-Transition gemäß Lifecycle-Diagramm.

---

### UC-K03: Sammelaktion "Bestellung komplett fertig"

**Akteur:** Küchenpersonal.
**Hauptablauf:**
1. Klick auf Bestellung statt einzelnes Item → „Alle als fertig markieren".
2. System setzt alle Items auf `READY`.
3. **Domain-Event:** `OrderReadyEvent` → SSE-Push an Service-Tablets / Online-Statusseite des Endkunden.

---

## Bereich S: Self-Order / Öffentliche Online-Bestellseite

### UC-S01: Online-Bestellseite öffnen

**Akteur:** Endkunde (Anonym oder eingeloggt).
**Trigger:** Endkunde besucht `https://<restaurant>.platform.de` oder eigene Restaurant-Domain (per Tenant-Konfiguration).
**Hauptablauf:**
1. System rendert Standort-Auswahl (falls Restaurant mehrere Standorte hat) ODER direkt Standort-Speisekarte (bei einzelnem Standort oder per QR-Tisch-Link).
2. UI zeigt Bestelltyp-Auswahl: Vor Ort (nur über QR sichtbar), Abholung, Lieferung.

---

### UC-S02: Speisekarte browsen

**Hauptablauf:**
1. Kategorien-Tabs.
2. Items mit Bild, Beschreibung, Basispreis, Allergen-Indikator.
3. Items, die zur aktuellen Tageszeit nicht verfügbar sind, werden ausgegraut.
4. Klick auf Item → Detail-Modal mit Allergen-Liste, Variante-Auswahl, Optionen, Menge, „In den Warenkorb".

---

### UC-S03: Bestelltyp + Daten wählen

**Hauptablauf je nach Bestelltyp:**

**Vor Ort (über Tisch-QR):**
- Tisch ist aus QR-Token identifiziert.
- System prüft: Tisch ist in aktivem Standort? Standort akzeptiert Bestellungen?
- Endkunde optional: Eigenname (für Service).

**Abholung:**
- Endkunde wählt Abholzeit (Min-Vorlauf konfigurierbar pro Standort, Default 20 min).
- Endkunde gibt Kontakt (Telefon, optional E-Mail).

**Lieferung:**
- Endkunde gibt Adresse ein.
- System validiert PLZ gegen `Location.deliveryZipCodes`.
- Falls nicht: Fehlermeldung „Außerhalb Liefergebiet".
- Falls ja: Lieferzeit-Schätzung (Default-Schwelle aus Annahme-Policy + 30 min Lieferweg).

---

### UC-S04: Warenkorb und Checkout

**Hauptablauf:**
1. Endkunde sieht Warenkorb mit Items, Summen, Steuer-Aufschlüsselung.
2. „Zur Kasse" → Login-Wahl: Gast / Login / Magic-Link.
3. Gast: minimale Pflichtfelder (Name, E-Mail, Telefon, ggf. Adresse).
4. Eingeloggt: Daten vorausgefüllt.
5. **Pflicht-Anzeige LMIV:** Allergen-Hinweise nochmal vor Bestellabschluss.
6. **Pflicht-Anzeige:** AGB-Akzeptanz-Checkbox (bezieht sich auf Restaurant-AGB), Datenschutzerklärung-Verlinkung.
7. **Pflicht-Anzeige:** Widerrufs-Hinweis („Speisen für sofortigen Verzehr — kein Widerrufsrecht").
8. „Zahlungspflichtig bestellen" → Übergang zur Zahlung.

---

### UC-S05: Online-Zahlung über PaymentPort

**Hauptablauf:**
1. System ruft `PaymentPort.initiatePayment(orderId, amount, currency, returnUrl)` auf.
2. `FakePaymentAdapter` (Demo) gibt simulierten Session-Token zurück.
3. Endkunde wird auf Provider-Hosted-Page geleitet (im Demo: simulierte Seite mit „Zahlen" + „Abbrechen"-Buttons).
4. Endkunde klickt „Zahlen" → Erfolg simuliert; nach 1-3 s mit 95 % Wahrscheinlichkeit Erfolg, 5 % Fehler.
5. Webhook (im Demo simuliert) trifft Server: `PaymentPort.handleWebhook(...)` → Setzt `Payment.status=SUCCEEDED`.
6. **Domain-Event:** `OrderConfirmedEvent`.
7. Endkunde sieht Bestätigungsseite mit Bestellnummer und Status-Tracking-Link.

**Alternativen:**
- **A1:** Bei `PICKUP` oder `DELIVERY`: Endkunde wählt „Bezahlung bei Abholung/Lieferung" statt online → kein PaymentPort-Aufruf, `Payment.status=INITIATED` bleibt offen, wird beim Abrechnen am POS oder durch Fahrer geschlossen.

---

### UC-S06: Bestellung trifft im Standort ein und durchläuft Annahme-Policy

**Hauptablauf:**
1. Nach `OrderConfirmedEvent` → `AcceptancePolicyService.evaluate(order)`.
2. Je nach Modus:
   - `AutoAnnahme` → `Order.status = ACCEPTED`, sofortiger SSE-Push an KDS.
   - `ManuellAnnahme` mit Zeitfenster: `Order.status = AWAITING_ACCEPTANCE`, Timer läuft. UI im Standort zeigt Bestellung mit Countdown. Bei Ablauf ohne Aktion → `REJECTED` mit Grund „Zeitüberschreitung", automatischer Refund-Stub.
   - `ManuellAblehnung` mit Zeitfenster: `Order.status = ACCEPTED` provisorisch; UI zeigt Bestellung mit Ablehnen-Button + Countdown. Bei Ablehnung im Zeitfenster: Bestellung wird zurückgenommen, Refund-Stub. Bei Ablauf ohne Aktion: bleibt akzeptiert.
3. Detail siehe [`acceptance-policies.md`](acceptance-policies.md).

---

### UC-S07: Endkunde verfolgt Bestellstatus

**Hauptablauf:**
1. Endkunde öffnet Tracking-Link aus Bestätigungsmail oder im eingeloggten Bereich.
2. Statusseite zeigt aktuellen Lifecycle-Status (DRAFT/CONFIRMED/ACCEPTED/IN_PROGRESS/READY/...).
3. SSE-Verbindung für Live-Updates.

---

### UC-S08: Tisch-Bestellung und Kellner-Ruf vom Gast-Smartphone

**Akteur:** Endkunde am Gast-Smartphone.
**Trigger:** Endkunde scannt permanenten Tisch-QR.

**Hauptablauf (Tisch-Bestellung):**
1. URL `https://platform/t/<qrToken>` öffnet Standort-Speisekarte mit gesetztem `tableId` (aus Token).
2. Wie Self-Order, aber Bestellung als `ON_SITE` mit `tableId`.
3. Zahlung online (Self-Order-Modus) oder am Ende über Personal („zur Tischrechnung hinzufügen", erfordert Personal-Bestätigung).

**Hauptablauf (Kellner-Ruf):**
1. Endkunde tippt „Kellner rufen".
2. System prüft Rate-Limit: gibt es offenen `WaiterCall` für diesen Tisch? Letzter `RESOLVED < 3 min`? Falls ja: Hinweis „Bitte warten Sie".
3. Sonst: `WaiterCall` mit `status=OPEN`, SSE-Push an Personal-Geräte des Standorts.
4. Personal sieht Liste aktiver Rufe in Bestelltablet (separater Tab).
5. Personal markiert „Erledigt" → `status=RESOLVED`.
6. Personal kann Tisch blocken bei Spam → `status=BLOCKED` für X Stunden, weitere Rufe gehen direkt auf `IGNORED`.

**Anti-Abuse:**
- IP-Fingerprint pro Call gespeichert.
- Mehr als 3 Calls von derselben IP in 10 min an verschiedene Tische → Captcha.
- Personal-Block-Funktion siehe oben.

**Audit-Log:** `WAITER_CALL_TRIGGERED/RESOLVED/BLOCKED`.

---

## Bereich O: Online-Bestellungs-Annahme im Standort

### UC-O01: Annahme-Policy konfigurieren

Siehe [`acceptance-policies.md`](acceptance-policies.md).

### UC-O02: Eingehende Online-Bestellung manuell annehmen / ablehnen

**Akteur:** Personal mit `ONLINE_ORDER_ACCEPT_REJECT`.
**Hauptablauf:**
1. Personal-UI zeigt Liste eingehender Bestellungen mit Status `AWAITING_ACCEPTANCE` und Countdown.
2. „Annehmen" → `status=ACCEPTED` → `IN_PROGRESS` → KDS-Push.
3. „Ablehnen" → Pflichtbegründung → `status=REJECTED` → Refund-Stub (manuell zu klären) → Mail an Endkunde mit Begründung.

---

## Bereich E: Endkunden-Konto

### UC-E01: Endkunde registriert sich

**Hauptablauf:**
1. Endkunde gibt E-Mail, Passwort, Name, Telefon ein.
2. System schickt E-Mail-Verifizierungs-Link (Stub).
3. Bei Klick: `Customer.emailVerifiedAt` gesetzt.
4. Login möglich.

### UC-E02: Login / Passwort vergessen / Magic Link

**Hauptablauf (Magic Link):**
1. Endkunde gibt nur E-Mail ein.
2. System sendet Mail mit Einmal-Login-Link (30 min Gültigkeit).
3. Klick auf Link → automatische Session.

### UC-E03: Bestellhistorie und aktive Bestellungen

**Hauptablauf:**
1. Eingeloggter Endkunde öffnet „Meine Bestellungen".
2. Liste sortiert nach Datum, mit Status und Live-Update für aktive.

### UC-E04: Adressen verwalten

Standard-CRUD auf `CustomerAddress`.

---

## Bereich X: Querschnitts-Use-Cases

### UC-X01: Audit-Log durchsuchen

**Akteur:** Plattform-Admin (`PLATFORM_AUDIT_VIEW`) ODER Restaurant-User (`AUDIT_VIEW`, scope-gefiltert).
**Hauptablauf:**
1. Audit-Tab öffnen.
2. Filter: Zeitraum, Akteur-Typ, Aktion, Ziel-Entität.
3. Treffer-Liste mit Drill-Down.

### UC-X02: Eigenes Passwort ändern

Standard. Validierungs-Regeln siehe [`../architecture/authentication.md`](../architecture/authentication.md).

### UC-X03 [Demo-Profil only]: System zurücksetzen

**Akteur:** Beliebiger User im Demo-Profil.
**Trigger:** Aufruf `/demo/reset` oder Button im Demo-UI.
**Hauptablauf:**
1. Bestätigung „Alle Daten werden gelöscht und neu geladen."
2. System löscht alle Tenant-Daten, lädt Demo-Initial-State.

**Verfügbar nur bei aktiviertem Demo-Profil.**

### UC-X04 [Demo-Profil only]: Zeit-Sprung simulieren

**Akteur:** Demo-User.
**Hauptablauf:**
1. „Zeit-Sprung" → System erzeugt synthetische Bestellungen mit Datumsverschiebung, sodass Reporting der letzten 30 Tage gefüllt ist.

---

## Use-Case-zu-Permission-Matrix

Vereinfachte Übersicht. Vollständig in [`permissions.md`](permissions.md).

| UC | Erforderliche Permission | Scope |
|---|---|---|
| UC-A01 | PLATFORM_TENANT_CREATE | Plattform |
| UC-A02 | PLATFORM_TENANT_SUSPEND / DELETE | Plattform |
| UC-A05 | PLATFORM_TENANT_IMPERSONATE | Plattform |
| UC-R02 | LOCATION_CREATE / EDIT | Tenant |
| UC-R03 | TABLE_MANAGE | Standort |
| UC-R05 | USER_INVITE | Tenant oder Standort |
| UC-R10 | MENU_MASTER_EDIT | Tenant |
| UC-R11 | MENU_LOCATION_EDIT | Standort |
| UC-R12 | REPORT_VIEW | Tenant oder Standort |
| UC-B05 | ORDER_CREATE | Standort |
| UC-B06 (vor IN_PROGRESS) | ORDER_CANCEL_PRE_KITCHEN | Standort |
| UC-B06 (nach IN_PROGRESS) | ORDER_CANCEL_POST_KITCHEN (via Manager-PIN) | Standort |
| UC-B09 | ORDER_CHARGE | Standort |
| UC-B09 (Steuer-Override) | ORDER_TAX_OVERRIDE | Standort |
| UC-O02 | ONLINE_ORDER_ACCEPT_REJECT | Standort |
