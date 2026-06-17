# Jay Nogueira

Personal site: articles, technical docs, proposals, and RFCs on AI-native
software delivery and harness engineering.

Published via GitHub Pages: https://jaynogueira-vt.github.io/articles/

## Structure

- `index.html` — landing page (hub linking to the two surfaces below)
- `articles/index.html` — article listing
- `articles/<slug>/index.html` — one folder per article
- `docs/index.html` — docs / proposals / RFCs listing
- `docs/<slug>/index.html` — one folder per doc
- `assets/` — shared design system (CSS tokens, self-hosted fonts)
- `diagrams/` — article diagrams (SVG)

## Adding a new article

1. Create `articles/<slug>/index.html` (copy an existing one as template)
2. Add diagrams under `diagrams/<topic>/`
3. Add a card to `articles/index.html`

## Adding a new doc / proposal / RFC

1. Create `docs/<slug>/index.html` (copy `docs/software-factory/` as template)
2. Add a card to `docs/index.html`

## Design system

Light + dark theme (toggle persists to `localStorage`). Tokens in
`assets/site.css` (Tailwind v4 + shadcn naming); self-hosted fonts in
`assets/fonts.css` — Playfair Display (display), Source Serif 4 (body),
Inter (UI). Single accent `#057dbc`. Diagrams use `currentColor` and CSS
variables so they adapt to the active theme.
