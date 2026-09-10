---
name: implement-orchestrated
description: "Implement a spec's ticket graph with parallel coder subagents, each in its own worktree: dispatch the frontier, review every ticket, send fixes back to the same coder, merge, repeat, then run the final review gate."
argument-hint: "<spec path | issue URL or number> [max parallel coders, default 3]"
disable-model-invocation: true
---

# Implement, orchestrated

You are the **orchestrator**. You write no feature code: you dispatch, verify, merge, and keep the **frontier** moving until the spec is built on one branch.

The tickets came from `/to-tickets`: a **task graph** of tracer-bullet slices, each declaring the tickets that **block** it. The frontier is every open ticket whose blockers are all closed. Work it with up to N `coder` subagents at once (N from the argument, default 3).

Talk to subagents through **context pointers** (paths, issue URLs, branch names, commit SHAs). Content they can read themselves stays out of the message.

`docs/agents/issue-tracker.md` tells you how to fetch, claim, comment on, and close a ticket. If it is missing, stop and tell the user to run `/setup-matt-pocock-skills`.


## Agent names

Installed as a plugin, the three agents are registered under the plugin's namespace: `implement-orchestrated:coder`, `implement-orchestrated:reviewer`, `implement-orchestrated:final-reviewer`. Copied by hand into `~/.claude/agents/`, they are the bare `coder`, `reviewer`, `final-reviewer`. Check the Agent tool's available types once at the start and use whichever form is registered; the rest of this skill writes the short form.

## Preconditions

Check all three before dispatching anything; on a failure, stop and tell the user what to change.

- The working tree is clean.
- The `worktree.baseRef` setting is `head` (grep `baseRef` in `~/.claude/settings.json` and `.claude/settings*.json`). Without it, coder worktrees branch from the remote default branch, so a ticket that depends on a merged ticket would build on stale code.
- Every ticket has a `Blocked by` line, and the initial frontier is non-empty. An empty frontier with open tickets means a cycle in the graph.

## Steps

### 1. Read the graph

Fetch the spec and every ticket. Build the state table: ticket, title, blocked by, status, coder, branch, worktree. Write it to `<notes>/state.md`, where `<notes>` is a directory outside the repo: `<session scratchpad>/implement-orchestrated/<feature-slug>/`, or the OS temp dir when no scratchpad is listed. Update `state.md` after every dispatch, verdict, and merge: it is your memory across compaction.

Restate the plan to the user in at most ten lines (branch, ticket count, first frontier, N), then proceed without waiting.

Done when every ticket has a status and a blocker list in `state.md`.

### 2. Prepare the branch

- On the default branch, create `feat/<feature-slug>` from HEAD; on any other branch, stay on it.
- Run the full test suite once and record the result in `state.md`: this baseline makes later failures attributable.
- With a GitHub remote, push the branch and open a **draft PR** whose body closes the spec issue and every ticket.
- Only when tickets touch areas the spec leaves unexplained, spawn one `general-purpose` **exploration subagent** with the spec pointer; it writes `<notes>/notes.md`. Every coder brief points at it, so implementers implement rather than explore.

Done when the branch exists, the baseline is recorded, and the PR (if any) is open.

### 3. Dispatch the frontier

For each frontier ticket while fewer than N coders are running: claim the ticket per the tracker doc, then spawn a `coder` with worktree isolation (`subagent_type: implement-orchestrated:coder`, `isolation: worktree`, in the background) using the coder brief below. Record the agent's id or name in `state.md`; when it reports, record its branch and worktree path.

Done when every frontier ticket is claimed and dispatched, or N coders are running.

### 4. Verify each finished ticket

When a coder reports, spawn a `reviewer` with the reviewer brief. Two verdicts:

- **Approved**: go to step 5.
- **Needs fixes**: save the findings to `<notes>/review-<NN>-r<k>.md`, then send the **same coder** the fix follow-up below. It resumes with its previous context intact, so it needs the pointer and nothing else. Review again when it reports.

After the second fix round still ends in Needs fixes, leave that ticket claimed, record it as **escalated** in `state.md`, and continue with the rest of the frontier. Tell the user at the end.

Done when the ticket is Approved or escalated.

### 5. Merge and close

In the main checkout, on the feature branch:

1. `git merge --no-ff <ticket-branch>`. On a conflict, call the Skill tool with "resolving-merge-conflicts".
2. Run the full test suite. Red means an integration problem the ticket-level review could not see: spawn a `coder` **without isolation** on the feature branch with the failing output saved to `<notes>/integration-<NN>.md` as its pointer. Merges are serial, so only one such coder runs at a time.
3. Close the ticket per the tracker doc, with the merge commit SHA in the closing comment.
4. `git worktree remove <path>` and delete the ticket branch.

Recompute the frontier and return to step 3. Done when every ticket is closed or escalated.

### 6. Final gate

1. Call the Skill tool with "code-review", fixed point = the commit the feature branch started from. Fix every finding with one `coder` without isolation on the feature branch; commit.
2. Spawn `final-reviewer` with the spec pointer and `git diff <base-commit>...HEAD`. On **With fixes**, run one more fix round with the same coder; on **No**, escalate.
3. Push. Mark the PR ready for review, or report the branch name when there is no remote.
4. Remove any remaining worktrees and ticket branches.

Report: tickets closed with their merge SHAs, the PR link or branch, and every escalated ticket with its review pointer.

## Briefs

Fill the angle brackets; send nothing else.

### Coder brief

```
Ticket: <ticket pointer>. Spec: <spec pointer>. Notes: <notes>/notes.md (read it if it exists).

You are in your own git worktree on your own branch; every command and edit stays inside it. It holds tracked files plus any local config listed in `.worktreeinclude`, so run the project's install step (for example `npm ci`) before the first test. The glossary is `CONTEXT.md` in your worktree; if it is missing, read <main checkout>/CONTEXT.md and <main checkout>/docs/agents/. Build this ticket.
```

### Reviewer brief

```
Ticket: <ticket pointer>. Spec: <spec pointer>.
Diff: `git diff <feature-branch>...<ticket-branch>`. Commits: `git log <feature-branch>..<ticket-branch> --oneline`.

Check every acceptance criterion against the diff, and run the tests the coder reports as passing rather than trusting the report. Report criteria that are missing or partial, behaviour the ticket did not ask for, implementations that look wrong, and standards problems, each with file:line. Under 400 words. End with the verdict: Approved / Needs fixes.
```

### Fix follow-up (to the same coder)

```
Review round <k> found issues: read <notes>/review-<NN>-r<k>.md. Fix them in your worktree, rerun the full suite, commit, and report as before.
```
