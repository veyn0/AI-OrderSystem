# Compliance-Übersicht

> **Status:** Accepted
> **Letzte Aktualisierung:** 2026-04-28

Die Plattform muss verschiedene rechtliche Vorgaben einhalten. Dieses Dokument fasst die Themen zusammen und verweist auf die Detail-Dokumente.

## Anwendbare Vorschriften (Deutschland)

| Vorschrift | Bezug | Detail |
|---|---|---|
| **DSGVO** | Personenbezogene Daten von Endkunden, Mitarbeitern, Inhabern | [`gdpr.md`](gdpr.md) |
| **BDSG** | Begleitendes Bundesgesetz zu DSGVO | [`gdpr.md`](gdpr.md) |
| **GoBD** | Aufbewahrung steuerrelevanter Daten 10 Jahre | [`gobd-tse.md`](gobd-tse.md) |
| **KassenSichV** | Technische Sicherheitseinrichtung (TSE) für Bargeschäfte | [`gobd-tse.md`](gobd-tse.md) |
| **AO § 146a** | Aufzeichnungspflichten elektronischer Kassen | [`gobd-tse.md`](gobd-tse.md) |
| **LMIV** | Allergen-/Zusatzstoff-Angaben bei Lebensmittel-Verkauf | [`lmiv-impressum-agb.md`](lmiv-impressum-agb.md) |
| **TMG / DDG** | Impressumspflicht | [`lmiv-impressum-agb.md`](lmiv-impressum-agb.md) |
| **BGB** | AGB-Recht, Verbraucherschutz, Widerruf | [`lmiv-impressum-agb.md`](lmiv-impressum-agb.md) |
| **PAngV** | Preisangabenverordnung | [`lmiv-impressum-agb.md`](lmiv-impressum-agb.md) |

## Compliance-Status pro Phase

| Anforderung | MVP | Pre-Echtkunden | Phase 2 |
|---|---|---|---|
| Audit-Log mit 10-Jahres-Retention für Finanz-Daten | ✓ | ✓ | ✓ |
| TSE-Port mit Stub | ✓ | – | – |
| Echte TSE-Integration | – | ✓ | ✓ |
| DSGVO-Lösch-/Anonymisierungs-Workflow | ✓ | ✓ | ✓ |
| AVV-Template + Workflow für Tenants | – | ✓ | ✓ |
| Datenexport-API für Endkunden | ✓ | ✓ | ✓ |
| Allergen-Pflichtangaben pro MenuItem | ✓ | ✓ | ✓ |
| Impressum/AGB/Datenschutz pro Tenant | ✓ (Pflichtfelder) | ✓ (validiert) | ✓ |
| DATEV-Export | – | – | ✓ |
| Cookie-Banner | – | – | ✓ (falls Tracking) |
| Externe Datenschutz-Folgenabschätzung (DSFA) | – | erwägen | – |

## Verantwortlichkeiten

### Plattform-Owner
- Stellt technische Maßnahmen für DSGVO-Konformität bereit.
- Ist Auftragsverarbeiter ggü. Tenants.
- Stellt AVV-Vorlage zur Verfügung.
- Ist verantwortlich für Server-Sicherheit, Backups, Verfügbarkeit.

### Restaurant-Tenant (Inhaber)
- Ist Datenverantwortlicher ggü. seinen Endkunden.
- Schließt AVV mit Plattform-Owner ab.
- Pflegt Impressum, AGB, Datenschutzerklärung selbst.
- Verantwortet steuerliche Aufzeichnungspflichten und TSE-Anbindung.
- Verantwortet LMIV-konforme Allergen-Pflege.

### Endkunde
- Erteilt Einwilligung zur Datenverarbeitung beim Bestellprozess.
- Hat Auskunfts-, Berichtigungs-, Lösch-, Datenübertragungs-Rechte.

## Standardvertragliche Strukturen

### Plattform-Owner ↔ Tenant
- **Lizenz-/Service-Vertrag** (regelt Funktionsumfang, Preis, SLA).
- **Auftragsverarbeitungsvertrag (AVV)** (DSGVO Art. 28).

### Tenant ↔ Endkunde
- **AGB des Restaurants** (regelt Bestellung, Lieferung, Stornierung).
- **Datenschutzerklärung** (informiert über Datenverarbeitung).
- **Impressum** (Pflichtangaben).

## Audit / Verifikation

- Audit-Log enthält alle compliance-relevanten Aktionen (DSGVO-Lösch-Anfragen, Order-Lifecycle, Steuer-Overrides etc.).
- Reporting hat Drill-Down bis auf Bestell-Ebene für Steuerberater-Zugriff (Phase 2: extra Steuerberater-Rolle).
- 10-Jahres-Retention der Finanzdaten technisch erzwungen.

## Risiko-Bewusstsein

- TSE-Stub im MVP ist NICHT KassenSichV-konform — eine Echtkunden-Inbetriebnahme ohne echte TSE-Anbindung wäre **rechtlich problematisch**. Daher: keine Echtkunden vor TSE-Integration.
- Allergen-Pflicht beim Online-Verkauf ist streng — fehlende Angaben können abmahnfähig sein. Im MVP zeigen wir „Allergene auf Anfrage", was rechtlich grenzwertig ist; vor Echtkunden-Go-Live MUSS eine vollständige Allergen-Pflege erzwungen werden.
- Datenleck zwischen Tenants wäre DSGVO-meldepflichtig und reputational ruinös. Daher Defense-in-Depth bei Mandantentrennung.

## Zukünftige Themen

- **PSD2/SCA** (bei Online-Zahlungen): wird durch Payment-Provider abgedeckt (Stripe etc.).
- **EU-DLT/eIDAS** (digitale Signaturen): nicht im Scope.
- **Barrierefreiheits-Stärkungsgesetz BFSG (Juni 2025)**: Online-Bestellseite betroffen, WCAG 2.1 AA anstreben Phase 2.
- **EU AI Act**: aktuell keine KI-Komponenten, irrelevant.
- **NIS2-Richtlinie**: Restaurantbranche meist außerhalb des Anwendungsbereichs.
