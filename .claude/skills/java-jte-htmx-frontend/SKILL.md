---
name: java-jte-htmx-frontend
description: Author server-rendered frontend pages and fragments using JTE templates, HTMX, and Pico CSS. Use when adding/modifying .jte templates, HTMX-driven interactions, or page styling.
---

# JTE + HTMX + Pico CSS Frontend

## Layout

- Templates live in `src/main/jte/`.
- Full pages at the root: `src/main/jte/home.jte`, `bookings.jte`, etc.
- HTMX fragments under `src/main/jte/fragments/`.
- Shared layout: `src/main/jte/layout.jte` — wraps pages with `<head>`, includes, and common chrome.
- Custom CSS goes in `src/main/resources/static/css/` (served from `/css/...`).

## Required CDN Includes

Use these exact tags in the layout (do not change versions/hashes):

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
<script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.8/dist/htmx.min.js"
        integrity="sha384-/TgkGk7p307TH7EXJDuUlgG3Ce1UVolAOFopFekQkkXihi5u/6OCvVKyz1W+idaz"
        crossorigin="anonymous"></script>
```

## Markup — Semantic HTML First

Pico styles elements directly. Prefer semantic tags over class soup:

```html
<!-- GOOD -->
<main class="container">
  <article>
    <header><h2>Bookings</h2></header>
    <table>
      <thead><tr><th>Time</th><th>Master</th></tr></thead>
      <tbody>...</tbody>
    </table>
  </article>
</main>

<!-- AVOID -->
<div class="card"><div class="card-header">...</div></div>
```

Only reach for classes when Pico needs them (`.container`, `.grid`, `.contrast`, etc.) or for project-specific styles.

## HTMX Patterns

- View controller endpoints return either a full page template or a fragment template.
- Use `hx-get`/`hx-post` on the triggering element; target a container with `hx-target` and `hx-swap`.
- Fragment endpoints return only the fragment template (no layout).

```html
<button hx-get="/bookings/new" hx-target="#booking-form" hx-swap="innerHTML">
  New booking
</button>
<div id="booking-form"></div>
```

```java
@GetMapping("/new")
public String newForm() {
    return "fragments/booking-form";  // fragment, no layout
}
```

## CSS

- Project-specific CSS files under `src/main/resources/static/css/`.
- Reference from layout: `<link rel="stylesheet" href="/css/app.css">`.
- Keep selectors element/attribute-based to stay aligned with Pico's semantic approach.