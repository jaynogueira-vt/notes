# Jay Nogueira

Personal site: articles, technical docs, proposals, RFCs, and reusable
AI-agent skills on AI-native software delivery and harness engineering.

Published via GitHub Pages: https://jaynogueira-vt.github.io/notes/

## Structure

- `index.html`: landing page (hub linking to the three surfaces below)
- `articles/index.html`: article listing
- `articles/<slug>/index.html`: one folder per article
- `docs/index.html`: docs / proposals / RFCs listing
- `docs/<slug>/index.html`: one folder per doc
- `skills/index.html`: AI-agent skills listing
- `skills/<slug>/index.html`: one folder per skill, with the raw `.md` alongside
- `assets/`: shared design system (CSS tokens, self-hosted fonts)
- `diagrams/`: article diagrams (SVG)

## Adding a new article

1. Create `articles/<slug>/index.html` (copy an existing one as template)
2. Add diagrams under `diagrams/<topic>/`
3. Add a card to `articles/index.html`

## Adding a new doc / proposal / RFC

1. Create `docs/<slug>/index.html` (copy `docs/software-factory/` as template)
2. Add a card to `docs/index.html`

## Adding a new skill

1. Create `skills/<slug>/index.html` (copy `skills/ux-ui-design-review/` as template)
2. Drop the raw skill Markdown in the same folder as `skills/<slug>/<slug>.md`
   so it's downloadable
3. Add a card to `skills/index.html`
4. Scrub the skill of any internal/proprietary context before publishing:
   personal paths, internal repo/ticket/channel names, client-specific labels.
   These are public.

## Design system

Light + dark theme (toggle persists to `localStorage`). Tokens in
`assets/site.css` (Tailwind v4 + shadcn naming); self-hosted fonts in
`assets/fonts.css`: Playfair Display (display), Source Serif 4 (body),
Inter (UI). Single accent `#057dbc`. Diagrams use `currentColor` and CSS
variables so they adapt to the active theme.
