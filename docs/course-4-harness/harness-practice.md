# Harness Practice

Theory done; now harden a real agent. The worked example is a coding agent on a web app with a Postgres database, run through Claude Code, but every step translates to any harness — including the seventy-line loop from [build the loop](../track-harness/build-the-loop.md). Work the rings in order.


> The audit half of this session is packaged as a skill: install [the course plugin](../../plugins/README.md) and run `/agents-course:harness-audit` before you harden by hand.

## Step one: build the walls

Before the first prompt, decide what the agent cannot reach. Give it a worktree instead of your checkout, and a database user that can only read:

```bash
git worktree add ../agent-workspace -b agent/task-one
```

```sql
CREATE USER agent_ro WITH PASSWORD 'rotate-me';
GRANT CONNECT ON DATABASE app TO agent_ro;
GRANT USAGE ON SCHEMA public TO agent_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO agent_ro;
```

If the agent needs to install packages or hit APIs, run it in a container with a network allowlist (your package registry and model API, nothing else). The test of this step: imagine the worst session possible and check the blast radius is a deleted worktree.

## Step two: write the guides

One AGENTS.md at the repo root, short enough to be read every session:

```markdown
- App code in src/, tests in tests/, run everything from the repo root.
- `make check` must pass before you call any task done.
- Never hand-edit files in migrations/ — generate them with `make migration`.
- Your database user is read-only. Ask before proposing any schema change.
```

Every line states a rule the agent can act on. No philosophy, no history.

## Step three: wire the sensors

Make the deterministic checks one command, then have the harness run it after every change rather than hoping the model remembers:

```make
check:
	ruff check .
	mypy src
	pytest -q
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "make check" }
        ]
      }
    ]
  }
}
```

Failures land in the transcript immediately, while the mistake is one edit old.

## Step four: strip the permissions

Least privilege, then approvals only for the irreversible:

```json
{
  "permissions": {
    "allow": ["Read", "Grep", "Glob", "Bash(make check)", "Bash(git diff:*)", "Bash(git status)"],
    "ask": ["Bash(git commit:*)", "Bash(git push:*)"],
    "deny": ["WebFetch", "Bash(rm:*)"]
  }
}
```

People approve almost every prompt they see, so a short ask-list is the only kind that gets read. If you find yourself approving something daily without thinking, move it to allow — or better, make it impossible and remove the prompt.

## Step five: add one hook

Pick the rule from AGENTS.md whose violation hurts most and enforce it:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | grep -q 'migrations/' && { echo 'Blocked: generate migrations with make migration' >&2; exit 2; } || exit 0"
          }
        ]
      }
    ]
  }
}
```

One hook, well chosen, beats ten that fire constantly.

## The quarterly review

Every harness piece is a bet that the model cannot do something, and the bets expire — Anthropic deleted a whole scaffolding component (context resets for premature wrap-ups) after one model upgrade made it dead weight. So put a recurring entry in the calendar. For each hook, guide line and denied permission, write the sentence "this exists because the model cannot X", then test X against the current model. Delete what no longer earns its keep; a harness that only grows is compensating for a model that no longer exists.

## Tips

- Add rules to AGENTS.md only after the agent gets something wrong twice. Speculative rules rot.
- Log blocked hook firings. A hook that never fires is a candidate for deletion at review.
- Tool descriptions are guides too — see [tools and MCP](../track-harness/tools-and-mcp.md) before adding a tool.

## Prove it to yourself

1. Run the same task with and without AGENTS.md in a fresh session each time. Compare the transcripts.
2. Count the permission prompts in one working day and how many you approved. Compute your own approval rate.
3. Disable one harness component, rerun a task it was meant to protect, and see whether anything actually degrades.

Next: sensors deserve a module of their own — [two kinds of checks](../course-5-evals/two-kinds-of-checks.md).
