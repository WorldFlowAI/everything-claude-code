# LEGENDS

[![Stars](https://img.shields.io/github/stars/TahaGPT/LEGENDS?style=flat)](https://github.com/TahaGPT/LEGENDS/stargazers)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Shell](https://img.shields.io/badge/-Shell-4EAA25?logo=gnu-bash&logoColor=white)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?logo=typescript&logoColor=white)
![Markdown](https://img.shields.io/badge/-Markdown-000000?logo=markdown&logoColor=white)

**An end-to-end Claude Code toolkit: from idea to reviewed PR to release.**

Agents, skills, commands, hooks, and rules that plan, design, implement test-first, review, verify, document, deploy, and release, for any project. TypeScript/JavaScript and Rust are supported out of the box, and every agent takes its conventions from your project's `CLAUDE.md` and CI.

> Built on [everything-claude-code](https://github.com/affaan-m/everything-claude-code) by Affaan Mustafa (MIT). See [Credits](#credits).

---

## The Guides

The original author's guides explain the foundations this toolkit builds on (setup, token optimization, memory, evals, parallelization).

<table>
<tr>
<td width="50%">
<a href="https://x.com/affaanmustafa/status/2012378465664745795">
<img src="https://github.com/user-attachments/assets/1a471488-59cc-425b-8345-5245c7efbcef" alt="The Shorthand Guide to Everything Claude Code" />
</a>
</td>
<td width="50%">
<a href="https://x.com/affaanmustafa/status/2014040193557471352">
<img src="https://github.com/user-attachments/assets/c9ca43bc-b149-427f-b551-af6840c368f0" alt="The Longform Guide to Everything Claude Code" />
</a>
</td>
</tr>
<tr>
<td align="center"><b>Shorthand Guide</b><br/>Setup, foundations, philosophy. <b>Read this first.</b></td>
<td align="center"><b>Longform Guide</b><br/>Token optimization, memory persistence, evals, parallelization.</td>
</tr>
</table>

| Topic | What You'll Learn |
|-------|-------------------|
| Token Optimization | Model selection, system prompt slimming, background processes |
| Memory Persistence | Hooks that save/load context across sessions automatically |
| Continuous Learning | Auto-extract patterns from sessions into reusable skills |
| Verification Loops | Checkpoint vs continuous evals, grader types, pass@k metrics |
| Parallelization | Git worktrees, cascade method, when to scale instances |
| Subagent Orchestration | The context problem, iterative retrieval pattern |

---

## Cross-Platform Support

This plugin now fully supports **Windows, macOS, and Linux**. All hooks and scripts have been rewritten in Node.js for maximum compatibility.

### Package Manager Detection

The plugin automatically detects your preferred package manager (npm, pnpm, yarn, or bun) with the following priority:

1. **Environment variable**: `CLAUDE_PACKAGE_MANAGER`
2. **Project config**: `.claude/package-manager.json`
3. **package.json**: `packageManager` field
4. **Lock file**: Detection from package-lock.json, yarn.lock, pnpm-lock.yaml, or bun.lockb
5. **Global config**: `~/.claude/package-manager.json`
6. **Fallback**: First available package manager

To set your preferred package manager:

```bash
# Via environment variable
export CLAUDE_PACKAGE_MANAGER=pnpm

# Via global config
node scripts/setup-package-manager.js --global pnpm

# Via project config
node scripts/setup-package-manager.js --project bun

# Detect current setting
node scripts/setup-package-manager.js --detect
```

Or use the `/setup-pm` command in Claude Code.

---

## What's Inside

This repo is a **Claude Code plugin** - install it directly or copy components manually.

```
LEGENDS/
|-- .claude-plugin/   # Plugin and marketplace manifests
|   |-- plugin.json         # Plugin metadata and component paths
|   |-- marketplace.json    # Marketplace catalog for /plugin marketplace add
|
|-- agents/           # Specialized subagents for delegation
|   |-- planner.md           # Feature implementation planning
|   |-- architect.md         # System design decisions
|   |-- tdd-guide.md         # Test-driven development
|   |-- code-reviewer.md     # Quality and security review
|   |-- security-reviewer.md # Vulnerability analysis
|   |-- build-error-resolver.md
|   |-- rust-build-resolver.md
|   |-- explorer.md
|   |-- e2e-runner.md        # Playwright E2E testing
|   |-- implementer.md       # Executes approved plans test-first
|   |-- database-architect.md # Schema design, safe migrations, indexing
|   |-- api-designer.md      # Contract-first API and interface design
|   |-- devops.md            # CI/CD, Docker, env config, deployments
|   |-- release-manager.md   # Versioning, changelog, tags, releases
|   |-- refactor-cleaner.md  # Dead code cleanup
|   |-- doc-updater.md       # Documentation sync
|
|-- skills/           # Workflow definitions and domain knowledge
|   |-- coding-standards/           # Language best practices
|   |-- backend-patterns/           # API, database, caching patterns
|   |-- frontend-patterns/          # React, Next.js patterns
|   |-- continuous-learning/        # Auto-extract patterns from sessions (Longform Guide)
|   |-- strategic-compact/          # Manual compaction suggestions (Longform Guide)
|   |-- tdd-workflow/               # TDD methodology
|   |-- rust-patterns/              # Rust idioms, async, testing, security
|   |-- security-review/            # Security checklist
|   |-- eval-harness/               # Verification loop evaluation (Longform Guide)
|   |-- verification-loop/          # Continuous verification (Longform Guide)
|
|-- commands/         # Slash commands for quick execution
|   |-- tdd.md              # /tdd - Test-driven development
|   |-- plan.md             # /plan - Implementation planning
|   |-- e2e.md              # /e2e - E2E test generation
|   |-- code-review.md      # /code-review - Quality review
|   |-- build-fix.md        # /build-fix - Fix build errors
|   |-- ship.md             # /ship - Idea to reviewed PR, end to end
|   |-- release.md          # /release - Cut a versioned release
|   |-- refactor-clean.md   # /refactor-clean - Dead code removal
|   |-- learn.md            # /learn - Extract patterns mid-session (Longform Guide)
|   |-- checkpoint.md       # /checkpoint - Save verification state (Longform Guide)
|   |-- verify.md           # /verify - Run verification loop (Longform Guide)
|   |-- setup-pm.md         # /setup-pm - Configure package manager (NEW)
|
|-- rules/            # Always-follow guidelines (copy to ~/.claude/rules/)
|   |-- security.md         # Mandatory security checks
|   |-- coding-style.md     # Immutability, file organization
|   |-- testing.md          # TDD, 80% coverage requirement
|   |-- git-workflow.md     # Commit format, PR process
|   |-- agents.md           # When to delegate to subagents
|   |-- performance.md      # Model selection, context management
|
|-- hooks/            # Trigger-based automations
|   |-- hooks.json                # All hooks config (PreToolUse, PostToolUse, Stop, etc.)
|   |-- memory-persistence/       # Session lifecycle hooks (Longform Guide)
|   |-- strategic-compact/        # Compaction suggestions (Longform Guide)
|
|-- scripts/          # Cross-platform Node.js scripts (NEW)
|   |-- lib/                     # Shared utilities
|   |   |-- utils.js             # Cross-platform file/path/system utilities
|   |   |-- package-manager.js   # Package manager detection and selection
|   |-- hooks/                   # Hook implementations
|   |   |-- session-start.js     # Load context on session start
|   |   |-- session-end.js       # Save state on session end
|   |   |-- pre-compact.js       # Pre-compaction state saving
|   |   |-- suggest-compact.js   # Strategic compaction suggestions
|   |   |-- evaluate-session.js  # Extract patterns from sessions
|   |-- setup-package-manager.js # Interactive PM setup
|
|-- tests/            # Test suite (NEW)
|   |-- lib/                     # Library tests
|   |-- hooks/                   # Hook tests
|   |-- run-all.js               # Run all tests
|
|-- contexts/         # Dynamic system prompt injection contexts (Longform Guide)
|   |-- dev.md              # Development mode context
|   |-- review.md           # Code review mode context
|   |-- research.md         # Research/exploration mode context
|
|-- examples/         # Example configurations and sessions
|   |-- CLAUDE.md           # Example project-level config
|   |-- user-CLAUDE.md      # Example user-level config
|
|-- mcp-configs/      # MCP server configurations
|   |-- mcp-servers.json    # GitHub, Supabase, Vercel, Railway, etc.
|
|-- marketplace.json  # Self-hosted marketplace config (for /plugin marketplace add)
```

---

## Installation

### Option 1: Install as Plugin (Recommended)

The easiest way to use this repo - install as a Claude Code plugin:

```bash
# Add this repo as a marketplace
/plugin marketplace add TahaGPT/LEGENDS

# Install the plugin
/plugin install legends@legends
```

Or add directly to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "legends": {
      "source": {
        "source": "github",
        "repo": "TahaGPT/LEGENDS"
      }
    }
  },
  "enabledPlugins": {
    "legends@legends": true
  }
}
```

This gives you instant access to all commands, agents, skills, and hooks.

---

### Option 2: Manual Installation

If you prefer manual control over what's installed:

```bash
# Clone the repo
git clone https://github.com/TahaGPT/LEGENDS.git

# Copy agents to your Claude config
cp LEGENDS/agents/*.md ~/.claude/agents/

# Copy rules
cp LEGENDS/rules/*.md ~/.claude/rules/

# Copy commands
cp LEGENDS/commands/*.md ~/.claude/commands/

# Copy skills
cp -r LEGENDS/skills/* ~/.claude/skills/
```

#### Add hooks to settings.json

Copy the hooks from `hooks/hooks.json` to your `~/.claude/settings.json`.

#### Configure MCPs

Copy desired MCP servers from `mcp-configs/mcp-servers.json` to your `~/.claude.json`.

**Important:** Replace `YOUR_*_HERE` placeholders with your actual API keys.

---

## Workflows

### Ship a feature end to end

```
/ship "<feature description>"
```
spec → explore → design (architect / database-architect / api-designer, as needed) → plan (**you approve**) → implement test-first → code + security review → fix loop → E2E → verify → docs + CI/infra → commit → PR (**you approve**)

### Cut a release

```
/release --dry-run   # preview version + changelog, change nothing
/release             # bump, changelog, verify, tag locally; you approve push/publish
```

### Step by step

```
/plan          # plan the implementation, wait for approval
/tdd           # tests first, then code
/verify        # build, types, lint, tests, security scan
/code-review   # self-review before the PR
/checkpoint    # save progress state
```

### Other commands

```
/build-fix       # fix build errors (TS/JS, or cargo via rust-build-resolver)
/refactor-clean  # find and remove dead code safely
/e2e             # generate and run Playwright journeys
/orchestrate     # run a custom sequence of agents
/learn           # extract reusable patterns from the session
```

### Which agent for which job

| Situation | Agent / Command |
|-----------|-----------------|
| Feature from idea to PR | `/ship` |
| Understanding unfamiliar code, starting a bug fix | `explorer` |
| Planning a feature | `/plan` → `planner` |
| System design decisions | `architect` |
| Schema changes, migrations, slow queries | `database-architect` |
| New or changed endpoints, RPCs, events, public interfaces | `api-designer` |
| Implementing an approved plan | `implementer` |
| Writing tests first | `/tdd` → `tdd-guide` |
| TS/JS build errors | `/build-fix` → `build-error-resolver` |
| Rust build, clippy, test failures | `/build-fix` → `rust-build-resolver` |
| Pre-PR quality check | `/code-review` → `code-reviewer` |
| Auth, input handling, secrets, payments, LLM data | `security-reviewer` |
| Critical user flows | `/e2e` → `e2e-runner` |
| Dead code cleanup | `/refactor-clean` → `refactor-cleaner` |
| Docs and codemaps | `doc-updater` |
| CI, Docker, env config, deployments | `devops` |
| Versioning, changelog, tags, releases | `/release` → `release-manager` |

### Language support

The agents detect the stack from the project's manifests. TypeScript/JavaScript is covered by the original skills. Rust is covered by the `rust-patterns` skill and the `rust-build-resolver` agent: `/verify`, `/build-fix`, and `verification-loop` switch to cargo commands when a `Cargo.toml` is present. Project conventions in your `CLAUDE.md` and CI config always take precedence over the defaults in this collection.

---

## Key Concepts

### Agents

Subagents handle delegated tasks with limited scope. Example:

```markdown
---
name: code-reviewer
description: Reviews code for quality, security, and maintainability
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior code reviewer...
```

### Skills

Skills are workflow definitions invoked by commands or agents:

```markdown
# TDD Workflow

1. Define interfaces first
2. Write failing tests (RED)
3. Implement minimal code (GREEN)
4. Refactor (IMPROVE)
5. Verify 80%+ coverage
```

### Hooks

Hooks fire on tool events. Example - warn about console.log:

```json
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(ts|tsx|js|jsx)$\"",
  "hooks": [{
    "type": "command",
    "command": "#!/bin/bash\ngrep -n 'console\\.log' \"$file_path\" && echo '[Hook] Remove console.log' >&2"
  }]
}
```

### Rules

Rules are always-follow guidelines. Keep them modular:

```
~/.claude/rules/
  security.md      # No hardcoded secrets
  coding-style.md  # Immutability, file limits
  testing.md       # TDD, coverage requirements
```

---

## Running Tests

The plugin includes a comprehensive test suite:

```bash
# Run all tests
node tests/run-all.js

# Run individual test files
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/hooks/hooks.test.js
```

---

## Contributing

**Contributions are welcome and encouraged.**

This repo is meant to be a community resource. If you have:
- Useful agents or skills
- Clever hooks
- Better MCP configurations
- Improved rules

Please contribute! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Ideas for Contributions

- Language-specific skills (Python, Go)
- Framework-specific configs (Django, Rails, Laravel)
- Cloud-specific DevOps guidance (Kubernetes, Terraform, AWS)
- Testing strategies (different frameworks)
- Domain-specific knowledge (ML, data engineering, mobile)

---

## Credits

LEGENDS is a fork of [everything-claude-code](https://github.com/affaan-m/everything-claude-code) by [Affaan Mustafa](https://x.com/affaanmustafa), released under the MIT License. The original agents, skills, hooks, rules, and guides come from that project.

LEGENDS adds the end-to-end pipeline (`/ship`, `/release`), the `explorer`, `implementer`, `database-architect`, `api-designer`, `devops`, `release-manager`, and `rust-build-resolver` agents, the `rust-patterns` skill, and project-agnostic versions of the original agents.

---

## Important Notes

### Context Window Management

**Critical:** Don't enable all MCPs at once. Your 200k context window can shrink to 70k with too many tools enabled.

Rule of thumb:
- Have 20-30 MCPs configured
- Keep under 10 enabled per project
- Under 80 tools active

Use `disabledMcpServers` in project config to disable unused ones.

### Customization

These configs are defaults. You should:
1. Start with what resonates
2. Modify for your stack
3. Remove what you don't use
4. Add your own patterns

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=TahaGPT/LEGENDS&type=Date)](https://star-history.com/#TahaGPT/LEGENDS&Date)

---

## Links

- **Repository:** [TahaGPT/LEGENDS](https://github.com/TahaGPT/LEGENDS)
- **Original project:** [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)
- **Shorthand Guide (Start Here):** [The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)
- **Longform Guide (Advanced):** [The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)

---

## License

MIT. See [LICENSE](LICENSE). Use freely, modify as needed, contribute back if you can.

---

**Star this repo if it helps. Read both guides. Build something great.**
