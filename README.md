# claude-attribution-audit

A Claude skill that audits GitHub posts made under the user's own identity to find ones that were drafted by Claude but lack the `drafted by Claude on behalf of Daniel Stephenson` sign-off, classifies them by confidence, and optionally revises them — with confirmation, and without ever putting words in the user's mouth.

The skill itself is [`claude-attribution-audit.md`](./claude-attribution-audit.md). Install it by placing (or symlinking) that file into `~/.claude/commands/`, then invoke it with `/claude-attribution-audit`.

## How it works

GitHub's authorship metadata cannot distinguish the user's own writing from Claude's drafts, so detection is heuristic. The skill buckets authored posts into four tiers:

| Tier | What it is | Default action |
|---|---|---|
| **A — Strong-signal, no sign-off** | Has a Claude footer/trailer but not the sign-off line | Auto-revise candidate, after confirmation |
| **B — Contextual (dev-loop / automation repos)** | Very likely Claude-drafted but no marker | Surfaced for the user to confirm; never silent-edited |
| **C — Probably human (work repos)** | The user's day-job / employer repos | Excluded by default; only touched if the user explicitly asks |
| **D — Ambiguous no-marker** | Everything else | Left for manual review; not auto-edited |

Tier B and Tier C membership is derived at runtime, not hard-coded here. Tier C wins over Tier A: a work-repo post stays excluded even when it carries a Claude marker. Recon and classification are read-only; nothing is edited until the user approves specific tiers after reviewing the worklist. Approved revisions convert authorial first-person prose to passive voice and append the sign-off. When in doubt, a post is left untouched. See the skill file for the full steps and rules.

## License

This project is licensed under the **Stephenson Software Non-Commercial License (Stephenson-NC)**.

**License:** Stephenson-NC © 2025 Daniel McCoy Stephenson  
See [LICENSE.md](./LICENSE.md) for the full legal text, or the canonical repository at <https://github.com/Stephenson-Software/stephenson-nc-license> for details and commercial-licensing inquiries.
