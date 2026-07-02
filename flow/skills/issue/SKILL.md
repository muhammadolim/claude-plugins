---
name: issue
description: >
  Create a GitHub issue in the current repo (or a mapped sibling repo). Invoke
  when the user asks to create, file, open, or log an issue, ticket, bug report,
  or feature request — even if they just describe the problem and say "create an
  issue for this". Picks the right issue type (bug, feature, chore, refactor).
---

# Issue

Create a well-formed GitHub issue and open it in the right repository.

## Input

$ARGUMENTS

If no argument is given, derive the issue from the current conversation context.

---

## Step 1 — Detect repo

Default to the current repo:

```bash
gh repo view --json nameWithOwner --jq .nameWithOwner
```

If the input points at a sibling repo (e.g. "file this on the frontend", "open a backend issue") **and** this repo's `CLAUDE.md` defines an issue repo map, resolve the target from that map. No map, or no cross-repo signal → current repo.

---

## Step 2 — Classify

Pick one type. The type drives the title prefix and body template.

| Type | When to use |
|---|---|
| `bug` | Something is broken, wrong, or behaving unexpectedly |
| `feat` | New capability that doesn't exist yet |
| `chore` | Cleanup, removal, migration, dependency update, config change |
| `refactor` | Restructuring existing code without changing behavior |

---

## Step 3 — Draft title and body

### Golden rule: describe the problem, not the fix

The issue body is for **motivation and information** — what's broken, why it matters, where it shows up. The developer who picks it up has more codebase context than you do; they will pick the right fix. Prescribing a solution from the outside is often subtly wrong (different field names, missed edge cases, schema mismatches) and wastes the dev's time disproving your suggestion before they can do the real work.

**Do** include:
- What the user-visible / API-visible behavior is
- Why it's wrong (privacy leak, wrong totals, broken UX)
- Pointers to where the bug lives (file paths, endpoint names, function names) — *as a starting place to look*, not as a recipe

**Do not** include:
- Step-by-step "apply X helper", "extract Y", "refactor Z"
- Specific code shapes, helper names, or refactor plans
- Confident claims about which lines are wrong unless you've read them in this session

If the user explicitly dictates an approach ("we want to do it this way"), include it under **Proposed approach** and attribute it to them. Otherwise, omit that section.

### Title format
```
<type>(<scope>): <short imperative description>
```
- `scope` = product area (e.g. `contacts`, `auth`, `orders`, `schema`) — omit if cross-cutting
- Max 72 characters total
- Imperative mood: "remove X", "add Y", "fix Z" — not "removes" or "removed"

### Body templates

**Bug:**
```markdown
## Description
<1-2 sentences: what is wrong and where, in user-visible / API-visible terms>

## Steps to reproduce
1. …
2. …

## Expected behavior
<what should happen>

## Actual behavior
<what actually happens — be specific about the impact: data leak, wrong total, blank screen, etc.>

## Where to look
<file paths, endpoints, function names that are likely involved — only as starting points. Omit if you don't have real pointers.>
```

**Feat / Chore / Refactor:**
```markdown
## Summary
<1-2 sentences: what capability or change is needed>

## Motivation
<the problem this solves or the goal it achieves — the "why">

## Acceptance criteria
- [ ] <user-visible / API-visible outcomes, not implementation steps>
- [ ] …
```

Keep the body concise. The developer should be able to act on it without back-and-forth.
Don't pad sections you don't have real information for — omit them rather than writing "N/A".
Acceptance criteria describe outcomes ("private calls don't appear for other team members"), not steps ("call buildXFilter in service Y").

---

## Step 4 — Assignee

Default: leave unassigned. Assign only when the user explicitly names someone → add `--assignee <login>`.

Resolve names against an assignee roster in this repo's `CLAUDE.md` / `CLAUDE.local.md` if one is defined; otherwise use the name as the login.

---

## Step 5 — Pick labels

```bash
gh label list --repo <repo> --limit 100
```

Pick existing labels that clearly apply: type (`bug`, `enhancement`, etc.), area/scope, and any other relevant ones. Pass each as `--label <name>`.

If no existing label fits but the concern is reusable, create one first:

```bash
gh label create "<name>" --repo <repo> --color <hex> --description "<purpose>"
```

Naming: lowercase kebab-case, or `prefix:value` for groups (`area:auth`, `priority:high`). No emoji.

Draw from standard names: `bug`, `enhancement`, `documentation`, `refactor`, `chore`, `performance`, `security`, `accessibility`, `dependencies`, `priority:{low,medium,high,critical}`, `area:<scope>`.

Skip if nothing genuinely fits. Don't create one-off labels.

---

## Step 6 — Create the issue

```bash
gh issue create \
  --repo <repo> \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)" \
  --assignee <login> \
  [--label <label> ...]
```

---

## Step 7 — Report

Output:
- The issue URL (single line, so the user can click it)
- One sentence summarizing what was filed

Do not print the full body back — the user just wrote it with you.
