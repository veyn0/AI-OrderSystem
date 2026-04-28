# ADR-017: Endkunden-Konten pro Tenant

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Endkunden (Restaurant-Gäste) bestellen online beim jeweiligen Restaurant. Frage: Sollten Endkunden ein **plattformweites Konto** haben (eine E-Mail = ein Konto, kann bei vielen Restaurants bestellen), oder **pro Tenant ein separates Konto**?

## Entscheidung

**Endkunden-Konten sind pro Tenant.** Eine E-Mail-Adresse kann in mehreren Tenants als separates Konto existieren. Die Konten sind nicht miteinander verknüpft.

Tabellen-Constraint:
```sql
ALTER TABLE customers ADD CONSTRAINT customers_email_per_tenant_unique
  UNIQUE (tenant_id, email);
```

## Alternativen

### Plattformweite Endkunden-Konten
Erwogen, verworfen:

- **DSGVO-Probleme**: Wer wäre Datenverantwortlicher? Ein plattformweites Konto würde Plattform-Owner ggü. Endkunde verantwortlich machen — anders als das Auftragsverarbeiter-Modell mit dem Restaurant.
- **Konflikt mit Mandantentrennung**: Endkunde sieht Bestellhistorie über mehrere Tenants → Tenants sehen, dass Kunde X auch bei Tenant Y bestellt hat. Wettbewerbs-rechtlich heikel.
- **Komplexere Auth-Flow**: Endkunde wählt zuerst Plattform-Login, dann Tenant — UX-Bruch.
- **Account-Hijacking-Risiko**: Ein Leak im Plattform-Konto würde Bestellhistorie über alle Tenants offenlegen.
- **Marketing-Vorteile** (z.B. plattformweite Werbung): widerspricht Restaurant-Selbstbestimmung.

### Hybrid: Plattform-Login + Tenant-Switch
Verworfen, identische Probleme wie plattformweit.

### Magic-Link-only ohne dauerhafte Konten
Erwogen für sehr leichten Onboarding, akzeptabel, aber kein vollwertiger Ersatz.
- Wir bieten Magic-Link **als Login-Option**, nicht als Konten-Ersatz.

## Konsequenzen

### Positiv
- **DSGVO-Sauberkeit**: Restaurant ist klar Datenverantwortlicher seiner eigenen Endkunden-Daten.
- **Mandantentrennung total**: keine Cross-Tenant-Verknüpfung über Endkunden.
- **Restaurant hat eigene Kundenliste** — auch für Marketing-Zwecke (mit Endkunden-Zustimmung) nutzbar.
- **Keine Account-Hijacking-Implikationen** über Tenants hinweg.

### Negativ
- **Endkunde, der bei mehreren Restaurants der Plattform bestellt**, muss mehrfach registrieren.
  - Mitigation: Magic-Link-Login senkt die Hürde stark — kein Passwort merken nötig.
  - Mitigation Phase 2: Browser-Autofill mit gespeicherten Daten.
- **Daten-Redundanz**: gleicher Endkunde mehrfach in der DB, verschiedene Tenants.

### Neutral
- Endkunden-Datenmenge insgesamt wächst etwas, aber moderat.

## Implementations-Hinweise

- Beim Endkunden-Onboarding wird der Tenant-Kontext aus URL bestimmt (siehe [`../architecture/tenancy.md`](../architecture/tenancy.md)).
- Login-Endpoint sucht in `customers` mit `(tenant_id, email)`.
- E-Mail-Verifikation mit Magic-Link bei erstem Login.
- Bei DSGVO-Auskunft: Endkunde muss pro Tenant Auskunft anfragen, da Konten getrennt sind.

## Querverweise

- [`../architecture/authentication.md`](../architecture/authentication.md)
- [`../architecture/tenancy.md`](../architecture/tenancy.md)
- [`../compliance/gdpr.md`](../compliance/gdpr.md)
