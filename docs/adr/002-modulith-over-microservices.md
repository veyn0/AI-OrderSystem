# ADR-002: Modulith statt Microservices

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform wird von einem **Solo-Entwickler** gebaut und initial betrieben. Sie soll auf einem dedizierten Server laufen. Wachstumshorizont: 50 Tenants × 3 Standorte × 5 Geräte = 750 simultane Geräte. Es gibt eine klare Modulgrenze in der Domäne (Tenant, Order, Menu, etc.), aber keine zwingende Anforderung, einzelne Module unabhängig zu skalieren.

Architektur-Optionen reichen von Monolith ohne Modulgrenzen bis zu vollwertigen Microservices mit Service-Mesh.

## Entscheidung

Wir setzen auf einen **Modulith** (Spring Modulith). Eine deploybare Anwendung, intern in **14 Module** mit klaren Grenzen, durchgesetzt durch Spring Modulith / ArchUnit.

Module: `platform`, `tenant`, `auth`, `menu`, `order`, `payment`, `tse`, `customer`, `device`, `notification`, `reporting`, `web`, `api`, `demo`, `shared`.

## Alternativen

### Monolith ohne Modulgrenzen
Verworfen. Ohne Modul-Grenzen entstehen schnell zyklische Abhängigkeiten und „Big Ball of Mud". Refactoring später teuer.

### Microservices
Verworfen für MVP. Begründung:

- **Operations-Overhead** ist immens: Service-Discovery, verteiltes Tracing, mehrere DB-Instanzen, Inter-Service-Calls. Solo-Entwickler kann das nicht stemmen.
- **Verteilte Transaktionen** sind ein offenes Problem: Order + Payment + TSE müssen atomar sein. Im Modulith trivial, in Microservices saga-basiert.
- **Lokale Performance** leidet durch Netzwerk-Hops zwischen Services.
- **Kein klarer Skalierungsbedarf** für einzelne Module bei 750 Geräten.
- **Migration auf Microservices ist möglich**, falls jemals nötig: Spring Modulith verifiziert klare Grenzen, Module sind extraktionsfähig.

### Service-Oriented Architecture (SOA)
Im Wesentlichen Microservices mit anderem Vokabular. Gleiche Argumente gegen.

## Konsequenzen

### Positiv
- Eine deploybare Einheit, einfaches CI/CD.
- Refactorings über Modul-Grenzen sind sicher (Compiler hilft).
- Lokale Calls statt Netzwerk-Hops — einfacher und schneller.
- Eine DB → ACID-Transaktionen über alle Module möglich.
- Spring Modulith generiert Modul-Reports und ArchUnit-Tests.

### Negativ
- Skalierung ist nur ganzheitlich (mehr Server-Ressourcen für die ganze Anwendung). Bei einzelnen heißen Pfaden müsste man horizontal die ganze App skalieren.
- Stärkere Coupling-Gefahr als Microservices — daher Modul-Grenzen-Verifikation Pflicht.
- Bei sehr großem Wachstum (1000+ Tenants) wäre Microservice-Migration zu prüfen — Phase 4+.

### Neutral
- Codebase größer als ein einzelner Microservice, aber kleiner als die Summe vieler Services.

## Querverweise

- [`../architecture/modulith-structure.md`](../architecture/modulith-structure.md)
- [`../architecture/overview.md`](../architecture/overview.md)
- [ADR-001](001-hexagonal-architecture.md)
