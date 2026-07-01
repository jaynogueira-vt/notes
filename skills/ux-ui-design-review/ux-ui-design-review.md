# UX/UI Design Review (Hermes skill)

A repository-grounded design-review pipeline for any React/Next.js (or general
web) frontend. Captures the current state of a UI surface (screenshots + source
greps), runs a 13-category audit checklist against it, and produces a
prioritized findings report with concrete patches that reuse the project's
existing design tokens, never inventing new colors, spacings, or radii without
approval.

> Drop this into `skills/software-development/ux-ui-design-review/SKILL.md` in any
> agent that supports skills (or your skills dir) and adapt the file-path examples
> to your repo layout.

## What it prevents

- A new UI feature ships visually inconsistent with the rest of the app
  (hardcoded hex instead of tokens, off-grid spacing, mismatched type scale,
  missing skeleton, no empty/error state).
- A "polish pass" happens too late (after PR review, after deploy) when
  fixing it costs 10x.
- The reviewer's eye drifts toward "what looks pretty" instead of "what matches
  THIS app's existing design language".

## When to use (auto-load triggers)

Load this BEFORE designing or implementing if the task involves:

- "review this screen", "design review", "audit the UI", "is this good
  design?", "does this look right?", "polish this", "match the design system",
  "redesign this", "review my component", "I added a button/header/form/table",
  "new page", "new modal", any UI feature request, any task that ends with
  screenshots.
- File-pattern triggers: any task touching `*.tsx`, `*.jsx`, `*.css`, `*.scss`,
  `*.module.css`, `globals.css`, `tailwind.config.*`, `src/components/**`,
  `src/app/**/page.tsx`, `app/**/layout.tsx`.
- Phase triggers: end of any plan/spec cycle that produces UI: run the review
  on the spec BEFORE implementation, then again on the implementation BEFORE PR.

If unsure, load it. False positives are cheap; missed reviews ship regressions.

## Scope discipline: "review" means read-only until explicitly asked otherwise

If the user asks to **review / audit / check** an existing UI, treat it as a
read-only design review by default. If they ask for a **plan for improvements**,
that is still read-only: inspect, research, compare, produce the plan. Do **not**
create tracker tickets, start implementation, patch files, run formatters, or
open branches unless explicitly asked to apply fixes.

Safe read-only actions: read project rules/source/tokens; start a local dev
server and inspect with browser tools; run grep/static checks that don't mutate
files; produce prioritized findings with suggested fixes. Verify with
`git status` that nothing changed.

Unsafe without approval: editing code/docs (even for "obvious" gaps); creating
tracker tickets; committing, pushing, opening a PR.

If a review reveals P0 issues, end with "I can apply these fixes if you want"
rather than applying them.

## The 5-step workflow

### Step 1: Establish baseline (the project's design DNA)

Learn the project's existing visual language first. **Do not import outside
conventions.**

```bash
# Find the design tokens
grep -rE -- "--accent|--bg-|--fg-|--border|--panel" src/app/globals.css

# Find the spacing scale (Tailwind class usage)
grep -rE "\b(p|m|gap|space)-(0|0\.5|1|1\.5|2|3|4|5|6|8|10|12|16|20|24)\b" src/components | head -50

# Find typography scale
grep -rE "text-(xs|sm|base|lg|xl|2xl|3xl|4xl)" src/components | head -50

# Find existing primitives
find src/components -name "*.tsx" | head -50

# Find any hardcoded hex (anti-pattern signal)
grep -rE "#[0-9a-fA-F]{3,8}\b" src/components | head -50
```

Output of Step 1: a short "design DNA" memo: color palette, spacing increments
actually in use, type scale actually in use, button styles, panel styles, which
icon set is imported.

### Step 2: Capture the surface under review

**For an existing page**: navigate to it, describe the visual hierarchy /
alignment / spacing / inconsistencies, and read the accessibility tree (reveals
semantics + missing labels). If the page needs auth and you can't log in, ask
for a screenshot.

**For a spec/plan (forward-looking review before implementation)**: the
highest-leverage use case. Reviewing a "Visual Spec" section in a written plan
BEFORE code ships catches the most expensive class of regressions (parallel-component invention) while the fix is still a markdown edit, not a
refactor PR.

- Run a **primitive-reuse audit** as the dominant check: list every existing
  primitive, then for each component the plan proposes, grep for prior art with
  similar responsibility (avatar, badge/chip/pill, button, selector/segmented,
  sparkline/chart, drawer/modal/sheet). If existing code covers >70% of the
  proposed surface, the plan should reuse it, not duplicate. "This plan creates
  `<X>` but `<Y>` already exists and does the same thing" is a P0 every time.
- **Audit token references.** Vague plan language ("accent border", "subtle
  hover") becomes hardcoded hex in code. Force the exact token name before
  approval.
- For findings, cite the plan section AND the codebase evidence so the fix is a
  markdown patch.

**For a PR/diff**: `git diff` the changed files, open the deploy preview if one
exists, capture it visually.

### Step 3: Run the 13-category audit checklist

(See below.) For each category, mark Pass / P0 / P1 / P2 / N-A and capture
evidence (file:line, screenshot region, or grep result).

### Step 4: Build the findings report

Every finding cites: **Where** (file:line or screenshot region), **What** (the
issue), **Why** (which checklist item it violates), **Fix** (concrete patch
using **existing tokens**, never invented values).

### Step 5: Hand back

Deliver findings as plain markdown. Offer to apply P0 fixes (with approval),
file follow-ups for P2, and stop to discuss any P0 needing a design decision.
Do NOT silently apply fixes; design changes need sign-off even when "obviously
right".

## Audit checklist

### 1. Visual hierarchy
- [ ] Primary action is visually dominant (size, color, position).
- [ ] Page has exactly one h1, used for the page title.
- [ ] Secondary actions don't compete with primary (no two accent buttons side
      by side).
- [ ] Z-axis (elevation) is consistent: drawers/modals > headers > panels >
      body.

### 2. Typography
- [ ] Type scale matches existing (no novel `text-[15px]` when the project uses
      `text-sm` everywhere).
- [ ] Body text >= 14px; never below 12px for primary content.
- [ ] Numerals use `tabular-nums` in tables, KPIs, or anywhere users compare
      digits column-wise.
- [ ] Font weight is purposeful: semibold for emphasis, medium for active
      states; avoid mid-weight noise.
- [ ] No mixed font families unless intentional (e.g. mono for keys/code).

### 3. Spacing & alignment
- [ ] Spacing follows the project's existing increments (Step 1 result). For
      Tailwind: usually 1/2/3/4/6/8 (~4/8/12/16/24/32px).
- [ ] No off-grid values like `mt-[7px]` unless solving a descender/icon-center
      problem (and then commented).
- [ ] Cards/panels have consistent internal padding: one padding step per panel
      size.
- [ ] Vertical rhythm: gaps between sibling sections are deliberate, not
      arbitrary.
- [ ] Alignment: table columns align consistently; numeric columns right-align;
      icons vertically center with their label.

### 4. Color & contrast (WCAG)
- [ ] Foreground/background pairs hit WCAG AA (4.5:1 normal text, 3:1 for >=18pt
      or bold >=14pt). Test critical pairs with a contrast tool.
- [ ] No hardcoded hex in component files: all colors via CSS variables or the
      theme tokens. Grep `#[0-9a-fA-F]{3,6}` in components and audit every hit.
- [ ] Status colors are consistent app-wide: one success, one warning, one
      danger token, reused.
- [ ] Color is never the SOLE carrier of meaning: pair red/green with icon +
      text label.
- [ ] Dark-mode safe: if the project is dark-only, confirm the
      `prefers-color-scheme` defaults.

### 5. Design-token consistency
- [ ] Every color/spacing/radius/shadow value resolves to a token. New values
      get a comment AND a "should this be a token?" question.
- [ ] No duplicate tokens with slightly different values (`--bg-1` vs
      `--background-primary`, pick one).
- [ ] Hover/focus/active states use derived tokens (e.g. `color-mix(...)`), not
      new hex.

### 6. Loading / empty / error trinity
- [ ] Every async data region has a sibling skeleton with matching layout
      dimensions (no layout shift).
- [ ] Every list/table has an explicit empty state: icon + headline + helper
      copy + optional CTA. Never a blank panel.
- [ ] Every fetcher has an error state: short message + retry button. Never a
      silent failure or a "stuck loading forever" UI.
- [ ] Skeletons render the correct row count for the page size.

### 7. Interactions & microinteractions
- [ ] Clickable elements have visible cursor and at least one of hover/focus/
      active state.
- [ ] Buttons disable themselves while their action is in flight (no
      double-submit).
- [ ] Form fields validate on blur, not every keystroke, unless validation is
      genuinely "as you type".
- [ ] Toasts auto-dismiss after 4-6s for success, sticky for errors.
- [ ] Animations respect `prefers-reduced-motion`.

### 8. Accessibility
- [ ] All interactive elements keyboard-reachable (Tab / Shift+Tab / Enter /
      Space / arrows for menus and segmented controls).
- [ ] Visible focus ring on every focusable element.
- [ ] Form fields have an associated label (or `aria-label`).
- [ ] Icon-only buttons have `aria-label` or `title`.
- [ ] Heading levels are sequential: no h1 -> h3 jumps.
- [ ] Tables use `<th scope="col">` for header cells.

### 9. Responsive behavior
- [ ] Renders at 1280x800 (desktop) without overflow.
- [ ] Renders at 768x1024 (tablet): secondary content can stack, no horizontal
      scroll.
- [ ] Renders at 375x667 (mobile): primary content readable.
- [ ] If desktop-only by design, document it and degrade gracefully.

### 10. State consistency
- [ ] Same backend concept (status, bucket, presence) shows the same value
      everywhere it appears on the page.
- [ ] Action buttons reflect current state: show only legal actions, disable
      in-flight actions.

### 11. Empty payload / pagination edge cases
- [ ] First page, last page, and middle page all render correctly.
- [ ] page > totalPages clamps to last page (no "Page 99 of 3").
- [ ] Total count is shown ("Showing 1-25 of 231").
- [ ] Empty filter result has its own empty state, distinct from "no data
      exists at all".

### 12. Copy & microcopy
- [ ] Single language across the surface (no stray second language leaking in).
- [ ] Sentence case for buttons and labels unless the brand mandates otherwise.
- [ ] Error messages tell the user WHAT to do next, not just what went wrong.
- [ ] Empty states have personality without being cute (one line of context +
      one CTA).

### 13. Metric explainability (opaque-label class)

Fire a P0/P1 when the surface has numbers or labels that hide a formula,
taxonomy, or business rule the user can't infer from context.

- [ ] Every **composite/derived score** (leaderboard score, health score, risk
      score, NPS, quality grade) carries a tooltip/info icon revealing the
      formula, weights, or aggregation rule. Not optional: users WILL ask "what
      does this mean?" the first time.
- [ ] Every **non-obvious metric column** carries a one-sentence tooltip
      explaining what the count represents and the time window.
- [ ] Every **status/classification badge** with a non-trivial taxonomy
      ("Stuck", "At risk", "On track", bucket names) carries a tooltip
      explaining the rule that placed the row in that state.
- [ ] **KPI cards** use a tooltip for any number that is a sum/average/
      percentage rather than a raw count.
- [ ] **Tooltip text is concise**: one sentence, <= ~140 chars; long
      explanations go behind a docs link.
- [ ] **Tooltip text stays in sync with the source of truth**: if the formula
      lives in a SQL view or a constant, the tooltip repeats those weights
      verbatim. Add a `// keep in sync with the tooltip on /<route>` comment next
      to the formula.
- [ ] **Self-explanatory columns get NO tooltip** (`name`, `rank`, `email`).
      Tooltips on obvious labels are noise.

## Output format

```markdown
# UX/UI Design Review: <route or component>

**Reviewed:** YYYY-MM-DD
**Scope:** <single sentence>

## Summary
- N P0 (broken UX, must fix before ship)
- N P1 (polish, should fix before ship)
- N P2 (nice-to-have, file as follow-up)

## Findings

| # | Pri | Category | Where | What | Why | Fix |
|---|-----|----------|-------|------|-----|-----|
| 1 | P0 | Color & contrast | Hero.tsx:42 | Score text on bg = 3.1:1 | WCAG AA fail (4.5:1 needed) | Use the fg token (8.7:1) |
| 2 | P1 | Typography | Table.tsx:88 | Score column not `tabular-nums` | Comparison columns need monospaced digits | Add `tabular-nums` to the `<td>` |
| 3 | P2 | Microinteractions | Selector.tsx | No arrow-key nav between segments | Segmented controls usually support arrows | Add ArrowLeft/ArrowRight handler |

## Patches (P0 only, applied immediately if approved)
(ready-to-apply diff blocks here, if approved)

## Follow-ups (P2, suggest as separate issues)
- [ ] Arrow-key nav on segmented controls: affects all consumers.
```

## Anti-patterns (don't do these)

- **Importing outside conventions.** "Stripe does it this way" is not a finding
  unless the project's design system already aligns with Stripe.
- **Inventing new tokens.** Suggesting a new color/spacing value without first
  asking "should we add this to the token file?" is a defect of the review.
- **Pixel-perfect tax.** Don't flag every 1-2px offset unless it accumulates
  into visible misalignment. Focus on systemic issues.
- **Generic best-practices dump.** Every finding cites real evidence in the
  codebase. "Add micro-interactions" without naming a button is noise.
- **Ignoring the user's explicit priorities.** If they said "desktop-only v1",
  don't open 12 mobile findings. Mention it once, move on.
- **Reporting wins.** This is a review, not a status update. Only deviations
  belong in the table.
- **Turning a review into implementation.** Findings first; fixes only after
  approval.

## Pitfalls

1. **Confused with a correctness/security code review.** That targets logic,
   types, security; THIS targets visual/UX. They run side-by-side, not as
   substitutes.
2. **Confused with functional QA.** Functional QA asks "does the button work?";
   this asks "does the button look right and fit the system?" Both can run on
   the same page.
3. **Auto-loaded but doesn't help.** If the task is purely backend (a route file
   with no UI implications) the skill should self-detect and bow out in one line,
   don't force a checklist on a non-UI change.
4. **Vision over-reads.** Screenshot analysis can hallucinate spacing in pixels,
   always cross-check with source grep before flagging.
5. **Forgetting the project's hard rules.** Read `AGENTS.md` / `CLAUDE.md` /
   `README` / `CONTRIBUTING` first, or the review will recommend things that
   violate project policy (e.g. "skeletons required everywhere", "no new
   vendors").
6. **Missing parallel-primitive invention is the dominant P0 class.** When a
   plan proposes new components, the FIRST grep should be the components dir,
   looking for prior art with similar responsibility. Skip this and you ship a
   parallel maintenance burden.
7. **Vague spec language hides hardcoded hex.** "Accent border", "subtle hover",
   "warm palette" in a plan become arbitrary `#...` values in code. Force the
   exact token name before approval.
8. **Findings without companion-task follow-through.** When two surfaces share
   an inline pattern, the fix is two-step: (a) extract a primitive for the new
   code, AND (b) add a task to refactor the existing call site to use it too.
   Skipping (b) leaves the duplication.
9. **Opaque metrics ship without explainers.** Composite scores and non-obvious
   columns are the highest-ROI P1 finding: users ask "what does this mean?" the
   day after ship. The fix is small (a tooltip + one sentence) but only cheap if
   flagged during review.

## A note on non-traditional surfaces

The same "treat it as a UI, audit it against a system" lens applies beyond web
pages, e.g. a status report rendered as rich message blocks (treat it as a UI
surface, not a plaintext dump; lead with the reader's language and the headline
question; preserve a previously-accepted section taxonomy when redesigning
rather than silently dropping fields). Adapt the checklist categories that make
sense and skip the ones that don't.

## Tools this skill uses

- file/content search: find tokens, primitives, hex codes in the codebase
- file read: inspect specific components
- browser navigate + visual capture: capture deployed pages
- accessibility-tree snapshot: read semantics + missing labels
- terminal: `git diff`, `npx eslint --format=json`, contrast checks, etc.
