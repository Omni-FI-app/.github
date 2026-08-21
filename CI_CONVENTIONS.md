# Omni-FI CI/CD Workflow Conventions

Canonical house style for GitHub Actions across all `omni-fi-app` repositories. New repos should start from the **org workflow templates** (Actions → New workflow → "Omni-FI CI (Bun)" / "Omni-FI CodeQL"); existing repos should match this document. Templates live in [`workflow-templates/`](./workflow-templates).

> Why this exists: uniform workflows let us require **one** status-check name (`CI`) org-wide in branch rulesets, keep supply-chain risk down via SHA pinning, and make every repo's CI legible at a glance.

## Repo stacks (the conventions adapt per stack)

| Repo | Stack | Package manager | CI runtime |
|---|---|---|---|
| `omni-fi-core` | Python / Django | `pip` (`requirements.txt`) | Python |
| `omni-fi-web` | Next.js portal (Bun) | `bun` | Bun + Node |
| `omni-fi-link` | Link Widget (Bun) | `bun` | Bun + Node |
| `omni-fi-react-link` | React SDK (Bun) | `bun` | Bun + Node |
| `omni-fi-internal` | Internal admin (Bun) | `bun` | Bun + Node |

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
| `actions/setup-python` | `a309ff8b426b58ec0e2a45f0f869d46889d02405` | v6.2.0 |
| `actions/cache` | `55cc8345863c7cc4c66a329aec7e433d2d1c52a9` | v6.1.0 |
| `actions/upload-artifact` | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` | v7.0.1 |
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

Work flows **feature → `staging` → `main`**. `main` is the (eventual) production branch — for now a placeholder. `staging` is the active integration branch and is where all work actually lands.

> **`staging` is currently protected MORE heavily than `main`, not less.** Only the `staging` rulesets require a passing `CI` check; no `main` ruleset does. See [Branch rulesets](#branch-rulesets) for the measured state — do not assume `main` is the hardened branch.

Practical consequences:

- **CI + CodeQL trigger on BOTH** branches: `on: push` / `pull_request` with `branches: [main, staging]`.
- **Dependabot targets `staging`** (`target-branch: staging`) so dependency updates land on staging, pass CI there, and promote to main — never landing on main unvalidated. For monorepo/placeholder repos where the real manifests live on `staging` (e.g. `omni-fi-link`), Dependabot also *reads* manifests from staging.

**Exception — public SDK repos use `main` + PR branches only (no `staging`).** This is the industry standard for published SDKs (e.g. `omni-fi-react-link`): contributions land via PR straight to `main`. There, CI triggers on `main` only and Dependabot targets `main`.

## Branch rulesets

This section describes what is **actually configured**, verified against the API on 2026-08-21. It previously described a target state, which is why it is now explicit about the gap.

**Every** branch ruleset carries: required PR review (+ code-owner), **Copilot code review**, no-force-push (`non_fast_forward`), and no-deletion.

**Only the three `Protect Staging` rulesets additionally require a passing `CI` check and `required_linear_history`** — added 2026-08-18.

| Repo | Ruleset | `CI` required | `required_linear_history` |
| --- | --- | --- | --- |
| `omni-fi-core` | Protect Staging | ✅ | ✅ |
| `omni-fi-web` | Protect Staging | ✅ | ✅ |
| `omni-fi-link` | Protect Staging | ✅ | ✅ |
| `omni-fi-core` | Protect Main | ❌ | ❌ |
| `omni-fi-web` | Protect Main | ❌ | ❌ |
| `omni-fi-link` | Protect Main | ❌ | ❌ |
| `omni-fi-react-link` | Protect Main | ❌ | ❌ |
| `omni-fi-internal` | Protect Main | ❌ | ❌ |
| `omni-fi-internal` | Protect Staging | ❌ | ❌ |

Two cases deserve attention rather than being read as oversights to copy:

- **`omni-fi-react-link`** has no `staging`, so `main` **is** its production branch — and it is the one production branch with no required status check.
- **`omni-fi-internal`** has both rulesets and neither carries the rules.

Four repos also have a `Protect Releases` **tag** ruleset (`omni-fi-core`, `omni-fi-web`, `omni-fi-link`, `omni-fi-react-link`, `omni-fi-internal`), which this document does not otherwise cover.

### The `CI` name, and why only one check is required

`CI` is the only required context, pinned to `integration_id: 15368` so a same-named check from another app cannot satisfy it. Deliberately excluded: `Analyze` reports **skipped** on private repos, and requiring a check that never reports deadlocks every PR; `deploy` only runs on push after merge, so requiring it is circular.

`strict_required_status_checks_policy` is **false** — with many open PRs, `strict: true` invalidates every other PR on each merge and forces a re-sync cycle for no safety gain.

### Admin bypass is deliberate, and is not recorded anywhere else

Every ruleset above grants `RepositoryRole 5` (admin) `bypass_mode: pull_request`. **An admin can merge a PR with red CI.** That is the intended trade on `staging` — admins must retain an emergency override — but it means the gate protects the team's PRs from landing red without constraining an admin merge.

**Production must not inherit it.** When a production ruleset is created it should carry `required_status_checks` from day one and **no** `bypass_actors`.

(`required_linear_history` requires squash or rebase merges — confirm the repo's merge strategy before enabling. All five repos already set `allow_merge_commit: false`, so it is belt-and-braces rather than a behaviour change.)

## README badges

Every **public** repository's `README` opens with two [shields.io](https://shields.io) badges, in this order:

1. **CI status** — links to the `CI` workflow run on the default branch:

   ```
   [![CI](https://img.shields.io/github/actions/workflow/status/<org>/<repo>/ci.yml?branch=<default-branch>&label=CI)](https://github.com/<org>/<repo>/actions/workflows/ci.yml)
   ```

2. **License**:

   ```
   [![License](https://img.shields.io/github/license/<org>/<repo>.svg)](./LICENSE)
   ```

Both are required on every public repo. Private repos may omit them (CI status is
visible internally on PRs and the Actions tab); add them when a repo goes public.
