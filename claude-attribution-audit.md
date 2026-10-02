# claude-attribution-audit

Audits GitHub posts made under the user's own identity (`dmccoystephenson`) to find ones that were drafted by Claude but lack the `drafted by Claude on behalf of Daniel Stephenson` sign-off, classifies them by confidence, and optionally revises them — with confirmation, and without ever putting words in the user's mouth.

---

## The hard truth this skill is built around

Posts are made **under the user's own GitHub account**. GitHub's authorship metadata therefore says "Daniel Stephenson" for *both* the user's own writing and Claude's drafts. **There is no metadata field that proves an agent wrote a post.** Detection is inherently **heuristic and probabilistic**, never exact.

The sign-off convention (`drafted by Claude on behalf of Daniel Stephenson`) was only adopted on **2026-06-06**. Almost everything Claude drafted before then carries *no* machine-detectable marker. So the absence of a sign-off proves nothing on its own.

Because of this, the skill's job is to **bucket** posts by confidence and only auto-revise the high-confidence bucket. The larger contextual buckets are surfaced for the user to confirm, never edited silently.

## Non-negotiable rules (carry through every revision)

- **Never put words in the user's mouth.** No first-person assertions ("I reviewed…", "I think…").
- **Passive voice is MANDATORY, not optional — appending the sign-off is NOT sufficient on its own.** Any post containing Claude's first-person authorial voice MUST have that prose converted to passive voice or explicit Claude attribution *in addition to* the sign-off. A sign-off bolted onto a body that still says "I found the bug and I fixed it" still reads as the user's own words above the line. Convert: "I found / I ran / I decided" → "The bug was found / Tests were run / Claude decided"; "my code / my analysis" → "the code / Claude's analysis".
- Every revised post ends with the sign-off line: `drafted by Claude on behalf of Daniel Stephenson`.
- **DO NOT rewrite — these are NOT authorial voice, leave them verbatim:** quoted text and blockquotes (`>` lines); code, code fences, and test names; file/repo names that merely contain a pronoun (e.g. `my-claude-skills`); technical tokens like `I/O`; ASCII diagrams; and **domain content** — first-person that belongs to the *subject* of the post, not its author (e.g. an `artificial-consciousness` project quoting a simulated agent saying "I am…"). Converting these corrupts meaning. First-person singular narration of the author's *own actions* is the target; everything else is noise.
- **When in doubt, do not edit.** Wrongly rewriting a post the user actually wrote themselves is the worst outcome — it misattributes their genuine words to Claude.
- Applies to every surface: PR titles/descriptions, review bodies, inline review comments, issue titles/bodies, PR/issue comments, commit messages.

---

## Steps

### 1 — Preflight: auth and scope

```bash
gh auth status
gh api user --jq '.login'   # expect: dmccoystephenson
```

If not authenticated, stop and ask the user to run `! gh auth status` / authenticate.

Decide scope with the user (default: all repos). They may narrow to a single repo (`--repo owner/name` on searches) or a date window (`created:>=YYYY-MM-DD`). Scoping by repo is the most useful lever — see the tiers in Step 3.

### 2 — Size the surface (read-only recon)

Count authored items and per-repo distribution. The GitHub search API caps results at 1000 — if a count returns exactly 1000, treat it as "≥1000, capped".

**CRITICAL: `gh search issues` and `gh search prs` are DISJOINT result sets** — `gh search issues` returns issues ONLY (no PRs), despite the underlying GitHub endpoint. You MUST run BOTH and sum them; running only one silently misses ~half the surface. The `isPullRequest` JSON field from `gh search` is unreliable (often `false` for everything) — **classify type by URL** (`/pull/` vs `/issues/`), not by that field.

```bash
# Issues (issues only) and PRs (PRs only) — run BOTH, they do not overlap
gh search issues --owner dmccoystephenson --author=@me --created '>=YYYY-MM-DD' --limit 1000 --json number,repository,url,title
gh search prs    --owner dmccoystephenson --author=@me --created '>=YYYY-MM-DD' --limit 1000 --json number,repository,url,title

# Per-repo breakdown (run for each of the two result sets)
#   --jq '[.[].repository.name] | group_by(.) | map({repo:.[0],n:length}) | sort_by(.n) | reverse | .[] | "\(.n)\t\(.repo)"'
```

Use `--owner dmccoystephenson` to scope to repos the user *owns* — this automatically excludes org-owned work repos (Tier C), a clean way to drop the riskiest bucket.

### 3 — Detect markers and classify into tiers

**Strong machine markers** (high precision — almost certainly Claude). As in Step 2, run every marker search against **both** `gh search issues` and `gh search prs` — they are disjoint, and PR descriptions are where the `Generated with Claude Code` footer most often appears. Classify the combined results by URL.

```bash
# Already-compliant — these are FINE, exclude from the worklist
gh search issues --author=@me --match body "drafted by Claude on behalf" --limit 1000 --json url,title,repository
gh search prs    --author=@me --match body "drafted by Claude on behalf" --limit 1000 --json url,title,repository

# Strong marker but (likely) no sign-off → primary revision candidates
gh search issues --author=@me --match body "Generated with Claude Code" --limit 1000 --json url,title,repository
gh search prs    --author=@me --match body "Generated with Claude Code" --limit 1000 --json url,title,repository
gh search issues --author=@me --match body "Co-authored-by Claude"       --limit 1000 --json url,title,repository
gh search prs    --author=@me --match body "Co-authored-by Claude"       --limit 1000 --json url,title,repository
```

For commits (markers live in commit messages, not the search-issues surface):

```bash
gh search commits --author=@me "Co-authored-by Claude"          --limit 100 --json repository,sha,commit
gh search commits --author=@me "Generated with Claude Code"     --limit 100 --json repository,sha,commit
```

**Classify each authored repo into one of four tiers:**

| Tier | What it is | Action |
|---|---|---|
| **A — Strong-signal, no sign-off** | Has a Claude footer/trailer but not the sign-off line | Auto-revise candidate (Step 5), after confirmation |
| **B — Contextual (dev-loop / automation repos)** | Repos targeted by a `*-dev-loop` skill or other automation; very likely Claude-drafted but no marker | Surface as a list for the user to confirm; never silent-edit |
| **C — Probably human (work repos)** | The user's day-job / employer repo cluster | Exclude by default; only touch if the user explicitly asks |
| **D — Ambiguous no-marker** | Everything else with no marker and no clear context | Leave for manual eyeball; do not auto-edit |

**Derive the tier lists at runtime — do not hard-code them here** (this file is public; the specific repo names are the user's private taxonomy):

- **Tier B** — list the installed dev-loop skills and map each to its target repo:
  ```bash
  ls ~/.claude/commands/ | grep -- '-dev-loop'
  ```
  Each `*-dev-loop` skill names the repo it drives in its body; read those to build the Tier B repo set.
- **Tier C** — ask the user which repos are work/employer-owned (or infer from the per-repo breakdown in Step 2: dense, recent, non-personal-project clusters). When unsure whether a repo is Tier C, treat it as Tier C (exclude) rather than risk editing the user's genuine work posts.

### 4 — Produce the revision worklist

Build a markdown report — do **not** edit anything yet:

- **Tier A (auto-revise candidates):** table of `repo · #num · title · URL · which marker · missing sign-off?`
- **Tier B (confirm-first):** per-repo counts and links; flagged as "likely Claude, needs your confirmation".
- **Tier C / D:** counts only, with a one-line note that these are excluded by default and why.
- A headline summary: "N strong-signal items need the sign-off; M contextual items await your confirmation; exact detection is impossible for the rest."

Present this and **stop for user direction.** Ask which tiers to act on.

### 5 — Revise (only what the user approved)

Each approved item gets **two passes, not one** (a sign-off alone is insufficient — see the Non-negotiable rules):

1. **Scan the body for authorial first-person.** Use a targeted pattern for Claude narrating *its own* actions/opinions, e.g.:
   `(^|[^A-Za-z\`])(I (found|ran|made|used|asked|inferred|optimized|noticed|resolved|started|chose|decided|wrote|authored|verified|think|realized|could|had|almost)|I've|I'd|## How I|What I did|my code|my analysis|my first attempt)`
   Then **manually triage every hit** against the exclusion list in the Non-negotiable rules (repo names like `my-claude-skills`, `I/O`, code/test names, blockquotes, ASCII, domain content). Most hits are noise — only convert true authorial narration.
2. **Convert** the surviving authorial sentences to passive / explicit-Claude voice, then **append** the sign-off block:
   ```
   \n\n---\n\n_drafted by Claude on behalf of Daniel Stephenson_
   ```
   Bodies with NO authorial first-person need only the sign-off appended.

**Editing mechanics (robust against bodies with special chars/heredocs):** build the new body in a file and PATCH via the REST API with `jq --rawfile`. **Issues and PRs use DIFFERENT endpoints** (classify by URL):

Give every item its own body file, keyed on repo and number (or comment id). A single shared path such as `/tmp/newbody.md` lets a stale body from an earlier item, or from a concurrent session, be PATCHed onto the wrong post.

```bash
# per-item body file — never reuse one path across items
BODY_FILE="/tmp/claude-attribution-audit-${OWNER_REPO//\//_}-$NUM.md"     # issue or PR
# BODY_FILE="/tmp/claude-attribution-audit-${OWNER_REPO//\//_}-c$CID.md"  # comment

# issue  (url contains /issues/)
jq -n --rawfile body "$BODY_FILE" '{body:$body}' | gh api -X PATCH "repos/$OWNER_REPO/issues/$NUM" --input -
# pull request  (url contains /pull/)
jq -n --rawfile body "$BODY_FILE" '{body:$body}' | gh api -X PATCH "repos/$OWNER_REPO/pulls/$NUM"  --input -
# comment (issue OR PR comment) — by comment id
jq -n --rawfile body "$BODY_FILE" '{body:$body}' | gh api -X PATCH "repos/$OWNER_REPO/issues/comments/$CID" --input -
```

To match the user's established format, place the sign-off after a `---` separator and italicize it: `_drafted by Claude on behalf of Daniel Stephenson_`.

Revision discipline:
- **Minimal rewrite.** Convert authorial first-person and append the sign-off. Do not restructure or add claims that weren't there.
- If a post contains genuine first-person content that reads like the *user* actually wrote it (not Claude's typical output), **flag it back to the user instead of editing** — it may belong to Tier D, not A.
- Editorial plural "we/our" in technical analysis is a softer case than first-person singular "I"; convert it where the user wants full passive coverage, but flag rather than silently rewrite collaborative product-planning issues ("We should add X") whose voice is intentional.
- Commit messages can't be safely rewritten in place on shared history — list them for the user rather than rewriting published commits.
- Cache each fetched body locally so a re-run / resume doesn't re-fetch; work in batches and log every edit (URL + OK/FAIL).

### 6 — Report

Summarize: how many items were revised (with links), how many were surfaced for confirmation and await a decision, and how many were deliberately left untouched (Tiers C/D) and why. Reaffirm that detection was heuristic and the untouched buckets are not "confirmed clean" — only "not confidently Claude".

---

## Self-audit

Run this section when the skill may have drifted from reality — e.g. after the target environment changed, `gh` flags changed, or results are consistently wrong.

1. Read this skill file from top to bottom.
2. For each command, path, or assumption, verify it is still correct:
   - `gh search`/`gh pr`/`gh issue`/`gh api` commands and flags still exist and behave as described
   - The sign-off text still matches the user's stated convention
   - The Tier B dev-loop repo list and Tier C work-repo list still reflect reality (re-derive Tier B from installed `*-dev-loop` skills)
3. For each problem found, open a GitHub issue. Write the body to a uniquely named file first (e.g. `/tmp/claude-attribution-audit-self-audit-<short-slug>.md`) with this content, in passive voice and ending with the sign-off:
   ```
   **Section:** <which step or section is wrong>
   **Problem:** <what is incorrect>
   **Expected behavior:** <what it should do instead>

   ---

   _drafted by Claude on behalf of Daniel Stephenson_
   ```
   Then pass it with `--body-file` (avoid `--body "$(cat <<'EOF' …)"`, which some harnesses reject):
   ```bash
   gh issue create --repo dmccoystephenson/claude-attribution-audit \
     --title "<problem summary>" \
     --body-file /tmp/claude-attribution-audit-self-audit-<short-slug>.md
   ```
4. Report a summary: how many issues were filed, or confirm the skill is up to date.
