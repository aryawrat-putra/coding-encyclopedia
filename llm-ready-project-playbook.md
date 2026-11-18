# LLM-Ready Project Playbook

A tool-agnostic setup so any coding agent (Kilo Code, Claude Code, Codex, Cursor, Copilot, Windsurf) can read your project with full context.

> Verified against vendor docs and changelogs in Nov 2026. Agent tooling changes fast, so re-check the "Who reads what" section every few months.

---

## 1. The one rule

**Write once into tool-neutral names. Make every tool-specific folder a thin pointer.**

- Canonical files: `AGENTS.md`, `.agents/skills/`, `docs/`
- Tool folders (`.claude/`, `.kilo/`, `.cursor/`) only symlink or import from the canonical files
- Never put real content only inside a tool-specific folder, or you are locked in

---

## 2. Standard layout

```
repo/
├── AGENTS.md                      # canonical instructions (uppercase, at root)
├── CLAUDE.md                      # one line: @AGENTS.md
├── .agents/
│   ├── skills/<name>/SKILL.md     # canonical skills (+ scripts/, references/, assets/)
│   └── agents/<role>.md           # your own role prompts (no cross-tool standard exists)
├── .claude/skills -> ../.agents/skills    # symlink
├── docs/
│   ├── architecture.md            # data flow + "where things live"
│   ├── conventions.md             # naming, errors, API style, state mgmt
│   ├── glossary.md                # domain terms the model can't guess
│   ├── decisions/0001-*.md        # ADRs: why, not what
│   └── plans/                     # in-flight task plans
├── apps/web/AGENTS.md             # per-area overrides
├── apps/api/AGENTS.md
├── .env.example
└── .kilocodeignore                # or your tool's equivalent ignore file
```

Naming notes:
- `AGENTS.md` must be uppercase. Lowercase `agents.md` is not reliably picked up.
- Skill folder names: lowercase letters, digits and hyphens only.

---

## 3. Who reads what

| Tool | Instructions file | Skills location | Notes |
|---|---|---|---|
| **Kilo Code** | Root `AGENTS.md`, plus nested `AGENTS.md` loaded on demand when the agent reads files in that folder | `.agents/skills/` loaded by default; `.claude/skills/` only if Claude Code compatibility is enabled | Memory bank is deprecated in favor of AGENTS.md |
| **Claude Code** | `CLAUDE.md` first. Since v2.1.277 it falls back to `AGENTS.md` only when no `CLAUDE.md` exists; if both exist, `CLAUDE.md` wins | `.claude/skills/` | Commit `CLAUDE.md` with `@AGENTS.md` for a guarantee |
| **Codex** | `AGENTS.md` | `.agents/skills/` by default | |
| **Cursor / Copilot / Windsurf** | `AGENTS.md` | Varies by tool | Check each tool's skills docs |

Windows note: symlinks need Developer Mode or admin rights. Use a junction instead: `mklink /J .claude\skills .agents\skills`.

---

## 4. Ingesting skills (Kilo and any Agent Skills tool)

1. **Place the folder.** Project: `.agents/skills/<name>/SKILL.md`. Personal, all projects: `~/.agents/skills/<name>/SKILL.md`.
2. **Write valid frontmatter.**
   - `name`: lowercase letters, digits, hyphens; max 64 characters; no leading or trailing hyphen
   - `description`: required; max 1024 characters
   - Keep `name` identical to the folder name (safest across tools)
3. **Reload.** Run `/reload` in Kilo or start a new session. Skills are scanned at session start.
4. **Verify.** Ask the agent: "What skills do you have available?"
5. **External or shared skills.**
   - `skills.paths` and `skills.urls` go in `kilo.jsonc`
   - Paths outside the repo must be declared in your **user-level** config (typically `~/.config/kilo/kilo.jsonc`); project-level config cannot reach outside the project
   - Remote skills need an `index.json` manifest at the URL

### How skills get triggered

The model chooses a skill from its `description` alone. There is no keyword matching or semantic search. Therefore:

- Write descriptions as triggers: "Use when ... Covers ..."
- Name the skill explicitly when you want certainty: "use the add-api-endpoint skill"
- Vague descriptions mean unreliable triggering

### Example skill

```markdown
---
name: add-api-endpoint
description: Use when adding or changing a REST endpoint in apps/api. Covers route, zod validation, service layer, tests, and OpenAPI update.
---

1. Read docs/conventions.md (API section).
2. Add the route in apps/api/src/routes/.
3. Validate input with a zod schema from packages/shared.
4. Add a test next to the route as *.test.ts.
5. Update the OpenAPI spec.
6. Run `pnpm check` and fix until green.
```

### Starter skills for a full-stack app

- `add-api-endpoint`
- `add-db-migration`
- `add-ui-component`
- `write-tests`
- `debug-ci-failure`
- `release`

### Your `.agents` agent/role files

There is no cross-tool standard for subagent definitions. Keep role prompts as plain Markdown in `.agents/agents/<role>.md`, then either:

- reference them from `AGENTS.md`, or
- wrap them as skills (most portable), or
- symlink into a tool's own agents folder if that tool supports Markdown agent files

---

## 5. AGENTS.md template

Save as `~/.agents/templates/AGENTS.md` and copy into every new repo. Keep it under about 150 lines: concrete rules, no long paragraphs. Detail lives in `docs/` and skills; this file is the index.

```markdown
# <Project name>: <one-line purpose>

## Stack
<framework + version>, Node <ver>, <DB + version>, <ORM>, <package manager>

## Commands (run these, don't guess)
- install: `pnpm i`
- dev: `pnpm dev`
- test: `pnpm test`
- typecheck: `pnpm typecheck`
- lint: `pnpm lint`
- full gate: `pnpm check`
- migrate: `pnpm db:migrate`

## Map
- apps/web = UI
- apps/api = REST API
- packages/shared = zod schemas and shared types
- See docs/architecture.md

## Conventions
- kebab-case files, PascalCase components, camelCase variables
- Tests colocated as `*.test.ts`
- No default exports
- Details: docs/conventions.md

## Boundaries
- Never edit: applied migrations, lockfiles, `.env*`
- Ask first: new dependencies, schema changes, auth changes

## Skills
Workflow skills live in `.agents/skills/`. Use them when the task matches.

## Definition of done
- `pnpm check` passes
- Tests added or updated
- Docs updated if behavior changed
```

---

## 6. Efficiency tips

1. **Give the agent a verification loop.** One `check` command (typecheck + lint + tests) lets the agent fix its own mistakes until green. This is the biggest quality jump.
2. **Plan in a file, then implement from it.** Ask for `docs/plans/<task>.md`, review it, then execute in a fresh session. The plan survives context resets and works in any tool.
3. **One task per session.** Long sessions degrade. Start fresh often.
4. **Split models by job.** Strong model for planning and architecture, cheaper model for implementation.
5. **Point rules at shared docs.** In Kilo, the `instructions` array in `kilo.jsonc` accepts file paths and globs:
   ```jsonc
   { "instructions": ["docs/conventions.md"] }
   ```
   This avoids duplicating content into `.kilo/rules/`.
6. **Retire memory-bank files.** Move that content into `AGENTS.md`.
7. **Keep noise out of the index.** Ignore `node_modules`, `dist`, build output, lockfiles and generated code. Enable codebase indexing.
8. **Commit before big agent runs.** Gives you a clean diff and easy rollback.
9. **Correct the file, not the chat.** When an agent makes the same mistake twice, add one line to `AGENTS.md` or a skill.

---

## 7. Bootstrap script

Save as `bootstrap-agent-ready.sh` and run from the root of each new repo.

```bash
#!/usr/bin/env bash
set -e
mkdir -p .agents/skills .agents/agents docs/decisions docs/plans .claude
[ -f AGENTS.md ] || cp ~/.agents/templates/AGENTS.md AGENTS.md
printf '@AGENTS.md\n' > CLAUDE.md
[ -e .claude/skills ] || ln -s ../.agents/skills .claude/skills
touch .env.example docs/architecture.md docs/conventions.md docs/glossary.md
echo "Agent-ready skeleton created. Now fill in AGENTS.md and docs/."
```

---

## 8. New-project checklist

Copy this into each project's issue tracker or first commit.

### A. Instructions layer
- [ ] `AGENTS.md` at root, under ~150 lines, containing stack and versions, exact commands, folder map, conventions, boundaries, definition of done
- [ ] `CLAUDE.md` containing only `@AGENTS.md`
- [ ] Per-area `AGENTS.md` in `apps/web`, `apps/api` etc. for rules that differ by area
- [ ] `.agents/skills/` with at least 3 workflow skills
- [ ] `.claude/skills` symlinked (or junctioned) to `.agents/skills`
- [ ] Every skill has `name` (matching its folder) and a trigger-style `description`

### B. Context layer (`docs/`)
- [ ] `architecture.md` with a data-flow diagram and a "where things live" map
- [ ] `conventions.md` covering naming, error handling, API style, state management
- [ ] `decisions/` with an ADR for each major choice (framework, ORM, auth, hosting)
- [ ] `glossary.md` for domain terms
- [ ] Schema source of truth the agent can read (Drizzle or Prisma schema)
- [ ] API contract the agent can read (OpenAPI spec and/or shared zod types)
- [ ] `plans/` folder for in-flight task plans

### C. Executable guardrails
- [ ] One `check` command that runs typecheck, lint and tests
- [ ] CI runs the same `check` command
- [ ] Strict TypeScript, ESLint and Prettier configured
- [ ] Pre-commit hook for lint and format
- [ ] `.env.example` complete and commented; no secrets in the repo
- [ ] Setup works from a clean clone in one or two commands (tested)
- [ ] At least one example test per layer (unit, API, UI) for the agent to copy
- [ ] Agent ignore file (`.kilocodeignore` or equivalent) excluding build output, lockfiles, generated code

### D. Naming and structure standards
- [ ] Predictable layout: every feature folder follows the same pattern
- [ ] Consistent casing: kebab-case files, PascalCase components, camelCase variables, SCREAMING_SNAKE env vars
- [ ] Files kept under ~300 lines, split by responsibility
- [ ] Tests colocated or mirrored consistently
- [ ] Shared types or contracts between frontend and backend (never duplicated)

### E. Smoke test (2 minutes, fresh session)
- [ ] Ask: "How do I run tests, and what skills do you have?" Answers match reality
- [ ] Ask the agent to add a trivial endpoint. It follows conventions and passes `check` without hints

### F. Maintenance loop
- [ ] Same mistake twice means one new line in `AGENTS.md` or a skill
- [ ] Monthly: delete stale rules
- [ ] Monthly: confirm `CLAUDE.md`, `AGENTS.md` and `docs/` don't contradict each other

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill not showing up | Wrong path, missing frontmatter, or not reloaded | Path must be `skills/<name>/SKILL.md`; add `name` and `description`; run `/reload` |
| Skill never triggers | Vague description | Rewrite as "Use when ..."; or call it by name |
| Claude Code ignores `AGENTS.md` | A `CLAUDE.md` exists without the import, or an older version | Add `@AGENTS.md` to `CLAUDE.md` |
| Agent follows old rules | Config cached | Start a new session or `/reload` |
| Conflicting behavior between tools | Rules duplicated in several files | Keep one canonical file; make others pointers |
| External skill path blocked | Declared in project config | Declare it in user-level config |
