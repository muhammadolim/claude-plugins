---
name: review-pr
description: Reviews a project PR end-to-end — delegates finding to the code-review skill, routes findings by verdict into auto-fix vs ask-once buckets, applies fixes, commits, pushes, verifies, and posts inline + summary comments on the PR. Invoke for /review-pr or any "review the pr", "review this", "look at the diff", "find issues in PR" phrasing. Prefer this over the built-in /review and bare /code-review for project PRs.
---

# Review PR

## Input

$ARGUMENTS — optional target (a PR number, `owner/repo#N`, or a PR URL), optional effort level (`low` | `medium` | `high` | `xhigh` | `max`), and optional free-form review focus. Defaults: the PR from session context if one was created/updated/fetched this session, else the current branch's PR; effort `high`. If the user asks for `ultra`, tell them to run `/code-review ultra <N>` themselves — it is user-triggered and billed.

## Context-aware shortcuts

- The target PR may live in ANY repo touched this session, not just the current directory. If you created, updated, or fetched a PR earlier this session, that PR — with its repo slug — is the default target; use it directly instead of re-deriving from the current repo's remote.
- Skip step 1's lookups when the PR number and repo slug are already known from this session; always fetch the head SHA fresh.

## Steps

1. **Resolve the target PR and pick the mode** — capture the head SHA fresh so inline comments anchor to the reviewed commit:
   - **PR + repo slug (`<slug>`, `<N>`):**
     - $ARGUMENTS carries a PR URL or `owner/repo#N` → parse both from it.
     - A bare PR number → pair it with the session-context repo, else the current repo.
     - No PR argument → the most recent PR created/updated/fetched this session if there is one; else `gh pr view --json number --jq .number` in the current repo; else ask the user.
   - **Metadata:** `gh pr view <N> -R <slug> --json headRefOid,headRefName,baseRefName,state,url`. Pass `-R <slug>` to EVERY `gh` call in this skill — never rely on the current directory's remote.
   - **Mode** — compare `<slug>` to the session's primary working repo (`gh repo view --json nameWithOwner --jq .nameWithOwner`, from the project root the `code-review` skill reviews against — not a transient shell `cd`):
     - `<slug>` == primary repo AND `state` is `OPEN` → **full mode** (checkout, fix, verify, push).
     - Different repo, or a merged/closed PR → **review-only mode** (find + comment; no checkout/fix/verify/push). The built-in `code-review` finder reads the session's own working tree, so it can't be aimed at another repo's tree — a foreign PR is reviewable only from its diff.
   - Merged/closed PR → post a `COMMENT` review only (never `APPROVE`/`REQUEST_CHANGES`); tell the user it's already merged/closed before posting.

2. **Prepare the diff:**
   - Full mode: if the primary repo's branch ≠ `headRefName` or its HEAD ≠ `headRefOid` → `gh pr checkout <N> -R <slug>` from the primary repo root (working tree dirty → stop and surface the changes; don't stash or discard). Already on `headRefOid` → use as-is. Checkout fails → drop to review-only.
   - Review-only mode: nothing to check out; the finder reads `gh pr diff <N> -R <slug>`. Skip steps 4–8; in step 9 post every finding as an inline comment. State the mode in the summary.

3. **Find issues** — invoke the `code-review` skill via the Skill tool:
   - Full mode args: `<effort> <N> — only flag changed code` (reviews the checked-out branch in the primary repo).
   - Review-only args: `<effort> <url> — only flag changed code` (the PR URL is a cwd-independent target `gh` accepts directly).
   - Append any free-form review focus from $ARGUMENTS. Never pass `--fix` or `--comment`.
   - Sanity-check that the returned findings reference files present in the PR diff; if they clearly don't, the finder reviewed the wrong tree — stop and report, don't post.

4. **Classify each finding** (full mode; review-only skips to step 9):
   - **Auto-fix**: CONFIRMED bugs and reuse/simplification/efficiency cleanups — only when the fix stays inside the reviewed diff.
   - **Ask-once**: PLAUSIBLE bugs; altitude findings; any fix extending beyond the reviewed diff, regardless of verdict.
   - At `low`/`medium` effort the report has no verdicts — classify by the same boundary using your own judgment; when uncertain → ask-once.

5. **Apply auto-fix findings** with Edit. Don't ask.

6. **Ask-once findings** (if any): one `AskUserQuestion` call, multiSelect, findings as options. More than 4 findings → split across up to 4 questions in the same call; more than 16 → keep the 16 most severe as options and fold the rest into the summary as `Noted, not applied:`. Apply chosen fixes.

7. **Commit and push** any fixes from steps 5–6. Conventional commit (`fix:` / `refactor:`).

8. **Verify** (full mode only — review-only has no tree, skip) — run this repo's PR verification:
   - Use the verify command documented in the repo's `CLAUDE.md` / `CLAUDE.local.md`. If none is documented, detect the package manager from the lockfile and run: a `typecheck` script → else a `build` script → else `npx tsc --noEmit`.
   - Tests — diff-related only, never the full suite: if the repo uses Jest, `npx jest --changedSince=origin/<baseRefName>`. No related tests, or no test runner → skip.
   - On errors → fix, commit, push, re-run until clean.

9. **Post the review** — one API call carries all inline comments plus the summary. Use the head SHA and repo slug from step 1. Run via Bash with a heredoc:

   ```
   gh api repos/<slug>/pulls/<N>/reviews -X POST --input - <<'EOF'
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
- If code-review returns no findings, skip steps 5–7; still run step 8 (verify, full mode only) and step 9 (review with summary only).
- Review-only mode summary `body`: `Reviewed at <effort> effort (review-only — no local checkout). <count> finding(s) posted; no fixes applied or verified.`
- If the code-review invocation fails or its report is unusable, stop and report the failure in chat — don't treat it as no findings, don't post a review.
- If the batch review POST returns 422, re-check each comment's `line` against the PR diff, move unanchorable entries into the summary `body`, and retry once — don't fall back to per-finding calls.
