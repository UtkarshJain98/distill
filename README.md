# distill
[![Anthropic Published](https://img.shields.io/badge/Anthropic-Official%20Published-ff6b35?logo=data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cGF0aCBkPSJNMTIgMkw0IDIwaDQuNUwxMiA4bDMuNSAxMkgyMEwxMiAyeiIgZmlsbD0id2hpdGUiLz48L3N2Zz4=)](https://claude.com/plugins)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blue)](https://claude.com/plugins)

<img width="3584" height="1184" alt="Gemini_Generated_Image_3n1r983n1r983n1r (1)" src="https://github.com/user-attachments/assets/aaefdcc5-1a18-4802-ab2f-4e9a8fa30d7a" />


A Claude Code plugin that distills your codebase into context files. Invest tokens once, save them on every future session.

## Why

On large codebases, Claude Code spends significant tokens reading source files just to orient itself before starting real work. This cost is paid every session, by every team member.

`distill` analyzes your codebase once and produces structured documentation files in `.claude/docs/`. Future sessions reference these files on demand — Claude reads `testing.md` when working on tests, `api.md` when working on endpoints, and nothing when the task doesn't require it.

This also solves onboarding: a new team member (or a new Claude session) gets instant project context without reading hundreds of source files.

## Installation

```bash
/plugin marketplace add utkarsh-jain/distill
/plugin install distill
```

## Quick Start

```bash
# Generate project orientation + architecture overview
/distill:generate

# Generate everything
/distill:generate --all

# Quick, minimal summaries (30-50 lines each)
/distill:generate --lean --all

# Exhaustive docs with extra sections
/distill:generate --deep --all

# Update only stale docs after making changes
/distill:generate --update

# Check which docs need updating (no changes made)
/distill:generate --status
```

## Flags

| Flag | Generates | Description |
|---|---|---|
| *(none)* | `onboard.md`, `architecture.md` | Project orientation + system design |
| `--structure` | `structure.md` | Per-module breakdown with file inventories |
| `--api` | `api.md` | Endpoints, auth, request/response shapes, middleware |
| `--data` | `data.md` | Models, schemas, relationships, migrations |
| `--internals` | `internals.md` | Core logic, key abstractions, state management |
| `--config` | `config.md` | Env vars, config files, feature flags, deploy config |
| `--testing` | `testing.md` | Test stack, how to run, patterns, fixtures, CI pipeline |
| `--all` | all of the above | Full documentation suite |
| `--update` | *(varies)* | Regenerate only files affected by changes since last run |
| `--status` | *(nothing)* | Report which docs are current vs stale |
| `--lean` | *(varies)* | Minimal 30–50 line summaries, reduced analysis |
| `--deep` | *(varies)* | Richer docs with extra sections and deeper analysis |
| `--sequential` | *(varies)* | Force sequential generation (useful behind rate limits or proxies) |
| `--install-hook` | *(nothing)* | Install a git pre-push hook that warns when docs are stale |

Flags combine: `/distill:generate --api --testing` generates both.

`--update` combines with specific flags: `/distill:generate --update --api` updates only `api.md` if stale.

## Depth Profiles

Control how much detail is generated per doc:

| Depth | Lines per doc | Reading budget | Use when |
|---|---|---|---|
| `--lean` | 30–50 | 5–8 files | Quick orientation, token-constrained environments |
| *(default)* | 60–200 | 15–20 files | Standard usage (varies by doc type) |
| `--deep` | 120–500 | 25–35 files | Comprehensive reference, complex codebases |

`--lean` generates only the most critical sections per doc (e.g., onboard gets "What This Is" + "Tech Stack" + "Quick Start" only). `--deep` adds extra sections like "Cross-cutting Concerns" in architecture and "Coverage Map" in testing.

The depth is recorded in each doc's frontmatter, so `--update` preserves it automatically.

Set a team default in `.distill.yml`:

```yaml
default_depth: lean
```

## Output

```
your-project/
├── CLAUDE.md                     ← index section added between markers
└── .claude/
    └── docs/
        ├── onboard.md            — what this is, tech stack, quick start, directory map
        ├── architecture.md       — system design, components, data flow, entry points
        ├── structure.md          — per-module breakdown
        ├── api.md                — API surface
        ├── data.md               — data layer
        ├── internals.md          — core business logic
        ├── config.md             — configuration
        └── testing.md            — test infrastructure
```

**How Claude uses these files:**

1. `CLAUDE.md` is loaded every session. It contains a small index listing available docs.
2. Claude sees the index and reads only the files relevant to the current task.
3. Each doc links to related docs via a "See also" section, so Claude can follow references across topics.
4. A session about fixing a test reads `testing.md`. A session about a new endpoint reads `api.md`. A quick bug fix might read nothing beyond the index.

Token cost is proportional to task relevance, not project size.

## The `--update` Workflow

Each generated file records the git commit it was generated from and which source paths it covers. When you run `--update`:

1. For each existing doc, diffs the current HEAD against its source commit
2. Maps changed files to their covering doc using recorded scope paths
3. Regenerates only stale docs — current ones are skipped with a status report

Recommended workflow:

```bash
# Before pushing, update any stale docs
/distill:generate --update
```

This keeps docs current with minimal token cost — only changed areas are re-analyzed.

## Automatic Staleness Checks

Install a git hook that warns when docs are stale before pushing:

```bash
/distill:generate --install-hook
```

This adds a lightweight pre-push hook that checks doc freshness by comparing git commit hashes in frontmatter against HEAD. If any docs are stale, the push is blocked with a reminder to run `--update`. The hook does not invoke Claude or spend tokens. Use `git push --no-verify` to bypass.

## Team Configuration

Create `.distill.yml` in your project root to customize behavior for your team:

```yaml
# Default depth profile (lean, default, deep)
default_depth: default

# Default flags when none are provided
default_flags:
  - --api
  - --testing

# Extra directories to ignore during analysis
ignore:
  - generated/
  - third_party/

# Override line targets per file
max_lines:
  onboard: 120
  api: 300

# Custom path mappings for --update staleness detection
# Use when your project doesn't follow standard directory conventions
scope_overrides:
  api:
    - server/routes/
    - server/handlers/
  data:
    - server/db/
    - prisma/

# Patterns to exclude from secret scanning (regex)
# Use when test fixtures or docs legitimately contain token-like strings
secret_scan_allowlist:
  - "sk-test-fake-key-\\w+"
  - "AKIA_EXAMPLE_\\w+"
```

Commit this file so the whole team gets consistent behavior.

## Monorepo Support

`distill` detects monorepo configurations (npm/yarn/pnpm workspaces, Lerna, Nx, Turborepo, Cargo workspaces, Gradle multi-project). When detected:

- `onboard.md` includes a package index with one-line descriptions
- Other docs clearly delineate which package each section covers
- For large monorepos (>20 packages), the plugin suggests running from individual package directories for more focused output

## Preserving Hand-Written Content

The plugin manages content between `<!-- distill:start -->` and `<!-- distill:end -->` markers in `CLAUDE.md`. Everything outside the markers is untouched.

Files in `.claude/docs/` are fully managed by the plugin and are overwritten on regeneration. Add custom context in separate files or in `CLAUDE.md` outside the markers.

## How It Works

The plugin is a Claude Code skill — a structured prompt that instructs Claude to:

1. Identify the project type, language, and framework
2. Map the directory structure and detect monorepo layouts
3. Read strategically: entry points, type definitions, config files, and structural files (not every source file)
4. When generating 3+ docs, spawn parallel agents — one per doc — for faster execution
5. Generate concise, accurate documentation with verified file paths
6. Scan generated output for accidentally included secrets before writing files
7. Record source commit and scope paths in frontmatter for incremental updates
8. Update the root `CLAUDE.md` index

The analysis is thorough but targeted — it reads the minimum files needed to produce accurate docs, using language-specific strategies (e.g., reading barrel files in TypeScript, `__init__.py` in Python, interface definitions in Go).

## License

MIT
