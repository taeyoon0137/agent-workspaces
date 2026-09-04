---
name: workspace-skill-create
description: Create or update a workspace-local skill under ~/.agents/workspaces/<id>/skills/ when the user asks for one or accepts a proposal; load only for that creation or update, never for ordinary work.
---

# Workspace Skill Create

A workspace-local skill captures a procedure that recurs inside one workspace so future sessions of that workspace follow it instead of re-deriving it. It is invisible to every other workspace and to every host's skill catalog; discovery is only through `CONTEXT.md` `## Skills` ([WSC-020]–[WSC-024] in `workspace-context`).

## When

- [WSKC-001] Create only after the user asked for the skill or accepted a proposal made under [WSC-023]. A proposal states the procedure name, the evidence that it recurred (record ids or the steps just executed), and the trigger sentence; it uses the user-action flow and waits.
- [WSKC-002] Do not create a local skill for one-off work, for anything already covered by a global skill or repository instructions, or for state that belongs in records.

## Layout

```
~/.agents/workspaces/<workspace-id>/skills/<name>/
└── SKILL.md
```

```markdown
---
name: <name>                      # lowercase, digits, hyphens; equals the folder name
description: <one sentence: what it does and the exact trigger, "Use when …">
---

# <Title>

<one line: goal and the state it assumes>

## Steps
1. <exact command or action> → verify: <observable check>
2. …

## Pitfalls
- <known failure and its fix>
```

## Rules

- [WSKC-003] Keep the skill procedural and short: numbered steps with a verification per step, exact commands and paths, known pitfalls. Target under 2,000 characters; split into two skills rather than exceed 4,000. No narrative, no restated policy, no secrets (reference the env var or the masked source instead).
- [WSKC-004] After writing the file: add `- <name> — load when <trigger>` under `## Skills` in `CONTEXT.md` (respecting its caps), add a `decision` record stating that the skill exists and why (`ctx add decision "Local skill <name> created" -t skill`), and run `ctx check`.
- [WSKC-005] Updating an existing local skill edits the file in place (procedures are not versioned by supersession) and adds a one-line `checkpoint` record naming what changed. Removing one deletes the folder, removes its `## Skills` line, and records a `decision`.
- [WSKC-006] Never copy a local skill into the repository, `~/.agents/skills/`, or another workspace as part of this skill. Promotion is a separate user decision ([WSC-024]).
