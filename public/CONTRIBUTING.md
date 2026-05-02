# Contributing to RAG Studio

Thank you for contributing to RAG Studio!

---

## Table of Contents

1. [Branch Strategy](#branch-strategy)
2. [Branch Protection Rules](#branch-protection-rules)
3. [Development Workflow](#development-workflow)
4. [Commit Convention](#commit-convention)
5. [PR Checklist](#pr-checklist)
6. [Code Style](#code-style)
7. [Testing Requirements](#testing-requirements)
8. [Local Setup](#local-setup)

---

## Branch Strategy

We follow a simplified **Git Flow**:

```
main          ← production-ready releases only (protected)
  └── develop ← integration branch; all features merge here first (protected)
        └── feature/<phase-id>-<slug>   ← one branch per task
        └── fix/<short-description>     ← bug fixes
        └── chore/<short-description>   ← tooling/config changes
```

| Branch | Purpose | Who can push |
|--------|---------|-------------|
| `main` | Production releases. Tagged with `vX.Y.Z`. | Merge from `develop` via PR only |
| `develop` | Integration. All CI checks must pass before merge. | Merge from feature branches via PR only |
| `feature/*` | One task per branch (see `TASKS.md`). | Author + reviewers |

**Dependency rule:** Complete tasks in the order listed in `TASKS.md`. Later phases depend on earlier ones being merged to `develop`.

---

## Branch Protection Rules

Configure these rules in **GitHub → Settings → Branches** for both `main` and `develop`:

### `main`

| Rule | Setting |
|------|---------|
| Require pull request before merging | ✅ Enabled |
| Required approvals | **2** |
| Dismiss stale pull request approvals | ✅ Enabled |
| Require review from code owners | ✅ Enabled (if `CODEOWNERS` exists) |
| Require status checks to pass | ✅ Enabled |
| Required status checks | `CI — All checks passed`, `Tests — All passed` |
| Require branches to be up to date | ✅ Enabled |
| Restrict who can push | Admins only |
| Allow force pushes | ❌ Disabled |
| Allow deletions | ❌ Disabled |

### `develop`

| Rule | Setting |
|------|---------|
| Require pull request before merging | ✅ Enabled |
| Required approvals | **1** |
| Dismiss stale pull request approvals | ✅ Enabled |
| Require status checks to pass | ✅ Enabled |
| Required status checks | `CI — All checks passed` |
| Require branches to be up to date | ✅ Enabled |
| Allow force pushes | ❌ Disabled |
| Allow deletions | ❌ Disabled |

---

## Development Workflow

```bash
# 1. Fork and clone
git clone https://github.com/YOUR_USERNAME/rag-studio.git
cd rag-studio

# 2. Fetch latest develop
git fetch origin
git checkout develop
git pull origin develop

# 3. Create your feature branch (match the TASKS.md branch name)
git checkout -b feature/p1-json-model-catalogs

# 4. Start the dev environment
cp .env.example .env   # fill in API keys
docker compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up -d

# 5. Make changes, commit often
git add <files>
git commit -m "feat(p1-1): add embeddings.json catalog"

# 6. Push and open PR against develop
git push origin feature/p1-json-model-catalogs
# → Open PR on GitHub: base = develop, compare = feature/p1-json-model-catalogs
```

---

## Commit Convention

We use **Conventional Commits**:

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

| Type | When to use |
|------|-------------|
| `feat` | New feature or behaviour |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `chore` | Tooling, config, CI changes |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

**Scope** should be the task ID or module name: `p0-3`, `chunking`, `evaluation`, `docker`, etc.

**Examples:**
```
feat(p1-1): add embeddings.json with 10 model entries
fix(chunking): handle empty document input in SemanticChunker
chore(ci): add mypy to CI lint job
test(p2-2): add unit tests for MarkdownHeaderChunker
```

---

## PR Checklist

Before opening a PR, verify:

- [ ] Branch name matches `TASKS.md` exactly (e.g. `feature/p1-json-model-catalogs`)
- [ ] PR targets `develop` (not `main`)
- [ ] All CI checks pass locally before pushing:
  ```bash
  # Frontend
  cd apps/web && npm run lint && npm run type-check && npm test

  # Backend
  cd apps/api && ruff check . && ruff format --check . && pytest tests/ --ignore=tests/test_integration
  ```
- [ ] No secrets or API keys committed
- [ ] No unrelated files modified
- [ ] PR description includes: what was done, why, and how to test

---

## Code Style

### TypeScript / React

- Functional components with explicit type annotations
- No `any` — use `unknown` or proper types
- ESLint rules enforced (`npm run lint`)
- TypeScript strict mode (`npm run type-check`)

### Python

- Type hints on all function signatures
- Ruff for linting and formatting (`ruff check .` + `ruff format .`)
- Black-compatible formatting (Ruff handles this)
- Docstrings on public methods: one-line summary only — no multi-paragraph blocks

---

## Testing Requirements

| Layer | Tool | Minimum coverage |
|-------|------|-----------------|
| Backend unit | pytest | 70% |
| Backend integration | pytest + real Postgres/Redis/Qdrant | Key flows covered |
| Frontend unit | Vitest / Jest | 75% |
| E2E | Playwright | Designer flow + Autopilot flow |

Run integration tests locally:
```bash
# Requires db, redis, vector-db running
docker compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up -d db redis vector-db

cd apps/api
pytest tests/test_integration/ -v
```

---

## Local Setup

### Prerequisites

- Docker ≥ 24 + Docker Compose ≥ 2.20
- Node.js ≥ 18 + npm ≥ 9
- Python ≥ 3.11
- [uv](https://docs.astral.sh/uv/) (Python package manager)

### Frontend

```bash
cd apps/web
npm install
npm run dev   # → http://localhost:3000
```

### Backend

```bash
cd apps/api
uv venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
uv pip install -r requirements.txt -r requirements-dev.txt
uvicorn app.main:app --reload   # → http://localhost:8000
```

### Questions?

- Open an [Issue](https://github.com/yourusername/rag-studio/issues)
- Discord: https://discord.gg/ragstudio
