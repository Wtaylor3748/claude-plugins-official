# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is Anthropic's official Claude Code Plugin Marketplace — a centralized registry and reference implementation for Claude Code plugins. It hosts the `marketplace.json` source-of-truth that Claude Code reads to discover and install plugins, along with the actual plugin source files.

Plugins can include: slash commands, model-invoked skills, autonomous agents, event-driven hooks, and MCP server configurations.

## Validation Commands

The repository uses [Bun](https://bun.sh/) for all validation scripts. There is no `package.json` at the root — Bun is invoked directly.

```bash
# Validate marketplace.json (required fields, no duplicates)
bun .github/scripts/validate-marketplace.ts .claude-plugin/marketplace.json

# Check plugins are alphabetically sorted
bun .github/scripts/check-marketplace-sorted.ts

# Auto-fix sort order
bun .github/scripts/check-marketplace-sorted.ts --fix

# Validate YAML frontmatter in agent/skill/command .md files
cd .github/scripts && bun install yaml   # install dependency first
bun .github/scripts/validate-frontmatter.ts                    # scan all
bun .github/scripts/validate-frontmatter.ts plugins/my-plugin  # specific dir
bun .github/scripts/validate-frontmatter.ts path/to/file.md    # specific file
```

CI runs `validate-marketplace` and `check-marketplace-sorted` on any PR touching `.claude-plugin/marketplace.json`. It runs `validate-frontmatter` on any PR touching `agents/*.md`, `skills/*/SKILL.md`, or `commands/*.md`.

## Repository Structure

```
.claude-plugin/marketplace.json   # Central registry — source of truth for all plugins
plugins/                          # Internal plugins maintained by Anthropic (~34 plugins)
  example-plugin/                 # Reference template — read this when building new plugins
external_plugins/                 # Third-party partner plugins
.github/
  scripts/                        # Bun validation scripts
  workflows/                      # CI/CD (validate, close-external-prs, python-publish)
```

## marketplace.json Schema

Each entry in the `plugins` array requires `name`, `description`, and `source`. Plugins **must be alphabetically sorted by name** (case-insensitive).

```json
{
  "name": "my-plugin",           // lowercase, unique
  "description": "...",
  "source": "./plugins/my-plugin",  // local path
  "author": { "name": "Anthropic", "email": "support@anthropic.com" },
  "category": "development",
  "homepage": "https://..."
}
```

For external/community plugins, `source` is an object:

```json
"source": {
  "source": "url",
  "url": "https://github.com/org/repo.git",
  "sha": "<commit-sha>"           // pin to a specific commit
}
// or for a subdirectory of another repo:
"source": {
  "source": "git-subdir",
  "url": "https://github.com/org/repo.git",
  "path": "plugins/my-plugin",
  "ref": "main",
  "sha": "<commit-sha>"
}
```

## Plugin Structure

Each plugin follows this layout (see `plugins/example-plugin/` for the canonical reference):

```
plugin-name/
├── .claude-plugin/plugin.json    # Plugin metadata (name, description, author)
├── .mcp.json                     # MCP server config (optional)
├── skills/
│   └── skill-name/
│       └── SKILL.md              # Preferred format for both commands and skills
├── commands/
│   └── command-name.md           # Legacy format (still supported, avoid for new plugins)
├── agents/
│   └── agent-name.md             # Autonomous agent definitions
└── hooks/
    ├── hooks.json                 # Hook event configuration
    └── *.py                       # Hook implementations
```

### Skill/Command Frontmatter

Skills in `skills/<name>/SKILL.md` (preferred over legacy `commands/*.md`):

```yaml
---
name: skill-name
description: Short description shown in /help
argument-hint: <required-arg> [optional-arg]
allowed-tools: [Read, Glob, Grep, Bash]
model: haiku   # optional override
---
```

User-invoked skills act as slash commands (`/skill-name`). Model-invoked skills are triggered automatically based on context — the `description` field is what Claude reads to decide when to activate them; write it as "This skill should be used when the user asks to '...', mentions '...', or discusses ...".

### Agent Frontmatter

```yaml
---
name: agent-name
description: What this agent does
---
```

Agents require both `name` and `description`. Commands require `description`. Skills require `description` or `when_to_use`.

### MCP Server Config (`.mcp.json`)

Supports `stdio`, `http`, `sse`, and `ws` types. Environment variables are expanded with `${VAR_NAME}`:

```json
{
  "server-name": {
    "type": "http",
    "url": "https://api.example.com/mcp/",
    "headers": { "Authorization": "Bearer ${MY_TOKEN}" }
  }
}
```

## Key Conventions

- **Internal plugins** (in `plugins/`) are Anthropic-maintained. The CI workflow `close-external-prs.yml` automatically closes PRs from non-Anthropic contributors.
- **External plugins** (in `external_plugins/` or remote URLs) are community/partner maintained.
- When adding a plugin to `marketplace.json`, always run the sort-fix script afterward to maintain alphabetical order.
- The `example-plugin` in `plugins/example-plugin/` is the authoritative reference for all plugin conventions — check it first when unsure about structure.
- LSP plugins use `"strict": false` in `marketplace.json` and a `lspServers` field describing the language server command and file extension mappings.
