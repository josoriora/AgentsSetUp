# Agent Skills Setup Guide

This guide explains how the multi-agent skill system in this repository works, how to replicate it from scratch, and how to add new skills so every agent brand picks them up automatically.

---

## 1. Overview

The system uses a **shared-docs-plus-adapters** pattern:

- **One shared source of truth** — tool-agnostic playbooks in `agent-skills/shared/` contain all the actual knowledge.
- **Thin per-agent adapters** — each agent brand has a small wiring file in its native format that points to the shared docs.
- **No duplication** — updating a shared playbook propagates to all agents automatically.

| Agent | Wiring mechanism | Auto-loads via |
|---|---|---|
| Claude Code | `@import` in `.claude/CLAUDE.md` | Loaded at session start |
| GitHub Copilot | `.github/copilot-instructions.md` | Loaded on every request |
| OpenAI Codex | `AGENTS.md` (context) + `SKILL.md` (trigger) | Directory tree walk |

---

## 2. File Structure

```
AGENTS.md                              ← universal project briefing
                                         (Codex primary, Copilot fallback)
.claude/
  CLAUDE.md                            ← @imports AGENTS.md + per-skill adapters
  settings.local.json                  ← Claude Code permissions (not skill-related)
.github/
  copilot-instructions.md              ← Copilot global instructions
  workflows/                           ← CI/CD workflows (not skill-related)
agent-skills/
  shared/                              ← source of truth — tool-agnostic playbooks
    <skill-name>.md
  claude/
    <skill-name>.md                    ← Claude-specific adapter per skill
  copilot/
    instructions/                      ← informational only; canonical file is .github/
  codex/
    <skill-name>/
      SKILL.md                         ← Codex skill (YAML frontmatter required)
```

**Key rule:** `agent-skills/shared/` is the only place where skill knowledge lives. Everything else is wiring.

---

## 3. How Each Agent Connects

### Claude Code

**Load chain:**

```
.claude/CLAUDE.md
  └─ @AGENTS.md                        (project context)
  └─ @agent-skills/claude/<skill>.md   (skill adapter)
       └─ references agent-skills/shared/<skill>.md
```

- `.claude/CLAUDE.md` is loaded automatically at the start of every Claude Code session.
- Uses `@path/to/file` import syntax — Claude expands these at launch (max 5 hops deep).
- Each skill adapter is a separate `@import` line in `CLAUDE.md`.

**Official docs:** [Claude Code — Memory and CLAUDE.md](https://docs.anthropic.com/en/docs/claude-code/memory)

---

### GitHub Copilot

**Load chain:**

```
.github/copilot-instructions.md
  └─ instructs Copilot to read agent-skills/shared/<skill>.md files
```

- `.github/copilot-instructions.md` is loaded automatically on every Copilot request in the repo.
- Copilot has no import syntax — the file contains Markdown text that directs Copilot to read specific files.
- Adding a skill means adding one line to the file list in `.github/copilot-instructions.md`.

**Official docs:** [GitHub Copilot — Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)

---

### OpenAI Codex

**Load chain:**

```
AGENTS.md                              (project context — loaded by directory walk)
agent-skills/codex/<skill>/SKILL.md   (skill trigger — matched by description field)
  └─ references agent-skills/shared/<skill>.md
```

- Codex walks the directory tree from the git root to the current working directory, loading every `AGENTS.md` it finds.
- `SKILL.md` files are triggered when Codex determines the task matches the `description:` field.
- `SKILL.md` requires YAML frontmatter with exactly two fields: `name:` and `description:`.

**Official docs:**
- [OpenAI Codex — AGENTS.md](https://platform.openai.com/docs/codex)
- [OpenAI Codex — Skills](https://platform.openai.com/docs/codex)

---

## 4. Setup from Scratch

Follow these steps to replicate this infrastructure in a new repository.

**Prerequisites:** git initialized, repo root established.

### Step 1 — Create the directory structure

```
mkdir -p agent-skills/shared
mkdir -p agent-skills/claude
mkdir -p agent-skills/copilot/instructions
mkdir -p agent-skills/codex
mkdir -p .claude
mkdir -p .github
```

### Step 2 — Create `AGENTS.md` at the repo root

This is the universal project briefing. All agents read it.

```markdown
# <Your Project Name>

<One paragraph describing what this repository is and does.>

## Shared playbooks

All detailed guidance lives in `agent-skills/shared/`:

- *(add entries here as skills are added)*

## Per-agent adapters

- **Claude Code:** `agent-skills/claude/<skill>.md`
- **Copilot:** `.github/copilot-instructions.md`
- **Codex skill:** `agent-skills/codex/<skill>/SKILL.md`
```

### Step 3 — Create `.claude/CLAUDE.md`

```
@AGENTS.md
```

*(Add `@agent-skills/claude/<skill>.md` lines here as skills are added.)*

### Step 4 — Create `.github/copilot-instructions.md`

```markdown
Use `agent-skills/shared/` as the source of truth for this repository.

Read only the files relevant to the task:
- *(add entries here as skills are added)*

Working rules:
- *(add any global constraints here)*
```

### Step 5 — Verify

At this point the following files should exist:

```
AGENTS.md
.claude/CLAUDE.md
.github/copilot-instructions.md
agent-skills/shared/        (empty, ready for playbooks)
agent-skills/claude/        (empty, ready for adapters)
agent-skills/copilot/instructions/   (informational use)
agent-skills/codex/         (empty, ready for skills)
```

The infrastructure is ready. Add skills using the walkthrough in Section 5.

---

## 5. Adding a New Skill

This walkthrough uses `my-skill` as the example skill name throughout. Replace it with your actual skill name.

---

### Step 1 — Create the shared playbook

**File to create:** `agent-skills/shared/my-skill.md`

This file is the single source of truth. Write tool-agnostic content: what the skill covers, key procedures, and guardrails.

**Example:**

```markdown
# My Skill

## Purpose

Describe what this skill is for and when to use it.

## Key guidance

- Guideline one.
- Guideline two.
- Guideline three.

## Guardrails

- Do not do X.
- Defer Y until Z condition is met.
```

---

### Step 2 — Create the Claude adapter and wire it

**File to create:** `agent-skills/claude/my-skill.md`

A brief Claude-specific briefing that points to the shared playbook.

**Example:**

```markdown
# Claude My Skill Briefing

Use `agent-skills/shared/my-skill.md` as the reference for tasks related to <topic>.

Key points:
- <summary point one>
- <summary point two>
```

**Then wire it into `.claude/CLAUDE.md`.**

Before:
```
@AGENTS.md
@agent-skills/claude/existing-skill.md
```

After:
```
@AGENTS.md
@agent-skills/claude/existing-skill.md
@agent-skills/claude/my-skill.md
```

---

### Step 3 — Update Copilot instructions

**File to edit:** `.github/copilot-instructions.md`

Add the new shared playbook to the file list.

Before:
```
Read only the files relevant to the task:
- `agent-skills/shared/existing-skill.md`
```

After:
```
Read only the files relevant to the task:
- `agent-skills/shared/existing-skill.md`
- `agent-skills/shared/my-skill.md`
```

---

### Step 4 — Create the Codex skill

**File to create:** `agent-skills/codex/my-skill/SKILL.md`

The YAML frontmatter is required. The `description:` field is what Codex matches against to decide when to load this skill — write it as a clear trigger statement.

**Example:**

```markdown
---
name: my-skill
description: Use this skill when working on <topic>. It provides guidance on <what it covers> using the shared playbooks in agent-skills/shared/.
---

# My Skill

Use `agent-skills/shared/my-skill.md` as the primary reference.

Scope:
- <what this skill handles>
- <what it does not handle>
```

> **Note:** Only `name:` and `description:` are valid in the frontmatter. Any other fields are ignored.

---

### Step 5 — Update `AGENTS.md`

**File to edit:** `AGENTS.md` (repo root)

Add the new skill to "Per-agent adapters" and the new playbook to "Shared playbooks".

Before:
```markdown
## Shared playbooks

- `existing-skill.md` — description of existing skill

## Per-agent adapters

- **Claude Code:** `agent-skills/claude/existing-skill.md`
- **Copilot:** `.github/copilot-instructions.md`
- **Codex skill:** `agent-skills/codex/existing-skill/SKILL.md`
```

After:
```markdown
## Shared playbooks

- `existing-skill.md` — description of existing skill
- `my-skill.md` — description of my skill

## Per-agent adapters

- **Claude Code:** `agent-skills/claude/existing-skill.md`, `agent-skills/claude/my-skill.md`
- **Copilot:** `.github/copilot-instructions.md`
- **Codex skill:** `agent-skills/codex/existing-skill/SKILL.md`, `agent-skills/codex/my-skill/SKILL.md`
```

---

### Checklist

After completing all five steps, the following files should exist for `my-skill`:

- [ ] `agent-skills/shared/my-skill.md` — created
- [ ] `agent-skills/claude/my-skill.md` — created
- [ ] `.claude/CLAUDE.md` — has `@agent-skills/claude/my-skill.md`
- [ ] `.github/copilot-instructions.md` — has `agent-skills/shared/my-skill.md` in file list
- [ ] `agent-skills/codex/my-skill/SKILL.md` — created with valid frontmatter
- [ ] `AGENTS.md` — updated with new skill and playbook entries

---

## 6. File Templates

Copy-paste starting points for each file type.

### `agent-skills/shared/<skill-name>.md`

```markdown
# <Skill Name>

## Purpose

<What this skill covers and when to use it.>

## Key guidance

- <Guideline>
- <Guideline>

## Guardrails

- <What to avoid or defer>
```

---

### `agent-skills/claude/<skill-name>.md`

```markdown
# Claude <Skill Name> Briefing

Use `agent-skills/shared/<skill-name>.md` as the reference for tasks related to <topic>.

Key points:
- <Summary point>
- <Summary point>
```

---

### `agent-skills/codex/<skill-name>/SKILL.md`

```markdown
---
name: <skill-name>
description: Use this skill when working on <topic>. It covers <what it handles> using the shared playbooks in agent-skills/shared/.
---

# <Skill Name>

Use `agent-skills/shared/<skill-name>.md` as the primary reference.

Scope:
- <What this skill handles>
- <What it does not handle>
```

---

### `AGENTS.md` — new skill entry (Shared playbooks section)

```markdown
- `<skill-name>.md` — <one-line description>
```

---

### `.github/copilot-instructions.md` — new skill entry (file list)

```markdown
- `agent-skills/shared/<skill-name>.md`
```
