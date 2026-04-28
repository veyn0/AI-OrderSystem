# Restaurantverwaltungs-Plattform — Pre-Development-Dokumentation

> **Status:** Pre-Development Spezifikation, Version 1.0
> **Stand:** 2026-04-28
> **Autor:** Plattform-Owner (Solo-Entwickler)
> **Zielgruppe:** Implementierende Person (Solo-Entwickler), zukünftige Mitwirkende, externe Reviewer

## Zweck dieses Dokuments

Dieses Dokumentations-Bundle ist die **vollständige Pre-Development-Spezifikation** der Restaurantverwaltungs-Plattform. Es definiert verbindlich:

- Vision, Scope und Geschäftsmodell
- Domänen-Modell, Entitäten, Use Cases
- Architektur und Tech-Stack
- Berechtigungs- und Sicherheitsmodell
- Nicht-funktionale Anforderungen
- Integrations-Schnittstellen
- Compliance-Plan
- Roadmap und Risiken

Das Dokument ist so detailliert, dass die Implementierung gestartet werden kann, ohne fundamentale Strukturentscheidungen erneut treffen zu müssen. Detail-Entscheidungen auf Implementierungs-Ebene (UI-Layouts, Library-Eigenheiten, Edge-Case-Verhalten) werden während der Entwicklung getroffen und nachträglich dokumentiert.

## Verzeichnis

### Top-Level

- [`vision-and-scope.md`](docs/vision-and-scope.md) — Vision, Scope, Geschäftsmodell, Phasen
- [`glossary.md`](docs/glossary.md) — Begriffsdefinitionen, einheitliche Terminologie
- [`risks.md`](docs/risks.md) — Risiko-Register mit Mitigations-Strategien

### Domain (`docs/domain/`)

- [`domain-model.md`](docs/domain/domain-model.md) — Entitäten, Beziehungen, Domain-Sprache
- [`use-cases.md`](docs/domain/use-cases.md) — Use-Case-Katalog mit Akteuren und Abläufen
- [`order-lifecycle.md`](docs/domain/order-lifecycle.md) — Bestell-Zustandsmaschine
- [`acceptance-policies.md`](docs/domain/acceptance-policies.md) — Annahme-Policy-Regelwerk
- [`permissions.md`](docs/domain/permissions.md) — Permission-Katalog und Rollen-Templates

### Architecture (`docs/architecture/`)

- [`overview.md`](docs/architecture/overview.md) — High-Level-Architektur, C4 L1+L2
- [`tech-stack.md`](docs/architecture/tech-stack.md) — Vollständiger Technologie-Stack
- [`modulith-structure.md`](docs/architecture/modulith-structure.md) — Modul-Aufteilung und Kommunikation
- [`tenancy.md`](docs/architecture/tenancy.md) — Mandantentrennungs-Konzept
- [`authentication.md`](docs/architecture/authentication.md) — Auth-Architektur (Web, Geräte, Mobile)
- [`audit-log.md`](docs/architecture/audit-log.md) — Audit-Logging-Konzept
- [`data-model.md`](docs/architecture/data-model.md) — Datenmodell-Detail
- [`standalone-mode.md`](docs/architecture/standalone-mode.md) — Standalone-Lauffähigkeit (Demo-Profil)
- [`live-updates.md`](docs/architecture/live-updates.md) — Server-Sent-Events-Architektur

### API (`docs/api/`)

- [`api-design-principles.md`](docs/api/api-design-principles.md) — REST-Konventionen, Versionierung
- [`ports-and-adapters.md`](docs/api/ports-and-adapters.md) — Externe Schnittstellen-Spezifikationen

### Operations (`docs/operations/`)

- [`nfr.md`](docs/operations/nfr.md) — Nicht-funktionale Anforderungen
- [`deployment.md`](docs/operations/deployment.md) — Deployment-Strategie
- [`backup-recovery.md`](docs/operations/backup-recovery.md) — Backup und Disaster Recovery
- [`monitoring.md`](docs/operations/monitoring.md) — Logging, Metriken, Alerting
- [`security.md`](docs/operations/security.md) — Sicherheits-Architektur

### Compliance (`docs/compliance/`)

- [`compliance-overview.md`](docs/compliance/compliance-overview.md) — Compliance-Themen-Übersicht
- [`gdpr.md`](docs/compliance/gdpr.md) — DSGVO-Umsetzung
- [`gobd-tse.md`](docs/compliance/gobd-tse.md) — GoBD und KassenSichV/TSE
- [`lmiv-impressum-agb.md`](docs/compliance/lmiv-impressum-agb.md) — LMIV, Impressum, AGB

### Roadmap (`docs/roadmap/`)

- [`roadmap.md`](docs/roadmap/roadmap.md) — MVP, Phase 2, Phase 3 mit Feature-Mapping

### Architecture Decision Records (`docs/adr/`)

18 ADRs, von `001-hexagonal-architecture.md` bis `018-table-qr-anti-abuse.md`.
Siehe [`docs/adr/README.md`](docs/adr/README.md) für die vollständige Liste.

## Lese-Empfehlung

**Bei erstem Lesen (in dieser Reihenfolge):**

1. `vision-and-scope.md` — Was bauen wir?
2. `glossary.md` — Wie sprechen wir?
3. `domain/domain-model.md` — Wie ist die Welt strukturiert?
4. `domain/use-cases.md` — Was passiert in der Welt?
5. `architecture/overview.md` — Wie ist das System aufgebaut?
6. `architecture/tech-stack.md` — Womit?
7. ADRs — Warum diese Entscheidungen?
8. Rest nach Bedarf

**Beim Implementieren als Referenz:**

- Datenmodell, Permissions, Lifecycle, Ports — diese vier sind die Hauptarbeitsdokumente während der Entwicklung.

## Konventionen

- **Status-Marker** in einzelnen Dokumenten: `[MVP]`, `[Phase 2]`, `[Phase 3]`, `[NICHT IM SCOPE]`
- **Pflicht-Anforderungen:** "MUSS", "DARF NICHT" (RFC 2119-Stil)
- **Empfehlungen:** "SOLLTE"
- **Optional:** "KANN"

## Lebende Dokumentation

Dieses Bundle ist Teil des Repositories und wird mit Code zusammen versioniert. Bei jeder größeren Architekturänderung:

1. Neuer ADR wird angelegt.
2. Betroffene Dokumente werden aktualisiert.
3. Commit-Message verweist auf den ADR.

Veraltete Dokumente erhalten den Vermerk `> SUPERSEDED BY ADR-NNN` ganz oben.
