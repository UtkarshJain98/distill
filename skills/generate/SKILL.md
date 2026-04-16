---
name: generate
description: >
  Analyze the current codebase and generate context documentation files in .claude/docs/.
  These files give Claude Code instant project context without reading source code —
  saving tokens on every future session. Supports flags: --structure, --api, --data,
  --internals, --config, --testing, --all, --update, --status, --lean, --deep,
  --sequential, --install-hook. Default (no flag) generates a project orientation
  with architecture overview and onboarding guide.
user_invocable: true
---

# distill: generate

You are executing the `distill` plugin. Your job is to analyze this codebase and produce
structured, accurate context files that future Claude Code sessions will reference on demand.

**Core principle: accuracy over completeness.** These docs will be trusted by Claude in
every future session. A wrong file path, a hallucinated function name, or an outdated
architecture description is worse than a gap. When uncertain, leave it out or mark it
explicitly with `[unverified]`.

**Second principle: conciseness.** Every line in these files costs tokens across many
sessions. If a sentence doesn't help Claude make better decisions or navigate faster,
cut it.

---

## Phase 1: Parse Arguments

The user invoked: `/distill:generate $ARGUMENTS`

Valid flags:

| Flag | Effect |
|---|---|
| *(none)* | Generate `onboard.md` + `architecture.md`, update root `CLAUDE.md` index |
| `--structure` | Deeper per-module structural breakdown |
| `--api` | API surface: endpoints, auth, request/response shapes |
| `--data` | Data layer: models, schemas, relationships, migrations |
| `--internals` | Core business logic, algorithms, internal module APIs |
| `--config` | Configuration: env vars, config files, feature flags, deploy |
| `--testing` | Test infrastructure: frameworks, patterns, how to run |
| `--all` | Generate everything (default set + all flags above) |
| `--update` | Regenerate only files affected by changes since last generation |
| `--status` | Report which files are current vs stale, without regenerating |
| `--lean` | Minimal 30–50 line summaries per file, reduced reading budget |
| `--deep` | Richer docs with extra sections, increased reading budget and line targets |
| `--sequential` | Force sequential doc generation even when generating 3+ docs |
| `--install-hook` | Install a git pre-push hook that warns when docs are stale |

Flags can be combined: `--api --testing` generates both. `--update --api` updates only
`api.md` if stale.

`--lean` and `--deep` are mutually exclusive. If both are provided, report an error and
stop. These combine with all other flags: `--lean --api` generates a lean `api.md`,
`--deep --all` generates deep versions of everything.

If no flags and no arguments, generate the **default set**: `onboard.md` and
`architecture.md`, plus the root `CLAUDE.md` index.

---

## Phase 1b: Handle `--install-hook`

If `--install-hook` is set, install a git pre-push hook and stop. Do not proceed to
analysis or generation.

1. Check if `.git/` exists. If not, report an error: "Not a git repository. The
   pre-push hook requires git." Stop.

2. Check if `.git/hooks/pre-push` already exists:
   - If it exists and contains the string `distill`, report: "distill pre-push
     hook is already installed." Stop.
   - If it exists and does NOT contain `distill`, warn: "A pre-push hook already
     exists at `.git/hooks/pre-push`. To avoid overwriting it, manually append the
     distill hook." Then print the hook script below for the user to copy. Stop.

3. If no pre-push hook exists, write the following script to `.git/hooks/pre-push`
   and make it executable (`chmod +x`):

```bash
#!/usr/bin/env bash
# distill pre-push hook
# Checks if any generated docs in .claude/docs/ are stale.
# Does NOT invoke Claude or spend tokens — only checks git commit hashes.
# Bypass: git push --no-verify

set -euo pipefail

if [ ! -d ".claude/docs" ]; then
  exit 0
fi

if ! git rev-parse --git-dir > /dev/null 2>&1; then
  exit 0
fi

stale_docs=()

for doc in .claude/docs/*.md; do
  [ -f "$doc" ] || continue
  commit=$(awk '/^---$/{n++; next} n==1 && /^source_commit:/{sub(/^source_commit:[[:space:]]*/, ""); gsub(/["'"'"']/, ""); print; exit}' "$doc")
  if [ -z "$commit" ] || [ "$commit" = "no-git" ]; then
    continue
  fi
  if ! git cat-file -e "$commit^{commit}" 2>/dev/null; then
    stale_docs+=("$(basename "$doc") (source commit $commit no longer exists)")
    continue
  fi
  changes=$(git diff "$commit"..HEAD --name-only 2>/dev/null | head -1)
  if [ -n "$changes" ]; then
    stale_docs+=("$(basename "$doc")")
  fi
done

if [ ${#stale_docs[@]} -gt 0 ]; then
  echo ""
  echo "distill: Some docs in .claude/docs/ may be stale:"
  for doc in "${stale_docs[@]}"; do
    echo "  - $doc"
  done
  echo ""
  echo "Run '/distill:generate --update' to refresh them."
  echo "To push anyway: git push --no-verify"
  echo ""
  exit 1
fi
```

4. Report success: "Installed pre-push hook at `.git/hooks/pre-push`. Pushes will be
   blocked when docs are stale. Use `git push --no-verify` to bypass."

Then stop. Do not proceed to Phase 2 or any further phases.

---

## Phase 2: Pre-flight Checks

Before doing any analysis, verify the environment:

1. **Git check**: run `git rev-parse --short HEAD`. If this fails, the project is not a
   git repository. Warn the user that `--update` will not work (no commit tracking), but
   proceed with generation. Use `"no-git"` as the source_commit value.

2. **Project root**: confirm you're in the project root by checking for common root
   indicators (`.git/`, `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`,
   `pom.xml`, `build.gradle`, `Makefile`, `README.md`). If none are found, warn the
   user that this may not be a project root and ask whether to proceed.

3. **Existing docs**: check if `.claude/docs/` exists and list any files in it. This
   informs whether this is a first run or a regeneration.

4. **Config file**: check for `.distill.yml` in the project root. If it exists, read
   it and apply overrides (see Config File section below).

5. **Gitignore check**: check if `.claude/` or `.claude/docs/` is gitignored
   (`git check-ignore -q .claude/docs`). If it is, warn the user: the generated docs
   won't be committed or shared with the team unless they update `.gitignore`.

6. **Output directory**: ensure `.claude/docs/` exists. Create it if it doesn't.

---

## Phase 3: Handle `--status` Mode

If `--status` is set, do NOT generate anything. Instead:

1. List every file in `.claude/docs/`.
2. For each file, read its YAML frontmatter and extract `source_commit` and `generated_at`.
3. Run `git log --oneline <source_commit>..HEAD -- .` to count commits since generation.
4. Report a table:

```
File              Generated    Commits Since    Status
onboard.md        2025-01-15   3                stale
architecture.md   2025-01-15   3                stale
api.md            2025-01-18   0                current
testing.md        2025-01-10   12               stale
```

Then stop. Do not proceed to analysis or generation.

---

## Phase 4: Handle `--update` Mode

If `--update` is set:

1. Read each existing file in `.claude/docs/`. Extract the `source_commit` and
   `scope_paths` from the YAML frontmatter.

2. For each file, run `git diff <source_commit>..HEAD --name-only` to get the list
   of changed files since that doc was generated.

3. Determine staleness using the `scope_paths` field from frontmatter. Each generated
   doc records which source paths it was derived from. If any changed file falls under
   those paths, the doc is stale and needs regeneration.

   If a doc has no `scope_paths` (generated before this feature existed), fall back to
   the heuristic mapping in the table below:

   | Doc | Heuristic path patterns |
   |---|---|
   | `onboard.md` | `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `Makefile`, `Dockerfile`, `docker-compose*`, `README.md`, new/deleted top-level dirs |
   | `architecture.md` | New/deleted directories under src/ or lib/ or app/, changes to entry point files, changes to dependency manifests |
   | `structure.md` | Any new/deleted/renamed files or directories |
   | `api.md` | Files in `**/routes/**`, `**/controllers/**`, `**/api/**`, `**/endpoints/**`, `**/handlers/**`, `**/graphql/**`, `**/proto/**` |
   | `data.md` | Files in `**/models/**`, `**/schemas/**`, `**/migrations/**`, `**/entities/**`, `**/prisma/**`, `**/typeorm/**`, `**/sequelize/**` |
   | `internals.md` | Files in `**/services/**`, `**/core/**`, `**/lib/**`, `**/domain/**`, `**/engine/**` not matched by api or data |
   | `config.md` | `.env*`, `**/config/**`, `**/settings/**`, `*.yml`, `*.yaml`, `*.toml`, CI/CD configs, `Dockerfile*`, `docker-compose*` |
   | `testing.md` | Files in `**/test/**`, `**/tests/**`, `**/spec/**`, `**/__tests__/**`, test config files (`jest.config*`, `pytest.ini`, etc.) |

4. If `--update` is combined with specific flags (e.g. `--update --api`), only check
   and regenerate those specific files.

5. If `--update` is alone, check all existing docs and regenerate only stale ones.

6. If a doc doesn't exist yet, generate it fresh.

7. **Depth preservation**: if `--update` is used without `--lean` or `--deep`, preserve
   the depth from each existing doc's frontmatter `depth` field. If `--update` is combined
   with a depth flag (e.g. `--update --deep`), regenerate stale docs at the new depth
   and update their frontmatter accordingly.

8. Report which docs were updated, which were skipped (still current), and why.

---

## Phase 5: Analyze the Codebase

This phase builds your mental model before writing anything. Be methodical.

### Step 5a: Identify the project

1. List the root directory contents.
2. Read the primary manifest file (the first one found among: `package.json`,
   `Cargo.toml`, `go.mod`, `pyproject.toml`, `pom.xml`, `build.gradle`,
   `build.gradle.kts`, `Gemfile`, `composer.json`, `CMakeLists.txt`, `Makefile`).
3. Read `README.md` if it exists (first 100 lines only for large files).

From these, determine:
- Primary language and framework
- Project type (web app, API service, CLI tool, library, monorepo, mobile app, etc.)
- Build and dependency tools

### Step 5b: Monorepo detection

Check for monorepo indicators:
- `workspaces` field in `package.json`
- `lerna.json`, `nx.json`, `turbo.json`, `pnpm-workspace.yaml`
- `Cargo.toml` with `[workspace]` section
- Multiple `go.mod` files
- `settings.gradle` with multiple `include` statements

If this is a monorepo:
- Identify the top-level packages/services/apps
- Treat each package as a semi-independent unit
- In generated docs, clearly delineate which package each section refers to
- Consider generating sub-indexes per package if there are more than 5 packages

### Step 5c: Map the structure

Get the directory tree to 2 levels deep. For monorepos or very large projects (>50
top-level items), adapt: go deeper in important directories, shallower in generated
or vendor directories.

Ignore: `node_modules/`, `vendor/`, `target/`, `build/`, `dist/`, `.git/`,
`__pycache__/`, `.next/`, `.nuxt/`, `venv/`, `.venv/`, `.tox/`, `coverage/`,
`.nyc_output/`, `.terraform/`.

### Step 5d: Read strategically

You have a limited context window. Spend it wisely.

**Always read** (if they exist):
- The primary manifest file (package.json, Cargo.toml, etc.)
- The main entry point (main.ts, index.ts, app.py, main.go, Main.java, etc.)
- Root configuration files (tsconfig.json, .eslintrc, pyproject.toml, etc.)

**Read for specific flags:**

| Flag | What to read |
|---|---|
| `--api` | Route definitions, controller files, API middleware, OpenAPI/Swagger specs |
| `--data` | Model/entity definitions, schema files, migration files, ORM config |
| `--internals` | Service layer files, core module entry points, key type definitions |
| `--config` | All config files, .env.example, CI/CD configs, Dockerfiles |
| `--testing` | Test config, 1-2 representative test files, fixture definitions |
| `--structure` | Module index/barrel files, package manifests for sub-packages |

**Strategy by language:**

- **TypeScript/JavaScript**: read barrel files (`index.ts`), type definition files
  (`*.d.ts`, `types.ts`), route registrations. Use grep for `export` to find public APIs.
- **Python**: read `__init__.py` files, type hints in function signatures, FastAPI/Flask
  route decorators. Use grep for `@app.route`, `@router`, class definitions.
- **Go**: read `main.go`, `cmd/` entry points, interface definitions. Use grep for
  `type.*interface`, `func.*Handler`.
- **Java/Kotlin**: read `Application.java`, `@RestController` classes, entity classes.
  Use grep for `@Entity`, `@RestController`, `@Service`.
- **Rust**: read `main.rs`, `lib.rs`, `mod.rs` files, trait definitions.
- **Other languages**: adapt the strategy. Focus on entry points, type definitions,
  and public API surfaces.

**Reading budget** (per generated doc):

| Depth | Files per doc | Strategy |
|---|---|---|
| `--lean` | 5–8 | Read only manifests, entry points, and 1–2 structural files. Rely heavily on grep. Skip sub-files within modules. |
| default | 15–20 | Balanced reading of key files per flag type. |
| `--deep` | 25–35 | Follow imports one level further. Read representative files within each module, not just barrel/index files. |

If you're exceeding the budget, use grep to find patterns instead of reading entire files.

### Step 5e: Track what you read

As you read files, keep a mental list of which source paths informed each doc you're
going to generate. You'll record these in the `scope_paths` frontmatter field so
`--update` can track staleness accurately.

---

## Phase 6: Generate Files

### Execution strategy

When generating multiple docs, use parallel sub-agents for efficiency:

**Sequential** (default for 1–2 docs, or when `--sequential` is set): generate docs
directly in the current context, one at a time.

**Parallel** (for 3+ docs, unless `--sequential` or `--lean` is set): spawn one
sub-agent per doc using the Agent tool.

Parallel generation procedure:

1. **Complete Phase 5 (analysis) fully first.** The analysis results are needed by all docs.
2. **Prepare a context brief** containing:
   - Project type, language, framework (from Step 5a)
   - Monorepo layout if applicable (from Step 5b)
   - Directory tree (from Step 5c)
   - The `source_commit` hash
   - The depth profile (`lean`, `default`, or `deep`)
   - The full list of docs being generated (so each agent can build accurate See also links)
3. **Spawn one sub-agent per doc**, giving each:
   - The context brief
   - The specific doc template instructions from this skill
   - The reading strategy for that doc type
   - Instructions to write the file to `.claude/docs/<filename>`
4. **Wait for all sub-agents to complete.**
5. **Run Phase 8 (verification) in the main context** across all generated files. This
   must happen centrally because it verifies cross-references and runs the secret scan
   across all output.

**Constraints:**
- Each sub-agent independently reads the source files it needs. The per-doc reading
  budget still applies per agent.
- Sub-agents must NOT modify files written by other sub-agents.
- If a sub-agent fails, report the failure and continue with remaining docs.
- The main context handles Phase 7 (CLAUDE.md index), Phase 8 (verification), and
  Phase 9 (report) after all sub-agents complete.

**When NOT to parallelize** (use sequential even without `--sequential`):
- Generating only 1–2 docs: overhead isn't worth it.
- `--lean` mode: docs are small enough that parallelization overhead exceeds the time saved.

**Prime parallelization case**: `--deep --all` generates 8 docs at 200–500 lines each
with an expanded reading budget. Always parallelize this unless `--sequential` is set.

---

### Frontmatter

Each generated file MUST begin with this YAML frontmatter:

```yaml
---
generated_by: distill
doc_version: 1
depth: default
source_commit: <output of git rev-parse --short HEAD>
generated_at: <YYYY-MM-DD>
scope_paths:
  - src/routes/
  - src/middleware/auth.ts
  - src/api/
---
```

The `depth` field records which depth profile was used (`lean`, `default`, or `deep`).
This allows `--update` to preserve the depth when regenerating stale docs.

The `doc_version` field tracks the format version. This allows future versions of the
plugin to detect and handle docs generated by older versions gracefully.

The `scope_paths` field lists the directories and files that were analyzed to produce
this doc. This enables precise staleness detection in `--update` mode. List directories
(with trailing slash) for broad coverage, and specific files for targeted references.

### Verification requirement

After generating each file, **verify every file path you referenced**. Use glob or
ls to confirm the files exist. If a path doesn't exist, remove the reference or fix it
before writing the file. Do not write paths from memory — confirm them.

### Cross-document references

Every generated doc MUST end with a `## See also` section that links to other
generated docs covering related topics. This gives Claude (and humans) lateral
navigation between docs without loading the full index.

**Rules:**

1. Only reference docs that actually exist in `.claude/docs/`. If you're generating
   a subset of docs, only link to the ones present.
2. List 2–4 genuinely related docs. Not every doc — only ones with meaningful overlap.
3. Each reference is the filename (relative path within `.claude/docs/`) and a one-line
   explanation of what related content it contains.
4. Format:
   ```markdown
   ## See also

   - [architecture.md](architecture.md) — component relationships for modules described above
   - [api.md](api.md) — endpoint definitions that call these services
   ```

**Reference affinity map** (use as guidance, override when actual content suggests
different relationships):

| Doc | Typically references |
|---|---|
| `onboard.md` | `architecture.md`, `structure.md` |
| `architecture.md` | `onboard.md`, `structure.md`, `internals.md` |
| `structure.md` | `architecture.md`, `internals.md`, `api.md`, `data.md` |
| `api.md` | `architecture.md`, `data.md`, `internals.md`, `testing.md` |
| `data.md` | `architecture.md`, `api.md`, `internals.md` |
| `internals.md` | `architecture.md`, `api.md`, `data.md` |
| `config.md` | `onboard.md`, `architecture.md`, `testing.md` |
| `testing.md` | `onboard.md`, `config.md`, `api.md` |

---

### `onboard.md` — Project Orientation *(default, --all)*

The most frequently referenced file. Loaded at session start for orientation. Ruthlessly
concise — this is read on every session.

**Sections:**

**1. What This Project Is**
One paragraph. What it does, who it's for, where it fits. No marketing language.

**2. Tech Stack**
Bullet list. Language version, framework, key dependencies (only the ones that affect
how you write code), database, infrastructure. Include versions only when they matter
(e.g., React 18 vs 19 changes how you write components).

**3. Quick Start**
Exact, copy-pasteable commands for:
- Install dependencies
- Run the development server / build
- Run tests
- Any required setup (database, env file, etc.)

If different team members use different setups (e.g., Docker vs local), document both
briefly.

**4. Directory Map**
Every top-level directory and significant second-level directories with one-line
descriptions. Format:

```
src/              — application source
  src/api/        — REST endpoint handlers
  src/models/     — database entities
  src/services/   — business logic
tests/            — test suites (mirrors src/ structure)
scripts/          — build and deployment tooling
infra/            — Terraform / CloudFormation configs
```

**5. Key Conventions**
Only conventions that actually exist in the codebase and aren't obvious. Examples:
- Naming patterns for files, functions, or types
- Import ordering conventions
- Where new features should be added
- Code organization patterns (e.g., "each feature gets its own directory under src/features/")

Do NOT invent or prescribe conventions. Only document what's observable.

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "What This Project Is" + "Tech Stack" + "Quick Start" only |
| default | 80–150 | All 5 sections |
| `--deep` | 150–250 | All 5 sections + "Development Workflows" (common tasks like adding a feature, running a migration, deploying) |

End with a `## See also` section per the cross-document reference rules above.

---

### `architecture.md` — System Design *(default, --all)*

**Sections:**

**1. System Overview**
2-3 sentences. What the system does, its main responsibilities, where it sits in a
larger ecosystem (if applicable — e.g., "this is the auth service that other services
call for token validation").

**2. Component Map**
Major components/modules, each with a one-line responsibility and its root path:
```
API Layer (src/api/)          — HTTP request handling, validation, serialization
Service Layer (src/services/) — Business logic, orchestration
Data Layer (src/models/)      — Database entities, queries, migrations
Workers (src/workers/)        — Background job processing
```

**3. Component Relationships**
How components depend on each other. Show the dependency direction. Use a text diagram
for complex systems:
```
[HTTP Request]
     ↓
[Middleware] → auth, rate-limit, logging
     ↓
[Router] → maps to handler
     ↓
[Handler] → validates input, calls service
     ↓
[Service] → business logic, calls repository
     ↓
[Repository] → database queries
     ↓
[Database]
```

**4. Data Flow**
Trace one or two representative operations through the system. Name the specific files
involved at each step.
```
Creating a user:
  POST /api/users → src/api/routes/users.ts → src/api/handlers/createUser.ts
    → src/services/userService.ts (validates, hashes password)
    → src/models/user.ts (inserts into database)
    → src/events/userCreated.ts (emits event)
```

**5. Key Design Decisions**
Patterns in use (MVC, hexagonal, CQRS, event-driven, etc.) and notable architectural
choices that are *visible in the code*. If the reason for a choice is evident (e.g., a
comment or ADR), include it. Otherwise, just state the pattern without speculating on why.

**6. Entry Points**
Every way the system can be invoked, with file paths:
```
HTTP server:     src/index.ts → starts Express on :3000
CLI:             src/cli.ts → commander-based CLI
Worker:          src/worker.ts → processes jobs from Redis queue
Cron:            src/cron.ts → scheduled tasks via node-cron
```

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "System Overview" + "Component Map" only |
| default | 100–200 | All 6 sections |
| `--deep` | 200–350 | All 6 sections + "Cross-cutting Concerns" (logging, monitoring, error handling patterns) + "Performance Characteristics" (known bottlenecks, caching strategy) |

End with a `## See also` section per the cross-document reference rules above.

---

### `structure.md` — Module Breakdown *(--structure, --all)*

Deeper than the directory map in `onboard.md`. Goes into each major module/package.

**For each significant module:**
```
## src/services/

Business logic layer. Each service is a class/module that encapsulates one domain area.

Files:
- userService.ts    — user CRUD, password management, profile updates
- authService.ts    — login, token generation, token refresh, logout
- emailService.ts   — email template rendering, sending via SendGrid
- paymentService.ts — Stripe integration, subscription management

Dependencies: imports from src/models/, src/lib/
Depended on by: src/api/handlers/
```

For monorepos, organize by package/service with clear boundaries noted.

**Depth targets:**

| Depth | Lines | Content |
|---|---|---|
| `--lean` | 30–50 | Module list with one-line descriptions per module. No per-file breakdown. |
| default | 100–300 | Full per-module breakdown with file inventories |
| `--deep` | 200–500 | Full breakdown + internal dependency graph between modules + key export surfaces |

End with a `## See also` section per the cross-document reference rules above.

---

### `api.md` — API Surface *(--api, --all)*

**Sections:**

**1. Overview**
API type (REST, GraphQL, gRPC, WebSocket, CLI, library), base URL pattern, versioning.

**2. Authentication & Authorization**
How auth works, where enforced, what tokens/credentials are expected, which endpoints
are public vs protected.

**3. Endpoint Catalog**
Every endpoint, grouped logically. Format:
```
## Users

POST   /api/v1/users           — create user
  Body: { email, name, password }
  Returns: { id, email, name, created_at }

GET    /api/v1/users/:id        — get user by ID
  Auth: bearer token
  Returns: { id, email, name, profile }

GET    /api/v1/users             — list users (paginated)
  Query: ?page=1&limit=20&sort=created_at
  Returns: { data: User[], meta: { total, page, pages } }
```

Include request shape and response shape only when non-obvious. For CRUD endpoints
following standard patterns, a one-liner suffices.

For GraphQL: document queries, mutations, subscriptions with their input/return types.
For gRPC: document services and RPCs.

**4. Middleware Chain**
What middleware runs, in what order. Which endpoints it applies to.

**5. Error Handling**
Standard error format, common error codes, how validation errors are returned.

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "Overview" + endpoint list (method + path + one-liner only, no shapes) |
| default | varies | All 5 sections, complete but concise |
| `--deep` | +50% of default | All 5 sections + full request/response JSON shapes for every endpoint + error response catalog |

End with a `## See also` section per the cross-document reference rules above.

---

### `data.md` — Data Layer *(--data, --all)*

**Sections:**

**1. Overview**
Database type(s), ORM/query builder, connection configuration approach.

**2. Model Catalog**
Every model/entity with key fields (not every field — focus on primary keys, foreign
keys, unique constraints, and fields that affect behavior):
```
## User (src/models/user.ts)
- id: UUID (PK)
- email: string (unique, indexed)
- password_hash: string
- role: enum(admin, user, viewer)
- created_at, updated_at: timestamps
→ has_many: Post, Comment
→ has_one: Profile
```

**3. Relationships**
Text diagram:
```
User 1──* Post *──* Tag
  |                  |
  1                  *
  |               PostTag (join)
Profile
```

**4. Migrations**
How migrations work (tool, location, naming). Current state if notable.

**5. Data Access Patterns**
Repository pattern? Direct ORM? Raw SQL? Query builders? Caching layer?
Name the pattern and where it's implemented.

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "Overview" + model list (name, PK, key relationships only) |
| default | 100–200 | All 5 sections |
| `--deep` | 200–350 | All 5 sections + "Key Queries" (complex or performance-critical queries with explanation) |

End with a `## See also` section per the cross-document reference rules above.

---

### `internals.md` — Core Logic *(--internals, --all)*

**Sections:**

**1. Module Overview**
Core business logic modules and what domain each one owns.

**2. Key Abstractions**
Interfaces, base classes, core types that the system builds on. File paths included.

**3. Processing Pipelines**
Multi-step processes traced through the code with files at each step.

**4. State Management**
How app state is managed: global singletons, request context, stores, caches, sessions.

**5. Internal APIs**
How modules communicate. Key function signatures that are depended on across modules.

**6. Concurrency**
Threading model, async patterns, worker pools, queue consumers. Only if applicable.

**7. Notable Algorithms**
Non-trivial business rules or algorithms that take significant effort to understand from
code alone. Include the file path and a concise explanation of what the algorithm does
and why.

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "Module Overview" + "Key Abstractions" only |
| default | 100–250 | All 7 sections |
| `--deep` | 200–400 | All 7 sections + "Code Examples" (key function signatures and usage patterns) |

End with a `## See also` section per the cross-document reference rules above.

---

### `config.md` — Configuration *(--config, --all)*

**Sections:**

**1. Environment Variables**
Table format:
```
| Variable        | Purpose                  | Required | Default   |
|-----------------|--------------------------|----------|-----------|
| DATABASE_URL    | PostgreSQL connection     | yes      | —         |
| PORT            | HTTP listen port          | no       | 3000      |
| LOG_LEVEL       | Logging verbosity         | no       | info      |
| REDIS_URL       | Cache/queue connection    | yes      | —         |
```

Never include actual secret values. Use `<value>` placeholders.

**2. Configuration Files**
Each config file, its format, what it controls.

**3. Feature Flags**
Flag system, how flags are defined/checked, current flags if discoverable.

**4. Build & Deploy**
Build tools, CI/CD pipeline overview, deployment targets. Keep it brief — link to
CI config files rather than reproducing them.

**5. Secrets**
How secrets are provided (env vars, vault, mounted files). Never actual values.

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "Environment Variables" table + "Configuration Files" list only |
| default | 60–150 | All 5 sections |
| `--deep` | 120–250 | All 5 sections + full env var documentation with example values and validation rules |

End with a `## See also` section per the cross-document reference rules above.

---

### `testing.md` — Test Infrastructure *(--testing, --all)*

**Sections:**

**1. Test Stack**
Frameworks, assertion libraries, mocking tools, coverage tools. Versions only if they
matter.

**2. How to Run**
Exact commands:
```
All tests:           npm test
Single file:         npm test -- path/to/file.test.ts
Single test:         npm test -- -t "test name"
With coverage:       npm run test:coverage
Watch mode:          npm run test:watch
```

Adapt for the actual project's tooling (pytest, cargo test, go test, etc.).

**3. Directory Structure**
Where tests live, naming conventions, how test files map to source files.

**4. Patterns**
How tests are structured in this project. Setup/teardown, fixtures, factories,
shared helpers. Describe the patterns actually in use — cite a representative test file.

**5. Mocking & Fixtures**
What's mocked, what hits real dependencies, where fixtures live.

**6. Integration & E2E Tests**
If they exist: how they differ from unit tests, what infrastructure they need,
how to run them separately.

**7. CI Pipeline**
How tests run in CI. Any CI-specific behavior (e.g., different test suites, parallelism,
required checks).

**Depth targets:**

| Depth | Lines | Sections |
|---|---|---|
| `--lean` | 30–50 | "Test Stack" + "How to Run" only |
| default | 80–150 | All 7 sections |
| `--deep` | 150–250 | All 7 sections + "Coverage Map" (which modules have tests, which don't, current coverage %) |

End with a `## See also` section per the cross-document reference rules above.

---

## Phase 7: Update Root CLAUDE.md Index

After generating docs, update the project root `CLAUDE.md` with an index of available
context files.

**If `CLAUDE.md` exists**: look for `<!-- distill:start -->` and `<!-- distill:end -->`.
- If markers found: replace content between them.
- If markers not found: append the block at the end.

**If `CLAUDE.md` does not exist**: create it with only the index block.

**Important**: never modify content outside the markers. The rest of `CLAUDE.md` is
hand-written project instructions.

**Only list files that actually exist** in `.claude/docs/`. Omit entries for files that
weren't generated.

```markdown
<!-- distill:start -->
## Project Context (generated by distill)

Context files for Claude Code sessions. Read on demand — do not load all at once.

- [Onboarding](.claude/docs/onboard.md) — what this project is, tech stack, quick start, directory map
- [Architecture](.claude/docs/architecture.md) — system design, components, data flow, entry points
- [Module Structure](.claude/docs/structure.md) — detailed per-module breakdown
- [API Surface](.claude/docs/api.md) — endpoints, auth, request/response shapes
- [Data Layer](.claude/docs/data.md) — models, schemas, relationships, migrations
- [Core Logic](.claude/docs/internals.md) — business logic, key abstractions, state management
- [Configuration](.claude/docs/config.md) — env vars, config files, feature flags, deploy config
- [Testing](.claude/docs/testing.md) — test stack, how to run, patterns, CI pipeline
<!-- distill:end -->
```

---

## Phase 8: Post-generation Verification

Before reporting to the user, verify:

1. **Every file path referenced in the generated docs exists.** Run a quick check
   against the filesystem. Remove or correct any that don't. This includes verifying
   that every `## See also` link points to a file that was actually generated in
   `.claude/docs/`.
2. **Secret scan.** Grep the generated files for the patterns below. The goal is to
   catch real secrets that leaked from source code into docs, while ignoring false
   positives (pattern names, placeholders like `<your-api-key>`, redacted examples
   like `sk-xxxx...`).

   **Pattern categories to scan for:**

   | Category | Patterns |
   |---|---|
   | API keys / tokens | Strings starting with `sk-`, `pk_live_`, `pk_test_`, `rk_live_`, `ghp_`, `gho_`, `ghs_`, `github_pat_`, `AKIA`, `xoxb-`, `xoxp-`, `xoxs-`, `glpat-`, `pypi-`, `npm_`, `SG.`, `sq0atp-`, `sq0csp-`, `sk_live_`, `sk_test_` |
   | Auth headers | `Bearer [A-Za-z0-9\-._~+/]+=*` or `Basic [A-Za-z0-9+/]+=*` with inline credential values (not placeholders) |
   | Connection strings | URIs matching `://[^:]+:[^@]+@` (credentials in URL), including `mongodb(+srv)?://`, `postgres(ql)?://`, `mysql://`, `redis://`, `amqp://` |
   | Embedded secrets | `password\s*[=:]\s*["'][^"']+["']`, `secret\s*[=:]\s*["'][^"']+["']`, `token\s*[=:]\s*["'][^"']+["']`, `apikey\s*[=:]\s*["'][^"']+["']` (case-insensitive, only when the value is a real string, not a placeholder) |
   | Private keys | `-----BEGIN (RSA\|EC\|DSA\|OPENSSH )?PRIVATE KEY-----` |
   | High-entropy strings | Quoted or assigned strings longer than 40 characters that are purely alphanumeric + `+/=` (base64-like) or hex |

   **Action protocol when a match is found:**

   a. **Triage**: determine if it's a real secret or a false positive. False positives
      include: the pattern name itself (e.g., "strings starting with sk-"), placeholder
      values (`<your-api-key>`, `<value>`), obviously fake examples (`sk-xxxx...`,
      `AKIA_EXAMPLE_KEY`), and values matched by the `.distill.yml`
      `secret_scan_allowlist` (see Config File section).
   b. **Redact**: if real, replace the value with `<REDACTED:category>` — e.g.,
      `<REDACTED:api-key>`, `<REDACTED:connection-string>`, `<REDACTED:private-key>`.
      This is more informative than a bare placeholder.
   c. **Log**: record each redaction for the Phase 9 report.
   d. **Escalate**: if more than 3 secrets are redacted across all generated docs,
      add a warning in the Phase 9 report suggesting the user review their codebase
      for hardcoded secrets.

   **Second pass**: after writing all doc files, run a final sweep across every file
   in `.claude/docs/` to catch secrets that may have been introduced during late
   modifications (e.g., cross-doc references pulling in content).
3. **Line counts are within targets.** If a file significantly exceeds its target,
   it's probably too verbose. Tighten it.
4. **Frontmatter is complete.** Every generated file has `generated_by`, `doc_version`,
   `depth`, `source_commit`, `generated_at`, and `scope_paths`.

---

## Phase 9: Report

Print a summary table. If parallel generation was used, note it:

```
Generated files (4 parallel agents):

  File                Lines   Depth     Scope
  onboard.md          95      default   project root, manifests
  architecture.md     142     default   src/*, entry points
  api.md              178     default   src/api/, src/middleware/

  Updated: CLAUDE.md index (3 entries)

  Review the generated files in .claude/docs/ and commit them to version control.
```

If secrets were redacted, append:
```
  WARNING: Redacted 2 potential secret(s). Search for <REDACTED: in generated files to review.
    config.md: 1x connection-string
    internals.md: 1x api-key
```

If `--update` was used, also report:
```
  Skipped (current):
  testing.md — no changes in test paths since abc1234 (2025-01-18)
```

---

## Config File: `.distill.yml`

If a `.distill.yml` file exists in the project root, apply its settings. This lets
teams customize behavior without passing flags every time.

Supported fields:

```yaml
# Output directory (default: .claude/docs)
output_dir: .claude/docs

# Default depth profile (lean, default, deep)
# Applied when no --lean or --deep flag is passed
default_depth: default

# Default flags when no flags are provided (default: none, which means onboard + architecture)
default_flags:
  - --api
  - --testing

# Directories to always ignore during analysis (merged with built-in ignore list)
ignore:
  - generated/
  - vendor/
  - third_party/

# Maximum line target per file (overrides built-in targets)
max_lines:
  onboard: 120
  api: 300

# Custom scope_paths overrides for --update staleness detection
# Use when your project structure doesn't match standard conventions
scope_overrides:
  api:
    - server/routes/
    - server/handlers/
    - shared/api-types/
  data:
    - server/db/
    - prisma/

# Patterns to exclude from secret scanning (regex)
# Use when test fixtures or documentation legitimately contain token-like strings
secret_scan_allowlist:
  - "sk-test-fake-key-\\w+"
  - "AKIA_EXAMPLE_\\w+"
```

If the config file doesn't exist, use all defaults. Do not create one automatically.

---

## Edge Cases

- **Not a git repository**: skip commit tracking. Set `source_commit: "no-git"`.
  Warn that `--update` and `--status` won't work.
- **Empty or near-empty project**: generate a minimal `onboard.md` noting the project
  appears to be in early stages. Don't generate other docs for directories that
  don't exist yet.
- **Very large monorepo (>20 packages)**: generate a top-level index in `onboard.md`
  listing all packages with one-line descriptions, then suggest the user run
  `--all` from within individual package directories.
- **No manifest file found**: warn the user, then infer what you can from file
  extensions and directory structure.
- **Generated docs already exist and --update not specified**: overwrite without
  prompting. The files are generated artifacts — the frontmatter makes this clear.
- **CLAUDE.md has distill markers but user also hand-edited the generated section**:
  overwrite the section between markers. Content between markers is managed by the plugin.
- **`--install-hook` with an existing non-distill pre-push hook**: do not overwrite.
  Print the hook code and let the user integrate it manually.
