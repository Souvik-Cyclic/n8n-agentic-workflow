# Problem Statement — Vanguard

## User
Maintainers and reviewers of a GitHub repository where every pull request runs CI
(build + tests) before it can be merged.

## Pain point
Two chores repeat on every pull request:

1. **Reading the change** — someone has to open the diff, understand what changed, judge whether
   it's safe, and catch any secret accidentally committed.
2. **Decoding CI failures** — when the build or tests fail, someone has to open the log, scroll
   through it, and work out what broke and how to fix it.

Both are repetitive and slow the author's feedback loop. And the final call — actually approving
and merging — should stay with a human, not a bot.

## Workflow goal
Automatically review every pull request that passes CI, and diagnose every one that fails — so
maintainers get a quick, consistent read on each PR while a human still approves every merge.
AI handles the reasoning, deterministic logic handles the control, and the human keeps the decision.

## Expected output
- A GitHub **PR comment** — a review, a failure diagnosis, or the recorded human decision.
- An **email** with a one-click Approve / Request-changes form (the human-in-the-loop gate).
- A GitHub **label** (`approved`) when a human approves.
- Clean **JSON** from each AI step, which the workflow then formats and acts on.
