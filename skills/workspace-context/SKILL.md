---
name: workspace-context
description: Read, write, and maintain the repository-external workspace records (CONTEXT.md, INDEX.md, records/, FACTS.md, artifacts/, local skills/) through the `ctx` interface; load before any checkpoint, plan, fact, or record change, before reading any single record, and before loading a workspace-local skill.
---

# Workspace Context

Durable per-repository memory lives outside the repository at `~/.agents/workspaces/<workspace-id>/`. `CONTEXT.md` is a small fixed-schema hot-state file that is **replaced, never appended**; everything else is a separate record file discoverable through `INDEX.md`. The interface is `~/.agents/bin/ctx.sh` (POSIX sh; invoke as `sh ~/.agents/bin/ctx.sh …`; `ctx` below is shorthand for that).

## Layout

| Path | Role | Read when |
|---|---|---|
| `CONTEXT.md` | Hot state only: Identity, Objective, Constraints, State, Blockers, Next, Records, Skills. Hard caps **3,000 Unicode characters / 40 lines / 160 characters per line** (`CTX_MAX_CHARS`, `CTX_MAX_LINES`, `CTX_LINE_MAX`). | Every startup, before relying on conversation history. |
| `INDEX.md` | One generated line per record: id, type, status, created, title, tags, read-when. | When the hot state points at a record, or when searching for prior decisions or evidence. |
| `records/<T>-<NNNN>-<slug>.md` | One record per file with flat YAML frontmatter (`id`, `type`, `status`, `title`, `tags`, `created`, optional `read-when`, `supersedes`). Types: `decision` (D), `checkpoint` (C), `evidence` (E), `blocker` (B), `fact` (F), `history` (H). Statuses: `current`, `superseded`, `deferred`, `resolved`, `disputed`. Body caps: fact 300, blocker 400, checkpoint 600, decision 800 characters; title 80; evidence and history unlimited. | Only the single record whose `read-when` trigger or id matches the current action. |
| `FACTS.md` | **Generated** by `ctx index` from `records/F-*.md`: one line per non-superseded fact (id, status, title, first body line). Never hand-edited. | Startup; before asking the user anything already answered. `ctx facts` prints the same view. |
| `skills/<name>/SKILL.md` | Workspace-local skills: repeatable procedures valid only for this workspace (see Local skills). | Only when `CONTEXT.md` `## Skills` names it for the current action. |
| `artifacts/` | Everything that is not a record: `artifacts/plans/<dated-slug>.md` (plans and checklists; `CONTEXT.md` links the active one), `artifacts/reports/`, and free-form subfolders for scripts or generated outputs. | When the hot state or a record points at a file there. |

## Rules

- [WSC-001] Workspace identity: a Git repository is identified by the canonical real path of its root, a non-repository task by the canonical real path of its folder; the workspace id is `<sanitized-basename>-<sha256(path) first 16 hex>` and the directory is `~/.agents/workspaces/<workspace-id>/`. `ctx id` and `ctx dir` compute both; `ctx init` creates the skeleton.
- [WSC-002] Startup: run `ctx status`, read `CONTEXT.md` and the linked `FACTS.md`, and read `INDEX.md` only as far as the `## Records` section points. Never open, grep, glob, summarize, preload, or bulk-read `records/`; open one record at a time through `ctx show <id>` for a matching `read-when` trigger, an id named in the hot state, or an explicit parent handoff.
- [WSC-003] Writing: every material event (decision, phase change, delegated result, blocker found or cleared, user choice, reusable fact, evidence worth keeping) becomes **one new record** via `ctx add`, plus an in-place update of the affected `CONTEXT.md` field. Prose, chronology, rationale, and evidence never enter `CONTEXT.md`; it holds the conclusion and the record id. `## Records` lists only `current`/`deferred` records, one line each (`- <id> <title> — read when <trigger>`), at most 10 lines.
- [WSC-004] Cap enforcement: `ctx status` returns exit 1 when `CONTEXT.md` exceeds a cap; `ctx add` refuses a title or body over its type cap; `ctx check` fails on any over-cap line, title, or body. An over-cap write is invalid: keep the one-sentence conclusion in the capped record and move detail into an `evidence` record. There is no user consultation step and no headroom target; the cap is the contract.
- [WSC-004A] Brevity is mandatory in every record and hot-state field: one conclusion per record, statement first, no narrative, no restated constraints, no tool output. A fact is one sentence plus its scope and evidence pointer. Length that "might be useful later" goes to an `evidence` record with a precise `read-when`.
- [WSC-005] Supersession, not deletion: a changed decision is a new record with `-s <old-id>` (the old one becomes `superseded`); a cleared blocker is `ctx set <id> status resolved`. Records are never edited to rewrite history and never deleted; provenance stays in the record body.
- [WSC-006] `INDEX.md` and `FACTS.md` are generated (`ctx index`) and must be fresh after every record write; `ctx check` validates caps, section schema, frontmatter, id uniqueness, index and facts freshness, final newlines, local-skill listing, and that every id referenced by `CONTEXT.md` exists. Run `ctx check` before ending a turn that touched the workspace.
- [WSC-007] Ownership: only the parent or coordinating agent writes `CONTEXT.md`, `INDEX.md`, `records/`, `FACTS.md`, and plans; child agents report candidate records upward. All writes are secret-free (no tokens, credentials, cookies, raw `.env`), UTF-8, LF, final newline.
- [WSC-008] Negative constraints, deferred user choices, unresolved blockers, and active authority always remain in `CONTEXT.md` `## Constraints`/`## Blockers`; a record may hold their history but is never their only location.
- [POL-078] Residual `.omx`/`.omc` state and historical session artifacts are not authoritative current guidance unless the user explicitly directs otherwise.

## Local skills

- [WSC-020] A workspace-local skill lives at `~/.agents/workspaces/<workspace-id>/skills/<name>/SKILL.md` with standard frontmatter (`name` equal to the folder, `description` naming the trigger). It is visible only to sessions of that workspace: no host scans the folder, nothing is written into the repository, and other workspaces are unaffected.
- [WSC-021] Discovery happens through `CONTEXT.md` only: `## Skills` lists every local skill as `- <name> — load when <trigger>`. At startup read that section; load a local skill (`cat` its `SKILL.md`) only when its trigger matches the current action. `ctx skills` lists them; `ctx check` fails when the listing and the folder disagree.
- [WSC-022] Local skills hold procedure (how): exact commands, order, checks, and pitfalls for a task that recurs in this workspace. State, decisions, facts, and evidence stay in records. A local skill never restates shared policy or global skills.
- [WSC-023] Propose a local skill to the user — through the user-action flow — when the same multi-step procedure has been executed at least twice in this workspace, or when a sequential procedure with more than three ordered steps was reconstructed from records or conversation. Create one without a proposal only when the user asks. Creation and updates follow the global `workspace-skill-create` skill.
- [WSC-024] Promote a local skill to `~/.agents/skills/` only when the user asks or the same procedure is needed in a second workspace.

## Tracking and checkpoints (formerly workspace-records.md)

- [POL-056] When a task already has a plan or checklist, the parent or coordinating agent MUST re-read it and update item status whenever material progress, scope, evidence, or blockers change; when inputs are unchanged, plan, checklist, and dependency monitoring MUST remain incremental and event-driven without repeated full rescans.
- [POL-057] Immediately before ending a response, the parent or coordinating agent MUST verify that every plan or checklist item is either completed or explicitly marked with its remaining work or blocker. The agent MUST NOT claim completion while an item is silently pending.
- [POL-058] The parent or coordinating agent MUST update both the environment-provided plan tracker and the external cumulative checklist immediately after a delegated result is received, a phase changes, a material mutation completes, validation changes confidence, or a blocker is discovered or cleared.
- [POL-059] A conversational progress message does not count as a checklist update. The actual tracked item and its durable cumulative record MUST reflect the same current state before further phase transitions.
- [POL-071A] Before the main session changes work lanes and after each material child or dependency event, it MUST write a compact resume checkpoint: a `checkpoint` record via `ctx add checkpoint` (scope, owner/session identity, state, dependencies/blockers, evidence pointer, next resume action) and an updated `## State`/`## Next` in `CONTEXT.md`; the main session MUST reread the hot state before acting when it resumes. Checkpoints are incremental, concise, secret-free, and written only by the parent/main session; children report material events upward.

## FACTS.md ledger

Facts are `fact` records (`ctx add fact "<title>" -t <tags>` with the statement as body); `FACTS.md` is the generated summary of them. The rules below keep their identifiers and now apply to fact records.

- [POL-079] The parent or coordinating agent MUST record a fact when at least one evidence-backed reusable fact, user choice, exact identifier or path, or non-secret authority state is likely to be needed across sessions or has already triggered a repeated question. `CONTEXT.md` `## Identity` links `FACTS.md` when facts exist; `CONTEXT.md` remains the single hot-state source, and facts MUST NOT become a second narrative source.
- [POL-079A] Only the parent or coordinating agent MAY create or change fact records; child agents MUST report candidate facts upward. Before recording a candidate, the parent or coordinator MUST validate it against its source or evidence. Each fact record MUST contain the statement as its first body line (one sentence, ≤300 characters in total body), then scope, source or evidence pointer, and freshness or expiry expectation; status is `current`, `superseded`, `disputed`, or `deferred`. Corrections are new records with `-s <old-id>`; a stale or contested fact is set `disputed`; dynamic facts MUST be revalidated when their freshness is insufficient.
- [POL-079B] Fact records MUST record only reusable stable facts, user choices, exact identifiers or paths, and non-secret authority state, and MUST NOT contain credentials, raw secrets, plans, transient logs, commentary, or undigested tool output. Superseded or disputed history MUST be preserved rather than silently rewritten; conflicting sources MUST both remain recorded as `disputed` until resolved.

## Legacy layout (pre-records workspaces)

Workspaces created before this skill have a long narrative `CONTEXT.md`, optional `DETAIL.<value>.md` files, a `Detail Index` section, a hand-written `FACTS.md`, and loose top-level folders (`plans/`, `reports/`, `scripts/`, …). Until migrated: read `CONTEXT.md` as the hot state, treat each `Detail Index` entry exactly like a record `read-when` trigger (open one named DETAIL file only for its matching action, never bulk-read), read the hand-written `FACTS.md` as the fact source, and do not append further history to the legacy file — demote instead. `ctx status` flags the legacy layout; `ctx migrate` converts it losslessly: old `CONTEXT.md`, every `DETAIL.*.md`, and the hand-written `FACTS.md` become `history` records with verbatim bodies, each legacy fact becomes a `fact` record tagged `legacy` (exempt from the body cap until rewritten), every other top-level folder moves under `artifacts/`, and a fresh skeleton `CONTEXT.md` is written for the parent to fill from the records. Migration is run only when the user requests it or the parent is already rewriting that workspace.

## Superseded rules

`[POL-074]`, `[POL-075]`, `[POL-078A]`, `[POL-078B]`, `[POL-078C]`, `[POL-078D]`, `[POL-097]`, and the former `context-compaction` skill (`[CTX-001]`–`[CTX-022]`, including the 12,000-character consultation threshold and 9,000-character target) are replaced by the layout and rules above. Their former text was archived in the pre-refactor snapshot taken on 2026-09-04 and is no longer part of the live policy.
