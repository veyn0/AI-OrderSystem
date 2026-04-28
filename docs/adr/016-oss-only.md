# ADR-016: Open-Source-only

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Solo-Entwickler-Projekt mit:
- Privatem Lernziel zunächst.
- Kommerzielle Option (Echtkunden) später.
- Budget < 15 €/Monat externe Dienste in Demo-Phase.
- Wunsch nach Daten-Hoheit (Standalone-Modus).

Lizenz-Wahl von Bibliotheken hat langfristige Implikationen — eine kommerzielle Bibliothek im MVP würde später bei Kommerz-Use eine Lizenzgebühr nach sich ziehen oder Migrations-Aufwand bedeuten.

## Entscheidung

**Open-Source-only-Politik.** Keine Bibliothek, deren Lizenz kommerzielle Nutzung beschränkt oder Gebühren verlangt.

**Erlaubte Lizenzen** (Whitelist):
- Apache 2.0
- MIT
- BSD (alle Varianten)
- EPL 1.0/2.0
- LGPL 2.1/3.0
- MPL 2.0
- PostgreSQL License
- Public Domain / Unlicense / CC0

**Verbotene Lizenzen** (Blacklist, ohne Einzelfall-ADR):
- GPLv2 / GPLv3 ohne Classpath Exception (für Bibliotheken)
- AGPL
- Kommerzielle Lizenzen ("Free for Non-Commercial Use", "Pro-Edition")
- "Custom" Lizenzen ohne Klarheit

**CI-Check:** Maven License Report wird im CI generiert. Verbotene Lizenzen brechen den Build.

## Alternativen

### Pragmatisch erlauben (z.B. Vaadin Pro Components)
Verworfen.
- Vaadin Core (Apache 2.0) ist ausreichend für unsere Bedürfnisse.
- Pro Components würden monatliche Lizenzgebühr nach sich ziehen.
- Lock-in-Risiko bei Lizenz-Änderungen.

### Nur Apache/MIT (sehr streng)
Verworfen.
- Würde Hibernate (LGPL), Logback (EPL/LGPL) ausschließen — Standard-Bibliotheken.
- Whitelist mit EPL/LGPL ist Industrie-Praxis.

### AGPL erlauben
Verworfen.
- AGPL kann bei SaaS-Nutzung den eigenen Code virusartig „infizieren".
- Bei kommerzieller Plattform-Variante problematisch.

## Konsequenzen

### Positiv
- **Keine Lizenzgebühren** über die gesamte Lebensdauer.
- **Standalone-Modus zulässig**: Tenant kann selbst auf eigenem Server hosten ohne Lizenz-Komplikationen.
- **Klare Compliance** durch CI-Check.
- **Vorhersagbarkeit**: keine Überraschungen bei Provider-Lizenz-Änderungen.

### Negativ
- **Manche kommerzielle Bibliotheken** sind besser/produktiver — wir verzichten ggf. auf Komfort.
- **Vaadin Pro Components** (z.B. CRUD-Komponente, Charts) müssten selbst gebaut werden oder mit OSS-Alternativen ersetzt.
- **Bei manchen Themen** (Reporting-Tools, BI) gibt es weniger reife OSS-Alternativen.

### Neutral
- Whitelist ist regelmäßig zu überprüfen — neue Lizenz-Trends (z.B. Server Side Public License SSPL bei MongoDB) müssen adressiert werden.

## Konkrete Bibliotheken

Stichprobe der gewählten Bibliotheken und ihrer Lizenzen:

| Bibliothek | Lizenz |
|---|---|
| Spring Boot, Spring Security, Spring Session | Apache 2.0 |
| Hibernate ORM, Bean Validation | LGPL 2.1 |
| jOOQ OSS Edition | Apache 2.0 |
| Flyway Community Edition | Apache 2.0 |
| PostgreSQL | PostgreSQL License |
| zonky-test/embedded-postgres | Apache 2.0 |
| Vaadin Flow Core | Apache 2.0 |
| Thymeleaf | Apache 2.0 |
| Tailwind CSS | MIT |
| HTMX | BSD-2 |
| Alpine.js | MIT |
| Jackson | Apache 2.0 |
| Logback | EPL 1.0 / LGPL 2.1 |
| Apache PDFBox | Apache 2.0 |
| ZXing (QR-Codes) | Apache 2.0 |
| Argon2-jvm / Spring Security Argon2 | Apache 2.0 |
| Caddy | Apache 2.0 |
| Redis | Redis Source Available License (RSAL) — siehe Hinweis unten |

**Hinweis zu Redis:** Seit 2024 hat Redis seine Lizenz von BSD-3 auf RSAL/SSPL geändert. Für unsere Standalone- und Self-Hosting-Nutzung ist die RSAL akzeptabel; allerdings besteht ein **Restrisiko**, falls Redis später strenger wird. Alternative: **Valkey** (Linux Foundation Fork, BSD-3) oder **KeyDB**. Vor Echtkunden-Go-Live re-evaluieren.

## Querverweise

- [`../architecture/tech-stack.md`](../architecture/tech-stack.md)
- [`../operations/security.md`](../operations/security.md)
