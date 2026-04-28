# ADR-003: Java + Spring Boot als Backend-Stack

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Solo-Entwickler mit primärer Backend-Erfahrung in **Java**. Lernziel ist es, eine professionelle, produktionsreife Plattform zu bauen — kein Framework-Experiment. Die Sprache und das Framework müssen produktiv, langfristig wartbar und mit großem Ökosystem ausgestattet sein.

## Entscheidung

- **Sprache:** Java 21 (LTS).
- **Framework:** Spring Boot 3.x.
- **Build:** Maven.
- **Modulstruktur:** Spring Modulith.

## Alternativen

### Kotlin + Spring Boot
Erwogen, verworfen. Kotlin ist hervorragend, aber:
- Erfahrung des Entwicklers ist primär Java.
- Spring-Ökosystem-Doku ist überwiegend Java-zentrisch.
- Mischung Java/Kotlin im selben Codebase wäre möglich, aber Disziplin verlangend.

Kotlin könnte später für native Android-App (Phase 2) eingesetzt werden — dort ist es Standard.

### Java + Quarkus
Quarkus ist schnell und ressourcen-effizient (GraalVM Native), aber:
- Kleineres Ökosystem als Spring.
- Ungewohnter für den Entwickler.
- Spring Modulith fehlt äquivalent.
- Native Image-Builds mit GraalVM sind komplex bei dynamischem Reflection-Bedarf.

### Java + Micronaut
Ähnlich Quarkus, kleinere Community.

### Go
Verworfen. Sprachwechsel zu groß, Ökosystem für Web-Apps weniger reif als Spring.

### Node.js (TypeScript) + NestJS
Verworfen. Entwickler-Stack-Mismatch. Auch wenn NestJS Spring-Konzepte adaptiert, fehlt die Tiefe.

### Rust (Actix, Axum)
Sehr attraktiv für Performance und Sicherheit, aber:
- Lernkurve enorm.
- Bibliotheks-Ökosystem für Geschäfts-Apps weniger reif (kein JPA-Äquivalent in der Reife von Hibernate).
- Solo-Projekt mit Lerngrenzen — Rust würde alle Energie binden.

### .NET / C#
Verworfen. Außerhalb der Erfahrung des Entwicklers, ähnliche Größenordnung wie Java/Spring.

## Konsequenzen

### Positiv
- Höchste Produktivität für den Entwickler.
- Spring Boot Auto-Configuration spart Boilerplate.
- Spring Modulith → Modul-Verifikation eingebaut.
- Riesiges Ökosystem: JPA/Hibernate, Spring Security, Spring Session, Vaadin Flow, etc.
- Java 21 hat Records, Pattern Matching, virtuelle Threads — modernes Feature-Set.
- Java LTS bedeutet 8+ Jahre Support.
- Tooling exzellent (IntelliJ, Maven, Spring Tools).

### Negativ
- Höherer Speicher-Footprint als Quarkus / Go / Rust (initial ~300-500 MB JVM-Overhead).
- Kaltstart-Zeit (~5-15 s) länger als bei Native-Image-Frameworks. Für unsere Always-on-App irrelevant.
- Image-Größe größer (250+ MB).

### Neutral
- Java-Verbositäts-Argument ist mit Java 21-Records weitgehend entschärft.

## Querverweise

- [`../architecture/tech-stack.md`](../architecture/tech-stack.md)
- [ADR-002](002-modulith-over-microservices.md), [ADR-007](007-hybrid-frontend.md)
