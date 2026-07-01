---
name: review-loop
description: Run an automatic post-work review loop with a forked sandbox subagent. Use when Codex is asked to use review-loop, autoreview, subagent review after implementation, or to review the user's request plus current git diff and then have the main agent apply actionable fixes. Supports configurable maximum review rounds, defaulting to one, and optional project review standards from .agent/review-guidelines.md.
---

# Review Loop

Use this skill after completing the user's requested work and before the final response. The review agent is a reviewer only: it inspects the user instruction and git diff in its sandbox and reports findings. The main agent owns all fixes.

## Configuration

Determine `max_rounds` before starting:

- Use an explicit user value if provided, such as `N=3`, `max review loops 3`, or equivalent wording.
- Otherwise use `REVIEW_LOOP_MAX` from the environment when it is a non-negative integer.
- Otherwise default to `1`.
- If `max_rounds` is `0`, skip the review loop and say it was disabled.

Load optional extra standards from the project root:

```bash
project_root=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
test -f "$project_root/.agent/review-guidelines.md" && cat "$project_root/.agent/review-guidelines.md"
```

If `.agent/review-guidelines.md` is missing, continue without extra standards. Do not look for this file outside the current project root unless the user explicitly gives another path.

## Review Boundary

Run the loop only after the work command is complete enough to review. A "work command" means the user's requested implementation, bug fix, refactor, generated artifact, or command-driven code change; it does not mean every shell command run during investigation.

Before each review round, collect the review input from the live worktree:

```bash
git status --short --branch
git diff --stat
git diff --check
git diff --cached --stat
git diff --cached --check
git diff HEAD --
```

If there are untracked files, include their paths from `git status --short`. For relevant new text files that are not yet tracked, include enough content for review with `sed -n`, but do not dump large generated artifacts.

If there is no git diff and no relevant untracked file, do not spawn a reviewer. Report that there was nothing to review.

## Spawn The Reviewer

Use `multi_agent_v1.spawn_agent` with `agent_type: "explorer"` when available. Set `fork_context` to `false` unless the current conversation contains essential requirements that cannot be summarized.

Always run the reviewer with the best available model and maximum reasoning depth. In the current multi-agent tool surface, set:

```text
model: "gpt-5.5"
reasoning_effort: "xhigh"
```

If future tooling exposes a stronger review-capable model or a deeper reasoning setting, use that stronger option for review instead.

Give the reviewer a self-contained prompt containing:

- The user's latest requested work and any relevant constraints.
- The current `max_rounds` and review round number.
- The optional `.agent/review-guidelines.md` content, if present.
- The full review boundary: status, staged diff, unstaged diff, and relevant untracked file snippets.
- A read-only instruction: inspect only, do not edit files, do not run destructive commands.

Ask the reviewer to return only one of these forms:

```text
NO_FINDINGS
```

or:

```text
FINDINGS
- [P1] path/to/file:line - concise issue
  Evidence: what in the diff causes the issue.
  Fix: concrete change the main agent should make.
- [P2] ...
```

Severity guidance:

- `P0`: correctness, data loss, security, or build break that must block completion.
- `P1`: likely bug, regression, missing required behavior, or invalid test/benchmark boundary.
- `P2`: maintainability, edge-case, or test coverage issue worth fixing before finalizing.
- `P3`: optional cleanup; do not loop solely for P3.

Tell the reviewer to avoid broad style commentary unless it is tied to a real project guideline or concrete risk.

## Loop Rules

For `round` from `1` through `max_rounds`:

1. Spawn one reviewer and wait for its final result.
2. Close the reviewer after receiving the result.
3. If the reviewer returns `NO_FINDINGS`, stop the loop.
4. If findings are not actionable, explain why they are rejected and stop unless there are other actionable findings.
5. Apply fixes for actionable `P0`, `P1`, and `P2` findings in the main worktree.
6. Re-run the narrow validation needed for the changed area when practical. In this repository, prefer `scripts/test.sh` for pytest validation and preserve GPU/worker-count constraints from `AGENTS.md`.
7. If more rounds remain and fixes changed the diff, start the next round with the updated diff.

Never ask the reviewer to apply fixes. Never overwrite unrelated user changes. If a fix would require changing scope or contradicting a user constraint, stop and ask the user.

## Final Response

Include a concise review-loop summary:

- Number of review rounds run.
- Whether extra guidelines were loaded.
- Reviewer result: no findings, fixed findings, rejected findings, or remaining findings.
- Validation command(s) run after fixes, or why validation was not run.

If `max_rounds` is reached while actionable findings remain, do not claim the review loop passed. Report the remaining findings and the reason they were not fixed.
