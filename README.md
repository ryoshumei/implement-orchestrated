# implement-orchestrated

An orchestrator-subagent `/implement` for [Claude Code](https://code.claude.com), built on top of [Matt Pocock's skills](https://github.com/mattpocock/skills).

Matt's flow is `/grill-with-docs → /to-spec → /to-tickets → /implement`, where `/implement` builds one ticket per session. This plugin keeps the first three steps and replaces the last one with a single command that works the whole ticket graph:

- dispatches every unblocked ticket (the **frontier**) to a `coder` subagent, each in its own git worktree, up to N in parallel
- reviews every finished ticket with a `reviewer` subagent
- sends review findings back to the **same coder**, which resumes with its previous context intact
- merges, closes the ticket, recomputes the frontier, repeats
- finishes with Matt's `code-review` skill and a `final-reviewer` pass, then pushes and marks the PR ready

It uses only documented Claude Code features: the Agent tool with `isolation: worktree`, `SendMessage` to resume a subagent, the `worktree.baseRef` setting, and `.worktreeinclude`.

## Requirements

- Claude Code 2.1.263 or newer
- The `mattpocock-skills` plugin, with `/setup-matt-pocock-skills` run once in the repo (the orchestrator reads `docs/agents/issue-tracker.md` to fetch, claim and close tickets)
- Tickets produced by `/to-tickets`, each with a `Blocked by` line

## Install

As a plugin:

```bash
claude plugin marketplace add ryoshumei/implement-orchestrated
claude plugin install implement-orchestrated@ryoshumei
```

Or copy the files by hand:

```bash
git clone https://github.com/ryoshumei/implement-orchestrated
cp -r implement-orchestrated/skills/implement-orchestrated ~/.claude/skills/
cp implement-orchestrated/agents/*.md ~/.claude/agents/
```

Installed as a plugin, the agents are namespaced (`implement-orchestrated:coder` etc.); copied by hand they are the bare `coder` / `reviewer` / `final-reviewer`. The orchestrator handles both. `claude plugin details implement-orchestrated@ryoshumei` should list **Agents (3)**; the manifest relies on the default `agents/` directory because an explicit `agents` array in `plugin.json` is not loaded by Claude Code 2.1.267 even though the reference documents it.

Then set `worktree.baseRef` to `head` in `~/.claude/settings.json`:

```json
{
  "worktree": {
    "baseRef": "head"
  }
}
```

Why: Claude Code's default base for a new worktree is the remote default branch (`origin/main`), but the orchestrator merges each finished ticket into your feature branch and then dispatches the tickets that depend on it. With the default, ticket 2's worktree would branch from `origin/main` and never see ticket 1's merged code, so the coder builds on stale code (or re-implements ticket 1 and conflicts at merge). With `head`, every worktree branches from the current tip of the feature branch.

Claude cannot edit `~/.claude/settings.json` for you (the permission classifier refuses), so run this yourself once:

```bash
python3 -c "import json;p='$HOME/.claude/settings.json';d=json.load(open(p));d.setdefault('worktree',{})['baseRef']='head';json.dump(d,open(p,'w'),ensure_ascii=False,indent=2)"
```

If your repo keeps `CONTEXT.md`, `docs/agents/` or `.env` files gitignored, list them in a `.worktreeinclude` file at the repo root (gitignore syntax) so Claude Code copies them into each worktree:

```text
CONTEXT.md
docs/agents/
docs/adr/
CLAUDE.local.md
```

List specific paths. Do not list `.claude/` wholesale: `.claude/worktrees/` holds your other worktrees and would be copied into every new one (gigabytes, recursively). Name the subfolders you need instead (`.claude/hooks/`, `.claude/skills/`, `.claude/settings.json`). The Agent tool's worktree isolation does not run your `PostToolUse` hooks, so `.worktreeinclude` is the only thing that copies untracked files into a coder's worktree.

## Usage

Start a fresh session in the repo, on a clean working tree:

```
/implement-orchestrated <spec path | issue URL or number> [max parallel coders, default 3]
```

Example:

```
/implement-orchestrated screenshot-source 3
```

The orchestrator restates the plan (branch, ticket count, first frontier, N) and proceeds without waiting. It keeps a `state.md` outside the repo so it survives context compaction, escalates a ticket that fails review twice, and reports every merge SHA, the PR link, and any escalations at the end.

## What is in the box

| Path | What it is |
| --- | --- |
| `skills/implement-orchestrated/SKILL.md` | The orchestrator. User-invoked only. |
| `agents/coder.md` | Implementer: pre-agreed seams, red → green via Matt's `tdd` skill, reports by context pointers. Pinned to Opus with max effort; change `model`/`effort` to taste. |
| `agents/reviewer.md` | Per-ticket spec and quality review; verifies the coder's claims by running things. |
| `agents/final-reviewer.md` | Whole-branch review at the end. Inherits the session model. |

`coder` has no `isolation` in its frontmatter on purpose: the orchestrator passes `isolation: worktree` per call, so the same agent still works for ordinary bug fixes.

## How it differs from Matt's `implement-spec`

Matt's repo carries an unshipped `implement-spec` draft with the same shape (frontier, implementer subagents in worktrees, merge, code-review at the end). This plugin fills in the parts that draft leaves open: the review-fix loop that resumes the same coder, the briefs, the state file, escalation rules, and the worktree base-ref and include settings that make dependent tickets build on merged code.

## License

MIT
