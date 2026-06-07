# claude-attribution-audit

Audits GitHub posts made under the user's own identity (`dmccoystephenson`) to find ones that were drafted by Claude but lack the `drafted by Claude on behalf of Daniel Stephenson` sign-off, classifies them by confidence, and optionally revises them — with confirmation, and without ever putting words in the user's mouth.

---

## The hard truth this skill is built around

Posts are made **under the user's own GitHub account**. GitHub's authorship metadata therefore says "Daniel Stephenson" for *both* the user's own writing and Claude's drafts. **There is no metadata field that proves an agent wrote a post.** Detection is inherently **heuristic and probabilistic**, never exact.

The sign-off convention (`drafted by Claude on behalf of Daniel Stephenson`) was only adopted on **2026-06-06**. Almost everything Claude drafted before then carries *no* machine-detectable marker. So the absence of a sign-off proves nothing on its own.

Because of this, the skill's job is to **bucket** posts by confidence and only auto-revise the high-confidence bucket. The larger contextual buckets are surfaced for the user to confirm, never edited silently.

## Non-negotiable rules (carry through every revision)

- **Never put words in the user's mouth.** No first-person assertions ("I reviewed…", "I think…").
- Revisions use **passive voice** and **explicit Claude attribution** ("The diff was reviewed", "Claude found…").
- Every revised post ends with the sign-off line: `drafted by Claude on behalf of Daniel Stephenson`.
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

```bash
# Totals (note: `gh search issues` returns BOTH issues and PRs unless filtered)
gh search prs    --author=@me --limit 1000 --json url --jq 'length'
gh search issues --author=@me --limit 1000 --json url --jq 'length'

# Per-repo breakdown of authored PRs (most recent 1000)
gh search prs --author=@me --limit 1000 --json repository \
  --jq '[.[].repository.name] | group_by(.) | map({repo: .[0], n: length}) | sort_by(.n) | reverse | .[] | "\(.n)\t\(.repo)"'
```

### 3 — Detect markers and classify into tiers

**Strong machine markers** (high precision — almost certainly Claude):

```bash
# Already-compliant — these are FINE, exclude from the worklist
gh search issues --author=@me --match body "drafted by Claude on behalf" --limit 1000 --json url,title,repository

# Strong marker but (likely) no sign-off → primary revision candidates
gh search issues --author=@me --match body "Generated with Claude Code" --limit 1000 --json url,title,repository
gh search issues --author=@me --match body "Co-authored-by Claude"       --limit 1000 --json url,title,repository
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

For each approved item, fetch the current body, rewrite per the **Non-negotiable rules** above, and update:

```bash
# PR / issue body
gh pr  edit  <num> --repo <repo> --body "$(cat <<'EOF'
<rewritten body — passive voice, Claude attributed>

drafted by Claude on behalf of Daniel Stephenson
EOF
)"
gh issue edit <num> --repo <repo> --body "..."   # same shape

# A comment (issue or PR) — edit via the comments API by comment id
gh api -X PATCH repos/<owner>/<repo>/issues/comments/<comment_id> -f body="..."
```

Revision discipline:
- **Minimal rewrite.** Append the sign-off and convert first-person assertions to passive/attributed voice. Do not restructure or add claims that weren't there.
- If a post contains genuine first-person content that reads like the *user* actually wrote it (not Claude's typical output), **flag it back to the user instead of editing** — it may belong to Tier D, not A.
- Commit messages can't be safely rewritten in place on shared history — list them for the user rather than rewriting published commits.
- Work in batches; report each batch's edits with links so the user can spot-check.

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
3. For each problem found, open a GitHub issue:
   ```bash
   gh issue create --repo dmccoystephenson/claude-attribution-audit \
     --title "<problem summary>" \
     --body "$(cat <<'EOF'
   **Section:** <which step or section is wrong>
   **Problem:** <what is incorrect>
   **Expected behavior:** <what it should do instead>
   EOF
   )"
   ```
4. Report a summary: how many issues were filed, or confirm the skill is up to date.
