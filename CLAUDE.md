# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A **configuration and documentation repository** — no executable code, no build system, no tests. It is a plugin system for Claude Code consisting of agents, skills, and commands that extend Claude's capabilities with specialized domain expertise.

## Key Directories

- `.claude/agents/` — Global subagents available in all projects (codebase-analyst, library-researcher)
- `.claude/skills/` — Meta-skills for building more agents and skills (agent-development, skill-development, engineer-skill-creator, prp-core-runner)
- `.claude/commands/` — Slash commands (ultra-think, prp-core-*)
- `.claude/PRPs/templates/` — PRP (Planning/Requirements/Procedure) templates
- `dev-team/` — 25 domain-expert agents, each with 4–7 paired skills

## Architecture: Agents vs Skills

**Agents** are autonomous subprocesses invoked via the `Task` tool. They have a YAML frontmatter with `name`, `description` (containing trigger examples), `model`, `color`, and `tools`. The description's trigger examples are critical — they determine when Claude auto-invokes the agent.

**Skills** are knowledge documents loaded on demand via the `Skill` tool. They follow **progressive disclosure**: Claude first reads the YAML metadata, then loads `SKILL.md` only when the skill is relevant, then loads `references/` files only when deep detail is needed.

**Commands** (`.claude/commands/*.md`) are user-invocable slash commands that expand to full prompts.

## Agent File Format

```yaml
---
name: expert-name          # 3–50 chars, lowercase, hyphens only
description: |             # Third-person, includes <example> blocks
  Use this agent when...
  <example>
  Context: ...
  user: "..."
  assistant: "..."
  </example>
model: inherit             # Almost always 'inherit'
color: blue                # For UI identification
tools: ["Read", "Write", "Edit", "Grep", "Glob", "Bash"]
---

You are a [domain] specialist...
```

## Skill File Format

Each skill lives in a directory under `skills/` containing:
- `SKILL.md` — Main knowledge document with YAML frontmatter (`name`, `description`)
- `references/` — Deep-dive reference files loaded only when needed

## Dev-Team Structure

25 expert agents in `dev-team/`, each paired with domain skills:
- Frontend: react-expert, angular-expert, vue-expert, nextjs-expert, tailwind-expert
- Backend: nodejs-expert, python-expert, fastapi-expert, django-expert, go-expert, rust-expert
- Infrastructure: docker-expert, devops-expert, cloud-expert
- Data: database-expert
- Languages: typescript-expert, dotnet-expert, cpp-expert, langchain-expert
- Quality: testing-expert, codebase-analyst, system-architect
- Leadership: team-leader, uxui-designer, seo-specialist

## Naming Conventions

- Agent identifiers: `lowercase-hyphenated` (e.g., `react-expert`, `team-leader`)
- Skill identifiers: `domain-topic` format (e.g., `react-mastery`, `react-testing`)
- Commands: `prp-core-*` prefix for PRP workflow commands

## Design Principles

1. **Progressive disclosure** — Load only what's needed (metadata → SKILL.md → references)
2. **Principle of least privilege** — Agents only get the tools they need
3. **Trigger examples in descriptions** — Each agent description must have concrete `<example>` blocks showing when it should fire
4. **Third-person skill descriptions** — Skills describe themselves from Claude's perspective ("Use when the user needs...")

## PRP Workflow

The `prp-core-*` commands implement a structured development workflow:
1. `/prp-core-new-branch` — Create feature branch
2. `/prp-core-create` — Generate a PRP document in `.claude/PRPs/`
3. `/prp-core-execute` — Implement the PRP
4. `/prp-core-commit` — Commit changes
5. `/prp-core-pr` — Create pull request
6. `/prp-core-run-all` — Run the entire sequence

Use `/prp-core-runner` skill for orchestrated PRP execution.

## When Adding New Content

- **New expert agent**: Follow the agent format above; place in `dev-team/<expert-name>.md` or `.claude/agents/`
- **New skill**: Create `dev-team/<expert>/skills/<skill-name>/SKILL.md`; add references in `references/`
- **New command**: Add to `.claude/commands/<command-name>.md`
- **New PRP template**: Add to `.claude/PRPs/templates/`

Use the `agent-development` and `skill-development` skills (via `/agent-development` and `/skill-development`) for guided creation workflows.
