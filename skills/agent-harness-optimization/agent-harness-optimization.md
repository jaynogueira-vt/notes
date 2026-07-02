# Agent Harness Optimization (skill)

A method for auditing and standardizing a repository's AI-agent guardrail
harness: the `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `.cursor/rules`, and
`.github/copilot-instructions.md` files that tell AI coding agents (Claude
Code, Cursor, Copilot, Gemini, and any skill-aware agent) how to behave in your
codebase. It converges every repo on a single-source-of-truth architecture,
makes the rules blocking in CI instead of advisory, and ships the whole thing
as one reviewable PR per repo, with a zero-cost watchdog that babysits the PRs
to merge.

> Drop this into `skills/agent-harness-optimization/SKILL.md` in any agent that
> supports skills (or your skills dir). Save the watchdog script at the end as
> `scripts/babysit-prs.sh` next to it.

## What it prevents

- The same rules copy-pasted into three tool-specific files, already drifted
  from each other (one file says the UI library is v7, another says v9).
- Agents flying blind because the repo has an `AGENTS.md` but no
  `copilot-instructions.md` or `GEMINI.md`, so half your tools never see the
  rules.
- A 900-line monolithic rules file that burns the model's context before it
  reads the code.
- Rules the repo itself violates (a "Vitest only" rule next to a live Jest
  config; a gofmt mandate with 29 unformatted files), which teach agents that
  the rules are decorative.
- "Pre-push gates" that no CI job actually runs.

## When to use (auto-load triggers)

Load this when the task is any of: "optimize AGENTS.md", "standardize agent
instructions across repos", "our agent rule files drifted", "add Copilot or
Gemini or Cursor coverage", "split this huge AGENTS.md", "make agent rules
blocking in CI", or an audit of AI guardrails across a set of repositories.

## Target architecture: SSOT plus pointers

```
AGENTS.md                          <- SINGLE source of truth. All rules live here.
CLAUDE.md                          <- 3-line pointer: "Read AGENTS.md first..."
GEMINI.md                          <- same pointer
.github/copilot-instructions.md   <- same pointer
.cursor/rules/repo-guardrails.mdc  <- pointer with alwaysApply: true frontmatter
```

For LARGE rule sets (400+ lines), `AGENTS.md` becomes a ROUTER (~90 lines):

```
AGENTS.md                    <- router: hard rules (always apply) + "your change touches X -> read Y" table
.agents/rules/workflow.md    <- pre-push gates, PR conventions
.agents/rules/testing.md     <- test runner rules, coverage gates
.agents/rules/<domain>.md    <- focused per-domain files
```

Rationale: a huge monolith makes models lose context; the router loads only
what the change touches. Small repos (a 100-160 line `AGENTS.md`) do NOT need
the router; keep one file plus pointers.

Pointer file template (the whole file, nothing else):

```markdown
# <Tool> configuration

> Read `AGENTS.md` first. It is the single source of truth contract[, routing
> to focused rule files under `.agents/rules/`]. Its "Hard Rules" section is
> blocking, not advisory. Do not duplicate rules here.
> [Repo-critical warning worth repeating, e.g. "All PRs target `staging`,
> never `main`."]
```

The Cursor pointer needs `.mdc` frontmatter or Cursor never loads it:

```markdown
---
description: <Repo> rules. All agent guardrails live in AGENTS.md
globs:
alwaysApply: true
---

All rules for this repo live in [AGENTS.md](mdc:AGENTS.md). Read it before any
change. Its "Hard Rules" section is blocking, not advisory. Do not duplicate
rules here; this file exists only so Cursor loads the pointer.
```

## Step 1: audit each repo

Inventory plus drift detection:

```bash
for f in AGENTS.md CLAUDE.md GEMINI.md .github/copilot-instructions.md; do
  [ -f "$f" ] && echo "$f: $(wc -l < $f) lines" || echo "$f: MISSING"; done
ls .cursor/rules/*.mdc 2>/dev/null
```

Look for, in priority order:

1. DUPLICATION: the same rules copy-pasted across tool files. Diff them; drift
   is usually already present.
2. MISSING coverage: no `copilot-instructions.md` or `GEMINI.md` means those
   agents never see the rules.
3. SELF-CONTRADICTION: rules the repo itself violates. Fix the violation in
   the same PR, or the harness teaches agents to ignore rules.
4. UNENFORCED gates: "pre-push gate" rules with no CI job running them.
5. STALE facts: wrong library versions, references to removed configs, empty
   table rows.

## Step 2: work in isolated worktrees

Never touch the user's local checkouts, and never reuse a stale local branch.
Fetch first, then cut a fresh branch off the CURRENT origin default:

```bash
cd <local-checkout>
git fetch origin --prune
DEFAULT=$(gh repo view <owner>/<repo> --json defaultBranchRef --jq .defaultBranchRef.name)
git worktree add /tmp/harness-review/<alias> -b ai/agent-harness-optimization origin/$DEFAULT
```

## Step 3: transform

Per repo: rewrite `AGENTS.md` as SSOT (router if large), shrink every other
file to a pointer, create the missing pointers, fix self-contradictions
(delete stale configs, run the formatter, etc.), and add CI enforcement for
the repo's own gates. Go repo CI example:

```yaml
- name: gofmt
  run: |
    UNFORMATTED=$(gofmt -l $(git ls-files '*.go'))
    if [ -n "$UNFORMATTED" ]; then echo "$UNFORMATTED"; exit 1; fi
- run: go build ./...
- run: go vet ./...
- run: make test
```

Quote-safe variant for exotic paths: `git ls-files -z '*.go' | xargs -0 gofmt -l`.
Do NOT suppress stderr or add `|| true`: parse errors must fail the gate
loudly, not hide inside a pipeline.

### Content rules worth baking into every harness

Adapt to your house style, but these earn their place:

- Comment discipline: never cite ticket keys or PR numbers in code comments
  (deleted tickets take the context with them; the comment must carry the full
  reasoning on its own). No overcommenting: code self-documents; comment only
  the non-obvious why, never what a line does. No commented-out code. Track
  follow-up work in the tracker, not in TODO comments.
- Surface the repo's number-one destructive-mistake risk in EVERY pointer
  file, not just `AGENTS.md` (example: "PRs target `staging`, never `main`"
  for repos whose CI hard-fails main-targeted PRs).
- Reviewer instructions inline in the hard rules ("BLOCK any X that fails
  safety Y; link this section and ask for the missing safeties") so agent
  reviewers act consistently, not just agent authors.
- Defaults that remove back-and-forth: if user-facing changes should always
  ship with analytics/usage tracking, say it is the default and that agents
  must not ask first.
- When a rule bans a pattern the existing code still contains, say so
  explicitly ("existing comments violate this; do not imitate them, strip the
  reference when you touch one"), or agents will treat old code as house style.

## Step 4: verify locally, then one PR per repo

- Parse every `.mdc` frontmatter (YAML) and confirm `alwaysApply: true` plus a
  non-empty description.
- Run the repo's actual gates in the worktree (build, tests, typecheck,
  formatter), the same ones your new CI job runs.
- If you delete a test framework (e.g. Jest in favor of Vitest), grep the
  whole package tree for leftover deps, fix `tsconfig` types
  (`vitest/globals` replaces `@types/jest`), and regenerate the lockfile
  preserving its lockfileVersion.
- One PR per repo. Cross-apply review comments: a comment received on repo A's
  PR that also applies to repo B gets fixed in repo B's PR without being asked.

## Step 5: review loop, then babysit

Run your normal PR-comment/CI loop until every thread is addressed and every
check is green. Resolving your own already-addressed threads is part of the
job: a PR with zero unresolved threads is the reviewable state.

Once the only thing missing is human approval, do NOT keep polling with the
LLM. Start the zero-cost shell watchdog (`scripts/babysit-prs.sh`, below) as a
background process that notifies the agent on exit:

- BEHIND base branch -> `gh api -X PUT repos/{o}/{r}/pulls/{n}/update-branch`
- human-approved + green + zero unresolved threads -> squash merge + delete
  branch (ONLY when the user explicitly delegated merge authority)
- EXITS, waking the agent, on anything needing judgment: a check failure,
  CONFLICTING state, a new unresolved thread, a failed merge attempt, or all
  PRs closed.

Human-approval detection must exclude bots and use each reviewer's LATEST
review: a stale CHANGES_REQUESTED can coexist with newer approvals. After
addressing a change request, re-request review from that person to flip it.

## Step 6: closeout

1. Tracker ticket: comment with ALL PR links as clickable URLs (never bare
   IDs), satisfy any required fields, transition to Done.
2. Remove worktrees from the main checkout (`git worktree remove ...`), delete
   the local branches, remove the /tmp dir.
3. Announce: a short message listing what changed and linking every PR.

## Pitfalls

- The default branch differs per repo (main vs master vs staging); read it
  from the repo, and separately check which branch PRs must TARGET (it can
  differ from the default).
- Amending after pushing to multiple repos: `--force-with-lease` fails with
  "stale info" when the worktree's remote-tracking ref was never fetched (a
  narrow fetch refspec). Use the explicit form
  `--force-with-lease=refs/heads/<branch>:<old-sha>`.
- Bot reviewers (CodeRabbit and similar) auto-resolve threads when a member
  replies; human threads need an explicit `resolveReviewThread` GraphQL
  mutation, and only AFTER the fix is pushed and the reply posted.
- Pre-existing error baselines (e.g. typecheck error counts): run the count
  from the correct directory before claiming a number in a PR comment.
- Husky/lint-staged pre-commit hooks fail in worktrees without node_modules;
  commit with `--no-verify` after running the real gates manually.
- Your own commit messages count as authored prose: hold them to the same
  style rules the harness enforces.

## Verification checklist (before calling it done)

- [ ] Every tool file is a pointer; zero duplicated rule text (grep one
      distinctive rule sentence across the repo, expect exactly 1 hit)
- [ ] Every `.mdc` frontmatter parses, `alwaysApply: true`
- [ ] CI workflow YAML parses and runs every gate the rules claim
- [ ] The repo does not violate its own rules (formatter clean, no banned
      configs left)
- [ ] All PRs merged or explicitly handed off; zero unresolved threads
- [ ] Tracker closed with linked PRs; worktrees cleaned up

## scripts/babysit-prs.sh

```bash
#!/bin/bash
# Babysit PRs: keep updated against base branch, squash-merge on human approval.
# Zero-cost watchdog: run in the background; it exits (waking the agent) when
# something needs judgment.
# EDIT THE PRS ARRAY, then verify with bash -n before launching.
#
# ONLY use the merge path when the user explicitly delegated merge authority.
export GH_PAGER=cat
PRS=(
  "owner repo1 111"
  "owner repo2 222"
)
POLL=180

log() { echo "[$(date '+%H:%M:%S')] $*"; }

while true; do
  open_count=0
  for spec in "${PRS[@]}"; do
    set -- $spec
    owner=$1; repo=$2; num=$3; slug="$repo#$num"

    view=$(gh pr view "$num" -R "$owner/$repo" --json state,mergeable,mergeStateStatus,reviewDecision 2>/dev/null) || { log "$slug: gh view failed, retrying next tick"; open_count=$((open_count+1)); continue; }
    state=$(jq -r .state <<<"$view")
    if [ "$state" != "OPEN" ]; then
      log "$slug: $state, skipping"
      continue
    fi
    open_count=$((open_count+1))
    mergeable=$(jq -r .mergeable <<<"$view")
    mss=$(jq -r .mergeStateStatus <<<"$view")
    decision=$(jq -r .reviewDecision <<<"$view")

    if [ "$mergeable" = "CONFLICTING" ]; then
      log "NEEDS AGENT: $slug is CONFLICTING against base"; exit 0
    fi

    checks=$(gh pr checks "$num" -R "$owner/$repo" 2>/dev/null)
    fails=$(awk -F'\t' '$2=="fail"' <<<"$checks" | wc -l | tr -d ' ')
    pending=$(awk -F'\t' '$2=="pending"' <<<"$checks" | wc -l | tr -d ' ')
    if [ "$fails" -gt 0 ]; then
      log "NEEDS AGENT: $slug has $fails failing check(s):"
      awk -F'\t' '$2=="fail"{print "  " $1 " " $4}' <<<"$checks"
      exit 0
    fi

    unresolved=$(gh api graphql -f query="{ repository(owner: \"$owner\", name: \"$repo\") { pullRequest(number: $num) { reviewThreads(first: 100) { nodes { isResolved } } } } }" --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)] | length' 2>/dev/null)
    if [ -n "$unresolved" ] && [ "$unresolved" -gt 0 ]; then
      log "NEEDS AGENT: $slug has $unresolved new unresolved review thread(s)"; exit 0
    fi

    if [ "$mss" = "BEHIND" ]; then
      msg=$(gh api -X PUT "repos/$owner/$repo/pulls/$num/update-branch" --jq '.message' 2>&1)
      log "$slug: BEHIND -> update-branch ($msg)"
      continue
    fi

    if [ "$pending" -gt 0 ]; then
      log "$slug: $pending check(s) pending, waiting"
      continue
    fi

    if [ "$decision" = "APPROVED" ]; then
      # Latest review per human reviewer must be APPROVED (excludes bots; a stale
      # CHANGES_REQUESTED overridden by a newer approval counts as approved).
      human_approved=$(gh api "repos/$owner/$repo/pulls/$num/reviews" --paginate --jq '[group_by(.user.login)[] | map(select(.state=="APPROVED" or .state=="CHANGES_REQUESTED")) | last | select(. != null)] | map(select((.user.login | test("\\[bot\\]") | not) and .state=="APPROVED")) | length' 2>/dev/null)
      if [ -n "$human_approved" ] && [ "$human_approved" -gt 0 ]; then
        if gh pr merge "$num" -R "$owner/$repo" --squash --delete-branch 2>&1 | tail -2; then
          log "MERGED: $slug (squash, branch deleted)"
        else
          log "NEEDS AGENT: $slug merge attempt failed"; exit 0
        fi
      else
        log "$slug: reviewDecision=APPROVED but no human approval yet, waiting"
      fi
    else
      log "$slug: green+current, waiting on human review (decision=$decision)"
    fi
  done

  if [ "$open_count" -eq 0 ]; then
    log "ALL PRS MERGED/CLOSED. Babysit complete."; exit 0
  fi
  sleep "$POLL"
done
```
