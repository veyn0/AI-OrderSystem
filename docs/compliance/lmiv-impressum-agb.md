# LMIV, Impressum, AGB

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28
> **Hinweis:** Dieses Dokument ist eine technische Spezifikation, KEINE Rechtsberatung. Vor Echtkunden-Go-Live MUSS ein Anwalt die generierten Texte prüfen.

## LMIV (Lebensmittel-Informationsverordnung)

EU-Verordnung 1169/2011, in DE durch LMIDV ergänzt. Verlangt verpflichtende Allergen- und Zusatzstoff-Angaben für Lebensmittel.

### Anwendbarkeit

Bei **Online-Verkauf** (Fernabsatz) MUSS die Allergen-/Zusatzstoff-Information **vor Vertragsabschluss** zur Verfügung stehen. „Allergene auf Anfrage" ist im Fernabsatz NICHT zulässig.

Bei **Vor-Ort-Bestellung in der Gastronomie** ist die Information ebenfalls Pflicht, kann aber alternativ durch Aushang/Speisekarten-Hinweis oder mündliche Auskunft auf Nachfrage erfolgen.

### 14 Hauptallergene (LMIV Anhang II)

| Code | Allergen |
|---|---|
| A | Glutenhaltiges Getreide (Weizen, Roggen, Gerste, Hafer, Dinkel, Kamut) |
| B | Krebstiere |
| C | Eier |
| D | Fisch |
| E | Erdnüsse |
| F | Soja |
| G | Milch (inkl. Laktose) |
| H | Schalenfrüchte (Mandeln, Haselnüsse, Walnüsse, Cashews, Pekannüsse, Paranüsse, Pistazien, Macadamia) |
| I | Sellerie |
| J | Senf |
| K | Sesam |
| L | Schwefeldioxid und Sulfite |
| M | Lupinen |
| N | Weichtiere |

### Zusatzstoffe (Auswahl)

Müssen kenntlich gemacht werden, wenn enthalten:
- 1 — mit Farbstoff
- 2 — mit Konservierungsstoff
- 3 — mit Antioxidationsmittel
- 4 — mit Geschmacksverstärker
- 5 — geschwefelt
- 6 — geschwärzt
- 7 — gewachst
- 8 — mit Phosphat
- 9 — mit Süßungsmittel
- 10 — koffeinhaltig
- 11 — chininhaltig
- 12 — taurinhaltig

### Umsetzung in der Plattform

#### Daten-Modell

```sql
CREATE TABLE menu_item_allergens (
    menu_item_id UUID NOT NULL REFERENCES menu_items(id),
    allergen_code VARCHAR(10) NOT NULL,         -- 'A', 'B', ..., 'N'
    PRIMARY KEY (menu_item_id, allergen_code)
);

CREATE TABLE menu_item_additives (
    menu_item_id UUID NOT NULL REFERENCES menu_items(id),
    additive_code VARCHAR(10) NOT NULL,         -- '1', '2', ..., '12'
    PRIMARY KEY (menu_item_id, additive_code)
);
```

#### UI-Pflege

- Im MenuItem-Editor: zwei Multi-Select-Felder für Allergene und Zusatzstoffe.
- Tooltip mit Erklärung pro Code.
- Pflichtfeld-Status pro `MenuItem` konfigurierbar:
  - **Soft-Modus (MVP):** Felder optional, Hinweis „Allergene auf Anfrage" als Fallback bei leerem Feld auf der Online-Bestellseite.
  - **Strict-Modus (vor Echtkunden-Go-Live):** Allergene-Feld ist Pflicht, leer-lassen nicht möglich.

Die Strict-Modus-Aktivierung pro Tenant erfolgt über ein Flag `tenant_settings.lmiv_strict_mode`. Bei Aktivierung werden alle bestehenden MenuItems geprüft; fehlende Allergene werden einmalig auf eine spezielle Markierung „nicht angegeben" gesetzt, die der Tenant dann nachpflegen MUSS, bevor Online-Bestellung möglich ist.

#### Anzeige auf Bestellseite

- Pro MenuItem werden Allergen- und Zusatzstoff-Codes angezeigt.
- Klick / Hover öffnet Tooltip mit Vollnamen und Erklärung.
- Filter „Ohne Gluten", „Ohne Laktose" etc. möglich (Phase 2).

#### Anzeige auf Beleg

- Allergene werden auf dem Beleg pro Position aufgelistet (Codes).
- Eine Legende am Belegfuß erläutert die Codes.

#### Verantwortung

- Plattform stellt das Datenfeld bereit.
- Tenant verantwortet die korrekte Pflege.
- AVV/Lizenzvertrag enthält Hinweis: „Tenant trägt die Verantwortung für korrekte LMIV-Angaben."

### Rechtlicher Risiko-Hinweis

- Fehlende oder falsche Allergen-Angaben sind abmahnfähig durch Wettbewerber oder Verbraucherzentralen.
- Im Falle einer schweren Allergie-Reaktion kann der Tenant haftbar sein.
- Im MVP ist „Allergene auf Anfrage" als Fallback **rechtlich grenzwertig** und wird vor Echtkunden-Go-Live abgeschaltet.

## Impressumspflicht (TMG / DDG)

### Anwendbarkeit

Jeder Tenant, der seine Online-Bestellseite öffentlich anbietet, MUSS ein Impressum führen. Das Telemediengesetz (TMG) ist seit Mai 2024 ersetzt durch das Digitale-Dienste-Gesetz (DDG); Pflichten sind weitgehend identisch.

### Pflichtangaben

1. **Name und Anschrift** des Anbieters
   - Bei Einzelunternehmen: Vor- und Nachname + Postanschrift.
   - Bei juristischen Personen: Firmenname, vertretungsberechtigte Personen, Postanschrift.
2. **Kontakt:** E-Mail-Adresse, Telefonnummer.
3. **Rechtsform** (z.B. „Einzelunternehmen", „GmbH").
4. **Handelsregistereintrag** (falls vorhanden): Registergericht + HR-Nummer.
5. **Umsatzsteuer-Identifikationsnummer** (USt-ID, falls vorhanden).
6. **Aufsichtsbehörde** (falls erlaubnispflichtige Tätigkeit — bei Gastronomie ggf. das zuständige Gewerbeamt).
7. **Berufsbezeichnung** (bei reglementierten Berufen — bei Gastronomie eher nicht relevant).

### Verantwortlicher i.S.d. § 18 Abs. 2 MStV

Bei journalistisch-redaktionellen Inhalten (z.B. Blog-Bereich) ist zusätzlich eine verantwortliche Person zu benennen. Im Standard-Restaurant-Use-Case nicht relevant.

### Umsetzung in der Plattform

#### Tenant-Stammdaten

Bei Tenant-Onboarding sind diese Felder Pflicht:
- `legal_name` (Inhaber/Firma)
- `owner_full_name`
- `owner_email`, `owner_phone`
- `owner_address_*`
- `tax_id` (USt-ID, optional aber stark empfohlen)
- `registration_number` (Handelsregister, optional)

Diese Daten sind vor Live-Schaltung der Online-Bestellseite verpflichtend ausgefüllt.

#### Generiertes Impressum

```
Impressum

Anbieter:
{owner_full_name}
{legal_name}
{owner_address_street} {owner_address_house_number}
{owner_address_postal_code} {owner_address_city}
{owner_address_country_code}

Kontakt:
E-Mail: {owner_email}
Telefon: {owner_phone}

USt-IdNr.:
{tax_id}

Handelsregister:
{registration_number}
{registration_court}

Verantwortlich für den Inhalt:
{owner_full_name}, Anschrift wie oben.

Online-Streitbeilegung:
Die Europäische Kommission stellt eine Plattform zur Online-Streitbeilegung (OS) bereit:
https://ec.europa.eu/consumers/odr
```

URL: `https://<tenant-domain>/impressum` — automatisch verfügbar, sobald Tenant aktiv.

### Pflicht-Hinweis: Online-Streitbeilegung

EU-Verordnung 524/2013 verlangt einen Link zur OS-Plattform der EU. Plattform setzt diesen automatisch ein.

### Verbraucherschlichtung

§ 36 VSBG verlangt Information, ob der Anbieter zur Teilnahme an einem Schlichtungsverfahren bereit oder verpflichtet ist. Standard-Hinweis bei kleineren Restaurants:

> „Wir sind nicht verpflichtet und nicht bereit, an einem Streitbeilegungsverfahren vor einer Verbraucherschlichtungsstelle teilzunehmen."

Tenant kann diesen Text editieren, wenn er Schlichtung aktiv anbietet.

## AGB (Allgemeine Geschäftsbedingungen)

### Anwendbarkeit

Pflicht für jeden Online-Verkauf an Verbraucher (B2C). Müssen vor Vertragsabschluss zur Kenntnis genommen werden können.

### Pflichtbestandteile

1. **Vertragspartner und Vertragsschluss-Modalität**
   - Wer ist der Vertragspartner (Restaurant, nicht Plattform-Owner).
   - Wie kommt der Vertrag zustande (Bestellung = Angebot, Annahme durch Restaurant = Vertrag).
2. **Leistungsbeschreibung**
   - Was bietet das Restaurant an (Speisen, Lieferung, Abholung, etc.).
3. **Preise und Lieferkosten**
   - Endpreise inkl. MwSt.
   - Liefergebühren bei Lieferungen.
4. **Zahlung**
   - Akzeptierte Zahlungsmethoden.
   - Fälligkeit.
5. **Lieferung / Abholung**
   - Liefergebiet.
   - Lieferzeit.
   - Abholzeit.
6. **Widerrufsrecht**
   - Bei Speisen für sofortigen Verzehr (verderbliche Ware): **kein Widerrufsrecht** nach § 312g Abs. 2 Nr. 2 BGB.
   - Hinweis MUSS deutlich angegeben werden.
7. **Gewährleistung**
   - Bei Mängeln (z.B. falsche/fehlende Bestandteile): Nachbesserung oder Erstattung.
8. **Datenschutz-Hinweis**
   - Verweis auf separate Datenschutzerklärung.
9. **Streitbeilegung**
   - OS-Link.
   - Schlichtungsbereitschaft.

### Umsetzung in der Plattform

#### Generierungs-Modi

**Modus 1 (MVP, einfach):** Tenant lädt eigenes AGB-PDF hoch. URL: `/<tenant>/agb`.

**Modus 2 (Phase 2):** Plattform stellt Template-AGB mit Tenant-spezifischen Variablen bereit. Tenant kann personalisieren über UI.

#### Akzeptanz beim Bestellprozess

Vor Bestellabschluss zwingende Checkbox:
> [ ] Ich akzeptiere die [AGB] und habe die [Datenschutzerklärung] zur Kenntnis genommen.

Ohne Häkchen ist Bestellung nicht abschließbar.

Akzeptanz wird mit der Bestellung gespeichert (`order.terms_accepted_version` mit Hash der zum Zeitpunkt aktiven AGB-Version).

### Widerrufsrecht-Belehrung

Pflicht-Text auf Bestellseite vor Bestellabschluss:

> **Widerrufsbelehrung**
>
> Es besteht kein Widerrufsrecht für Lieferungen von Lebensmitteln für sofortigen Verzehr (§ 312g Abs. 2 Nr. 2 BGB). Speisen, die wir auf Ihre Bestellung hin zubereiten und ausliefern, fallen darunter.
>
> Bei mangelhafter Lieferung kontaktieren Sie uns bitte umgehend; wir werden den Mangel beheben oder den Betrag erstatten.

## PAngV (Preisangabenverordnung)

### Pflichten

- Endpreise inklusive MwSt. anzeigen.
- Bei Lieferung: Liefergebühren transparent.
- Grundpreise bei lose verkauften Waren (z.B. „X,XX € pro Liter") — bei Speisen meist nicht relevant.

### Umsetzung

- Im UI: alle Preise sind Brutto-Preise.
- Liefergebühren werden separat ausgewiesen, mit Hinweis „inkl. MwSt.".
- Bei Bestellung: aufgeschlüsselte Anzeige Zwischensumme + Liefergebühr + MwSt.-Anteil.

## Datenschutzerklärung

Detail siehe [`gdpr.md`](gdpr.md).

URL: `https://<tenant-domain>/datenschutz` — Pflicht.

Pflichtbestandteile siehe Art. 13 DSGVO. Im MVP wird Tenant verpflichtet, eigene Datenschutzerklärung hochzuladen. In Phase 2: Generator.

## Cookie-Hinweis

- Im MVP: keine Tracking-Cookies. Nur technisch notwendige Cookies (Session). Kein Cookie-Banner notwendig.
- In Phase 2: bei Einsatz von Analytics-Tools — Cookie-Banner mit Consent (TTDSG § 25).

## Wettbewerbsrechtliche Stolperfallen

- **„Beste Pizza der Stadt"** — werbliche Spitzenstellungs-Behauptung; rechtlich problematisch. Ist Tenant-Inhalt; Plattform überprüft das nicht.
- **„Frisch zubereitet"** — wenn nicht zutreffend, irreführend.
- **Sterne-Bewertungen** — bei Aufnahme: Pflicht zur Echtheitsprüfung (Phase 2 oder Verzicht).

## Compliance-Checkliste vor Live-Schaltung

Plattform-System prüft pro Tenant (vor Aktivierung der Online-Bestellung):

| Pflicht | Prüfung |
|---|---|
| Impressum-Pflichtfelder | Alle gefüllt? |
| Datenschutzerklärung-PDF | Hochgeladen? |
| AGB-PDF | Hochgeladen? |
| AVV mit Plattform | Unterzeichnet hochgeladen? |
| LMIV-Strict-Modus | Aktiv? Alle MenuItems mit Allergenen versehen? |
| Steuersätze | Pro MenuItem konfiguriert? |
| TSE-Adapter | Echter (kein Fake) konfiguriert? |
| Liefergebiet | Definiert? |

Erst wenn alle Checks grün sind, kann Tenant „Online-Bestellung aktivieren". Bis dahin ist die Online-Bestellseite nur als Vorschau für den Tenant selbst sichtbar.

## Verantwortungsteilung Plattform vs. Tenant

| Bereich | Plattform-Owner | Restaurant-Tenant |
|---|---|---|
| Bereitstellung Impressum-Generator | ✓ | – |
| Korrektheit der eingegebenen Stammdaten | – | ✓ |
| AGB-Template (Phase 2) | ✓ | – |
| Personalisierung der AGB | – | ✓ |
| LMIV-Datenfeld bereitstellen | ✓ | – |
| LMIV-Daten korrekt pflegen | – | ✓ |
| Datenschutzerklärung-Plattform-Anteil | ✓ | – |
| Datenschutzerklärung-Restaurant-Anteil | – | ✓ |
| TSE-Adapter | ✓ | – |
| Korrekte Steuersatz-Konfiguration | – | ✓ |
| Korrekte Speisekarten-Pflege | – | ✓ |
| Wettbewerbsrechtliche Aussagen in der Speisekarte | – | ✓ |

Diese Verantwortungsteilung ist Bestandteil des Lizenzvertrags / AVV.
