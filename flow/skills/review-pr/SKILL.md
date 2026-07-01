---
name: review-pr
description: Reviews a project PR end-to-end — delegates finding to the code-review skill, routes findings by verdict into auto-fix vs ask-once buckets, applies fixes, commits, pushes, verifies, and posts inline + summary comments on the PR. Invoke for /review-pr or any "review the pr", "review this", "look at the diff", "find issues in PR" phrasing. Prefer this over the built-in /review and bare /code-review for project PRs.
---

# Review PR

## Input

$ARGUMENTS — optional PR number, optional effort level (`low` | `medium` | `high` | `xhigh` | `max`), and optional free-form review focus. Defaults: current branch's PR, `high`. If the user asks for `ultra`, tell them to run `/code-review ultra <N>` themselves — it is user-triggered and billed.

## Context-aware shortcuts

- Skip step 1's lookups if the PR number or repo slug is already known from this session; always fetch the head SHA fresh.

## Steps

1. **Resolve PR metadata** — capture the head SHA before any fixes land so inline comments anchor to the reviewed commit:
   - PR number: argument → use it; none → `gh pr view --json number --jq .number`; still none → ask the user.
   - Head SHA, head branch, base branch: `gh pr view <N> --json headRefOid,headRefName,baseRefName`
   - Repo slug: `gh repo view --json nameWithOwner --jq .nameWithOwner`

2. **Sync to the PR head** if the current branch is not `headRefName` or local HEAD is not `headRefOid`:
   - Working tree dirty → stop and surface the uncommitted changes; don't stash or discard.
   - Clean → `gh pr checkout <N>`.
   - Checkout fails (branch held by another worktree slot, fetch error, …) → surface the error and stop.

3. **Find issues** — invoke the `code-review` skill via the Skill tool:
   - args: `<effort> <N> — only flag changed code`
   - Append any free-form review focus from $ARGUMENTS to the args.
   - Never pass `--fix` or `--comment`.

4. **Classify each finding**:
   - **Auto-fix**: CONFIRMED bugs and reuse/simplification/efficiency cleanups — only when the fix stays inside the reviewed diff.
   - **Ask-once**: PLAUSIBLE bugs; altitude findings; any fix extending beyond the reviewed diff, regardless of verdict.
   - At `low`/`medium` effort the report has no verdicts — classify by the same boundary using your own judgment; when uncertain → ask-once.

5. **Apply auto-fix findings** with Edit. Don't ask.

6. **Ask-once findings** (if any): one `AskUserQuestion` call, multiSelect, findings as options. More than 4 findings → split across up to 4 questions in the same call; more than 16 → keep the 16 most severe as options and fold the rest into the summary as `Noted, not applied:`. Apply chosen fixes.

7. **Commit and push** any fixes from steps 5–6. Conventional commit (`fix:` / `refactor:`).

8. **Verify** (always) — run this repo's PR verification:
   - Use the verify command documented in the repo's `CLAUDE.md` / `CLAUDE.local.md`. If none is documented, detect the package manager from the lockfile and run: a `typecheck` script → else a `build` script → else `npx tsc --noEmit`.
   - Tests — diff-related only, never the full suite: if the repo uses Jest, `npx jest --changedSince=origin/<baseRefName>`. No related tests, or no test runner → skip.
   - On errors → fix, commit, push, re-run until clean.

9. **Post the review** — one API call carries all inline comments plus the summary. Use the head SHA and repo slug from step 1. Run via Bash with a heredoc:

   ```
   gh api repos/<owner/repo>/pulls/<N>/reviews -X POST --input - <<'EOF'
   {
     "commit_id": "<headSha>",
     "event": "COMMENT",
     "body": "<summary>",
     "comments": [
       {
         "path": "<finding.file>",
         "line": <finding.line>,
         "side": "RIGHT",
         "body": "<finding summary + concrete failure scenario>"
       }
     ]
   }
   EOF
   ```

   - One entry per finding — auto-fixed, user-approved, or declined; prefix declined ones with `Noted, not applied:`.
   - Escape newlines as `\n` inside JSON string values; closing `EOF` at column 0.
   - Summary `body`, findings:
     ```
     Reviewed at <effort> effort. Applied: <X> auto-fix(es), <Y> approved fix(es); declined: <Z>. Verify clean.
     ```
   - Summary `body`, no findings — omit `comments` entirely:
     ```
     Reviewed at <effort> effort — no issues. Covered:
     - <angle/category 1 from the code-review report>
     - <angle/category 2>
     - ...
     ```
   - List only angles the code-review report itself names; if it names none, post the first line without the `Covered:` list.

## Notes

- Extract JSON fields with `gh api ... --jq '.field'`, not `| python`/`| jq`.
- The code-review skill only reads and reports — keep all PR writes (comments, commits, pushes) in the main context.
- If code-review returns no findings, skip steps 5–7; still run step 8 (verify) and step 9 (review with summary only).
- If the code-review invocation fails or its report is unusable, stop and report the failure in chat — don't treat it as no findings, don't post a review.
- If the batch review POST returns 422, re-check each comment's `line` against the PR diff, move unanchorable entries into the summary `body`, and retry once — don't fall back to per-finding calls.
