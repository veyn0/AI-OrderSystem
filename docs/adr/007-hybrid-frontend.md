# ADR-007: Hybrid Frontend (Vaadin Flow + Thymeleaf+HTMX)

**Status:** Accepted
**Datum:** 2026-04-28

## Kontext

Die Plattform hat zwei sehr unterschiedliche UI-Domänen:

1. **Backoffice / Geräte-UIs** (Plattform-Admin, Restaurant-Backoffice, Bestelltablet, KDS, Self-Order-Terminal): komplex, hoch-interaktiv, viele Daten-Tabellen, Formular-lastig, Echtzeit-Updates. Nutzer sind eingeloggt, Performance ist wichtig aber nicht kritisch (Geschäfts-Anwendung).

2. **Online-Bestellseite** (öffentliche Endkunden-Bestellung): muss schnell laden, SEO-fähig sein, mobil-optimiert, leichtgewichtig. Erste Eindrücke entscheidend.

Solo-Entwickler mit primär Java-Erfahrung. Lernziel ist Java-zentriert. Reines JavaScript-SPA-Framework wäre für ihn ein Sprachwechsel und doppelte Entwicklungsumgebung.

## Entscheidung

**Hybrid-Frontend** in zwei Stilen:

- **Vaadin Flow Core** für Backoffice und Geräte-UIs.
  - Server-side Rendering, Java-zentriert.
  - Komponenten-Bibliothek, Forms, Datentabellen, Server-Push.
  - Apache 2.0 Lizenz (Vaadin Core, NICHT Pro Components).

- **Thymeleaf + HTMX + Tailwind CSS + Alpine.js** für die öffentliche Online-Bestellseite und die Demo-UI.
  - Server-rendered HTML.
  - HTMX für leichtgewichtige Interaktivität ohne SPA-Komplexität.
  - Tailwind CSS für Styling.
  - Alpine.js für kleine Client-Logik (Modals, Toggle).

## Alternativen

### Reines Vaadin
- Verworfen für Online-Bestellseite, weil:
  - Initial-Load von Vaadin Flow enthält das ganze Vaadin-Bundle (Megabyte-Bereich).
  - SEO eingeschränkt (Vaadin rendert Vaadin-spezifisches HTML).
  - Mobile-Performance auf Long-Tail-Geräten suboptimal.

### Reines Thymeleaf+HTMX
- Verworfen für Backoffice. Komplexe Datentabellen, Live-Updates, viele Form-Felder mit dynamischem Verhalten würden in HTMX viel Boilerplate verlangen. Vaadin gibt vorgefertigte Komponenten.

### React / Vue / Svelte SPA
- Verworfen.
  - Sprachwechsel zu TypeScript für den Solo-Entwickler.
  - Zwei separate Codebases (Frontend, Backend), API-Verträge zu pflegen.
  - Build-Toolchain (Webpack, Vite, etc.) zusätzlicher Stack.
  - Lernkurve und Tool-Stack-Komplexität würde MVP verzögern.

### Angular
Verworfen, gleiche Argumente plus stärkere Kopplung an spezifischem Framework.

### JSF (Jakarta Faces)
Verworfen. Vaadin ist moderner, aktiver gepflegt.

### Server-rendered Java-Templating (Pebble, Mustache, FreeMarker)
Erwogen für die öffentliche Seite, aber Thymeleaf ist Spring-Standard und gut integriert.

## Konsequenzen

### Positiv
- **Java-zentrierte Produktivität für Backoffice** — Solo-Entwickler bleibt im einen Stack.
- **Schnelle, SEO-freundliche öffentliche Seite** mit minimalen Assets.
- **HTMX** ist klein und passt zur „server-as-source-of-truth"-Philosophie der Anwendung.
- Beide UI-Stile teilen denselben Backend-Stack, dieselbe Domäne.
- Tailwind CSS ergibt konsistentes Design auf öffentlicher Seite.

### Negativ
- **Zwei UI-Stile** → leicht unterschiedliches Look-and-Feel zwischen Backoffice und Online-Seite. Bewusste Entscheidung.
- **Tailwind CSS Build-Schritt** für die öffentliche Seite. Maven-Plugin oder Pre-Build-Script nötig.
- Vaadin Flow Backend-Push verwendet Vaadins eigenen Mechanismus, der parallel zum SSE für externe Geräte existiert.

### Neutral
- Vaadin Core ist auf Apache 2.0 Lizenz beschränkt — keine Pro Components, kein Pro-Lizenz-Risiko.

## Querverweise

- [`../architecture/tech-stack.md`](../architecture/tech-stack.md)
- [`../architecture/overview.md`](../architecture/overview.md)
- [ADR-016](016-oss-only.md)
