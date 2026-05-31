# Omni-FI CI/CD Workflow Conventions

Canonical house style for GitHub Actions across all `omni-fi-app` repositories. New repos should start from the **org workflow templates** (Actions → New workflow → "Omni-FI CI (Bun)" / "Omni-FI CodeQL"); existing repos should match this document. Templates live in [`workflow-templates/`](./workflow-templates).

> Why this exists: uniform workflows let us require **one** status-check name (`CI`) org-wide in branch rulesets, keep supply-chain risk down via SHA pinning, and make every repo's CI legible at a glance. This mirrors the standardization applied across the Tradable org.

## Repo stacks (the conventions adapt per stack)

| Repo | Stack | Package manager | CI runtime |
|---|---|---|---|
| `omni-fi-core` | Python / Django | `pip` (`requirements.txt`) | Python |
| `omni-fi-link` | Link Widget (Bun) | `bun` | Bun + Node |
| `omni-fi-react-link` | React SDK (Bun) | `bun` | Bun + Node |

What must be **uniform** is the *structure* (job name, permissions, concurrency, SHA-pinning, Dependabot, CodeQL strategy). The *phases* differ by stack (Python lint/type/test vs Bun lint/build/test).

## Core rules

| Rule | Value |
|------|-------|
| CI job id **and** name | `CI` (so the required status check is `CI` everywhere) |
| CodeQL job name | `Analyze` — **no `strategy.matrix`** (matrix renames the check to `Analyze (language)`) |
| Bun (where applicable) | pinned `bun-version: 1.3.14` |
| Node (Bun repos) | pinned via `.nvmrc` (org standard `24`); CI sets up Node **and** Bun |
| Action pinning | Pin every `uses:` to a full **commit SHA** + trailing `# vX.Y.Z` comment. Never a bare tag. |
| Least privilege | CI jobs declare `permissions: contents: read`. CodeQL declares `actions: read` + `contents: read` + `security-events: write`. |
| Concurrency | Every CI/CodeQL workflow has a `concurrency` block keyed on `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`. (Do **not** add this to agent workflows like `claude.yml` — it would cancel in-flight runs.) |

## Pinned action SHAs (current)

Bump via Dependabot (`github-actions` ecosystem, enabled in every repo). Keep the version comment in sync.

| Action | SHA | Version |
|--------|-----|---------|
| `actions/checkout` | `de0fac2e4500dabe0009e67214ff5f5447ce83dd` | v6.0.2 |
| `actions/setup-node` | `48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e` | v6.4.0 |
| `oven-sh/setup-bun` | `0c5077e51419868618aeaa5fe8019c62421857d6` | v2.2.0 |
| `github/codeql-action/*` | `7211b7c8077ea37d8641b6271f6a365a22a5fbfa` | v4.36.0 |
| `anthropics/claude-code-action` | `787c5a0ce96a9a6cfb050ea0c8f4c05f2447c251` | v1 |

## Bun + Node together (Bun repos)

Bun is the package manager and script runner, but the JS toolchains (Vite, eslint, etc.) are Node programs — `bun run vite` executes `node_modules/.bin/vite`, whose `#!/usr/bin/env node` shebang runs it under the runner's **system Node**. So **CI sets up Node before Bun**, pinned via `.nvmrc` (org standard `24`). Every Bun repo commits a `.nvmrc`.

## Step / phase naming

Steps that run commands are **named**, action-first, **no `Run ` prefix**:
- Bun repos: `Install dependencies`, `Lint`, `Build`, `Test`.
- Python repos: `Install dependencies`, `Lint`, `Type-check`, `Test`.
- Multi-package repos disambiguate with a parenthetical target and qualify **all** instances: `Test (sdk)`, `Test (widget)`.
- Don't mix `Test` / `Run tests` / `Test Step` — pick `Test` (+ qualifier).

## CodeQL: default vs advanced (important)

GitHub offers two mutually-exclusive code-scanning modes — **they conflict if both are on**:

- **Default setup** (one-click, no workflow file): zero-maintenance, auto-updates languages. **Preferred for public repos** (free) — e.g. `omni-fi-react-link` uses this.
- **Advanced setup** (a `codeql.yml` workflow): needed when you want the private-repo `$0` gate or custom config. Use the `workflow-templates/codeql.yml` template — matrix-free `Analyze`, `category` set, SHA-pinned, and gated:

  ```yaml
  if: ${{ !github.event.repository.private }}
  ```

  Code scanning is **free on public repos** but a **paid per-committer** feature on private ones; the gate skips it (neutral, `$0`) on private repos and auto-activates if the repo goes public. `omni-fi-core` uses this (Python, gated).

**Never run advanced + default together** — the advanced `Analyze` job fails when default setup is enabled. Pick one per repo.

## Package.json scripts (Bun repos)

Keep scripts runner-agnostic — no `npm`/`npx` baked in. Compose with `bun run <script>`; invoke local bins bare (`vite build`, not `npx vite build`). Set `packageManager` to `bun@1.3.14`.

## Dependabot

Every repo ships `.github/dependabot.yml` with the `github-actions` ecosystem (keeps pinned SHAs current) plus the package ecosystem for the stack (`bun` or `pip`), covering every directory that has a lockfile/manifest. Use unquoted scalars and `commit-message: { prefix: chore, include: scope }` (renders `chore(deps): …`). Enable Dependabot **security updates** in repo settings.

## Branch model: `main` + `staging`

Work flows **feature → `staging` → `main`**. `main` is the (eventual) production branch — for now a placeholder, but **protected as if production**. `staging` is the active integration branch and is **protected just as heavily**, so everything is validated before it reaches `main`. Practical consequences:

- **CI + CodeQL trigger on BOTH** branches: `on: push` / `pull_request` with `branches: [main, staging]`.
- **Dependabot targets `staging`** (`target-branch: staging`) so dependency updates land on staging, pass CI there, and promote to main — never landing on main unvalidated. For monorepo/placeholder repos where the real manifests live on `staging` (e.g. `omni-fi-link`), Dependabot also *reads* manifests from staging.
- Repos that don't yet have a `staging` branch (e.g. `omni-fi-react-link`) should create one for the promotion flow, or target `main` until they do.

## Branch rulesets (per repo — on BOTH `main` and `staging`)

Each repo has a "Protect Main" **and** a "Protect Staging" ruleset, both carrying the same heavy protection: **required status check `CI`**, required PR review (+ code-owner), **Copilot code review**, `required_linear_history`, no-force-push (`non_fast_forward`), and no-deletion. The uniform `CI` job name is what makes a single required-check name work across both branches and all repos.

(`required_linear_history` requires squash/rebase merges — confirm the repo's merge strategy before enabling.)
