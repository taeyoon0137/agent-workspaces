# Installation Guide

> Agent instruction. Read this whole file before acting, then execute the steps in order. Do not reinterpret or skip steps. `README.md` is for humans and MUST NOT be read.

Goal: install this repository as `~/.agents` — the shared policy (`AGENTS.md`), shared skills (`skills/`), the `ctx` tool (`bin/ctx.sh`), and one small bootstrap adapter per agent host — without losing any instruction the user already has.

Conventions: `MUST` is mandatory. "Ask" means the environment's native structured-question function (Codex/Claude/Gemini/Grok request-user-input, AskUserQuestion, or equivalent); if none is callable, say so in one line and ask as a numbered plain-text list in the same turn. Use `~` for the user's home. Never print secrets, tokens, or `auth.json` contents. Never delete a user file; move it to the backup folder instead.

## Step 0 — Preconditions

1. Confirm `git` is available and `~` is writable.
2. Set `TS=$(date +%Y%m%d-%H%M%S)` and `BACKUP=~/.agents-install-backup-$TS`. Every pre-existing file this guide changes or moves MUST first be copied into `$BACKUP` preserving its relative path. Report `$BACKUP` in the final summary.
3. If `~/.agents/AGENTS.md` already exists and `~/.agents/.git` points at this repository, this is an **update**, not an install: skip to Step 4 and use `git pull --ff-only` instead of clone.

## Step 1 — Ask the installation mode

Ask exactly one question with two choices:

- `Automatic` — proceed without further questions, deciding every promotion and conflict as specified below, and report all decisions at the end.
- `Manual` — ask the additional questions in Steps 2, 4 and 5.

## Step 2 — Discover existing instruction files

Scan these locations (only these, read-only):

| Host | Files to look for |
|---|---|
| Shared | `~/.agents/AGENTS.md`, `~/.agents/skills/*/SKILL.md`, `~/AGENTS.md`, `~/CLAUDE.md` |
| Claude Code | `~/.claude/CLAUDE.md`, `~/.claude/skills/*/SKILL.md`, `~/.claude/settings.json` (only the `hooks` and `plugins` keys) |
| Codex | `~/.codex/AGENTS.md`, `~/.codex/skills/*/SKILL.md`, `~/.codex/config.toml` (only instruction-related keys) |
| Gemini CLI | `~/.gemini/GEMINI.md`, `~/.gemini/commands/*` |
| Antigravity | `~/.gemini/config/GEMINI.md`, `~/.antigravity*/` instruction files if present |
| Grok | `~/.grok/AGENTS.md`, `~/.grok/GROK.md` |
| Copilot | `~/.copilot/copilot-instructions.md`, `~/.copilot/skills/*/SKILL.md` |
| Cursor / other | `~/.cursor/rules/*`, `~/.config/opencode/AGENTS.md` |

For each file found, record: path, size, whether it is a 4-line bootstrap adapter that already points at `~/.agents/AGENTS.md` (then it needs no decision), and a one-line summary of its unique content. Symlinks are recorded with their target.

Report the list to the user in one message. This report needs no reply.

Then classify every non-adapter instruction block:

- **Automatic**: a rule that appears in two or more host files (same intent, not necessarily same wording) is promoted to global. A rule found in only one host file is promoted to global when it is host-independent (language, register, safety, workflow, coding style) and kept host-local when it names host-specific tools, commands, models, or paths.
- **Manual**: for each file, ask one question: `Promote to global`, `Keep as host-local rule`, `Skip (archive only)`. Tell the user that conflicts will be re-adjusted in Step 5.

For each promoted block decide its destination: an inline `AGENTS.md` bullet when the rule must apply to every task from the first turn (safety, permission, language, register, output style, workflow gates), otherwise `~/.agents/skills/<topic>/SKILL.md` with a one-line routing entry in `AGENTS.md` (procedures, tool-specific mechanics, anything read only on a trigger). Prefer the skill when in doubt; `AGENTS.md` is loaded on every session and must stay small. Host-local content stays in that host's adapter under a `## Host-local rules` heading.

## Step 3 — Clone

```bash
git clone https://github.com/taeyoon0137/agent-workspaces.git /tmp/agents-install-$TS
```

Clone into the temporary directory only; never clone directly onto `~/.agents`. Verify the clone contains `AGENTS.md`, `bin/ctx.sh`, `skills/workspace-context/SKILL.md`.

## Step 4 — Install into `~/.agents`

1. If `~/.agents` does not exist: `mv /tmp/agents-install-$TS ~/.agents`.
2. If `~/.agents` exists: copy it to `$BACKUP/.agents` first, then merge:
   - Repository files (`AGENTS.md`, `skills/<name>/` for names present in the clone, `bin/ctx.sh`, `INSTALLATION.md`, `README.md`, `.gitignore`) are taken from the clone.
   - Files that exist only locally (`workspaces/`, `.skill-lock.json`, extra `skills/<name>/`, anything else) are kept.
   - Keep the clone's `.git` so future updates are `git pull`.
   - **Manual**: before merging, show the list of files that will be replaced and ask `Proceed` / `Abort`. **Automatic**: proceed and list them in the summary.
3. Apply Step 2 promotions as decided there: a new skill plus its routing line in `AGENTS.md`, or an inline `AGENTS.md` bullet with a new stable rule id. Do not rewrite or reorder existing `AGENTS.md` rules.
4. `chmod +x ~/.agents/bin/ctx.sh`; run `sh ~/.agents/bin/ctx.sh help` to confirm it executes, then `cd ~/.agents && sh bin/ctx.sh init` to create its own workspace (Step 8 verifies it).
5. Report every replaced or merged file to the user (no reply needed).

## Step 5 — Reconcile host files and resolve conflicts

For each host file found in Step 2 that is not already an adapter:

1. Compare its rules with `~/.agents/AGENTS.md` and the shared skills. A **conflict** is a local rule that contradicts a shared rule (different language, different register, permits something the shared policy forbids, or vice versa).
2. **Manual**: for each conflict ask one question with four choices: `Always prefer this guide` (shared policy wins now and for future updates), `Prefer this guide this time` (shared wins now; the local rule is kept in the adapter as a commented note for later review), `Prefer the local file this time` (local wins now; recorded as a deviation), `Always prefer the local file` (local wins; recorded as a permanent host-local override in the adapter).
3. **Automatic**: shared policy wins; the overridden local rule is preserved verbatim in the adapter under `## Host-local rules (overridden, review)`.
4. Record every decision as `decision` records in the workspace of `~/.agents` itself: `cd ~/.agents && sh bin/ctx.sh add decision "<summary>" -t install`. This is where a future run finds the user's earlier choices instead of asking again.

## Step 6 — Write the adapters

Replace (after backup) or create each host's global instruction file with the 4-line adapter below, appending any host-local rules decided above under `## Host-local rules`. Use the host's own name in lines 3 and 4.

```markdown
# <Host> Bootstrap

- Before every task, read and follow `~/.agents/AGENTS.md` as the canonical shared policy.
- Load shared skills from their canonical directory at `~/.agents/skills` whenever the shared policy or the current task requires them.
- Use <Host>-native tools and subagents when <Host> is the active environment, subject to the shared environment, coordination, permission, and user-control policies.
- Keep <Host>-only bootstrap guidance in this file. Do not duplicate shared policy or shared skill content here.
```

| Host | Adapter path |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md`; also `~/CLAUDE.md` as a relative symlink to `.claude/CLAUDE.md` if `~/CLAUDE.md` did not exist as a regular file |
| Codex | `~/.codex/AGENTS.md` |
| Gemini CLI | `~/.gemini/GEMINI.md` |
| Antigravity | `~/.gemini/config/GEMINI.md` |
| Grok | `~/.grok/AGENTS.md` |
| Copilot CLI | `~/.copilot/copilot-instructions.md` (create only if `~/.copilot` exists) |

Create an adapter only for hosts whose home directory exists. Do not create host directories.

## Step 7 — Optional native plugin registration

If the `claude` or `codex` CLI is on `PATH`, run the matching commands; otherwise skip and list them in the summary for the user to run later. **Manual**: ask `Register now` / `Skip` first. The shared instructions already apply without this step; registration only adds slash commands and hooks.

```bash
claude plugin marketplace add ayghri/i-have-adhd && claude plugin install i-have-adhd@i-have-adhd
claude plugin marketplace add DietrichGebert/ponytail && claude plugin install ponytail@ponytail
codex plugin marketplace add ayghri/i-have-adhd --ref main && codex plugin add i-have-adhd@i-have-adhd
codex plugin marketplace add DietrichGebert/ponytail && codex plugin add ponytail@ponytail
```

## Step 8 — Verify and report

1. `sh ~/.agents/bin/ctx.sh check` in `~/.agents` passes.
2. Every adapter written in Step 6 contains the line referencing `~/.agents/AGENTS.md`.
3. `~/.agents/AGENTS.md` still contains every rule id it had in the clone (no rule was dropped by the merge).
4. Report, in this order: installation mode; `$BACKUP` path; files found in Step 2 and what happened to each (promoted / host-local / archived); files replaced or merged in Step 4; conflict decisions from Step 5; adapters written; plugin commands run or left for the user; anything that could not be done and why. End with one next action: start a new session in any host and confirm it reads `~/.agents/AGENTS.md`.

## Re-running this guide

Running it again is safe: Step 0 detects the existing installation and switches to update mode, Step 2 finds only adapters (no decisions needed), and earlier conflict decisions are read from the `~/.agents` workspace records before any question is asked.
