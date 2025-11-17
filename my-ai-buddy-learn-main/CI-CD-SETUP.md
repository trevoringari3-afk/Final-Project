# CI/CD & Code Quality Setup Guide

This document describes the CI/CD pipelines, code quality gates, and real-time flagging setup for the Happy Learn project.

## Overview

Your project now has a comprehensive CI/CD orchestration with GitHub Actions, pre-commit hooks, and TypeScript/ESLint enforcement.

### Three-Layer Quality Assurance

1. **Local Development** — Pre-commit hooks prevent bad code from being committed
2. **Pull Request Checks** — GitHub Actions run ESLint, TypeScript, and build checks with PR annotations
3. **Deployment** — Separate workflows for frontend (Netlify) and Supabase Edge Functions (via CLI)

---

## Local Development Setup

### Install Dependencies

After cloning or pulling the repo, install the dependencies including new dev tools:

```bash
npm ci
```

This installs:
- `husky` — Git hook manager
- `lint-staged` — Run linters on staged files only (faster)
- Latest ESLint and TypeScript configs

### Initialize Husky

After first install, initialize Husky hooks:

```bash
npm run prepare
```

This sets up the `.husky/` directory and installs Git hooks.

### Pre-Commit Hooks

When you attempt to commit, Husky runs `lint-staged` automatically, which:

1. Runs ESLint with auto-fix on staged TypeScript/TSX files
2. Stages fixed files automatically
3. Blocks commit if there are still errors

**Example:**

```bash
git add src/some-file.tsx
git commit -m "fix: improve component"
# → Husky runs eslint --fix, then git add, then allows commit
```

To skip hooks (not recommended):

```bash
git commit --no-verify
```

---

## Pull Request & CI Checks

### `.github/workflows/ci.yml`

Runs on **every pull request and push to `main`/`develop`** branches.

**Jobs:**

1. **lint-and-typecheck** (parallel ESLint + tsc checks)
   - Installs dependencies
   - Runs `eslint "src/**/*.{ts,tsx}"` → generates report
   - Runs `tsc --noEmit` → typecheck report
   - Uses `reviewdog` to post **inline annotations** to PR comments
   - Shows each error/warning directly on the offending line in GitHub

2. **build** (depends on lint-and-typecheck)
   - Verifies the project builds successfully
   - Uploads `dist/` artifact for 5 days

**Real-Time Flagging in PRs:**

When you open a PR, GitHub Actions automatically posts comments with:
- **ESLint Issues** — code style, logic errors, React best practices
- **TypeScript Issues** — type mismatches, unused variables, implicit `any`

Example PR comment:
```
src/components/MyComponent.tsx:15:5
⚠️ ESLint: '@typescript-eslint/no-explicit-any' - Unexpected any. Specify a different type.
```

You can see these inline in the "Files changed" tab or in the checks summary.

---

## Deployment Workflows

### Frontend Deployment (`.github/workflows/deploy-frontend.yml`)

**Triggers:** `push` to `main` branch only

**Current Setup:** Deploys to **Netlify**

```yaml
jobs:
  deploy-to-netlify:
    - Builds the Vite project
    - Deploys using netlify/actions/cli@master
    - Requires secrets: NETLIFY_AUTH_TOKEN, NETLIFY_SITE_ID
```

**To Enable Vercel Instead (or alongside):**

Uncomment the `deploy-to-vercel` job in `.github/workflows/deploy-frontend.yml`:

```yaml
# Uncomment and configure when ready to deploy to Vercel
steps:
  - uses: actions/checkout@v4
  - name: Deploy to Vercel
    uses: amondnet/vercel-action@v26
    with:
      vercel-token: ${{ secrets.VERCEL_TOKEN }}
      vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
      vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
      vercel-args: '--prod'
```

Then add secrets to GitHub (see below).

### Supabase Functions Deployment (`.github/workflows/deploy-supabase.yml`)

**Triggers:** `push` to `main` branch, **only when** `supabase/functions/**` or `supabase/config.toml` changes

**Deploys 7 Edge Functions:**
- `chat`
- `gap-detector`
- `generate-demo-data`
- `studybuddy-hydrate`
- `studybuddy-next`
- `studybuddy-report`
- `teacher-insights`

**Requires Secrets:**
- `SUPABASE_ACCESS_TOKEN` — your Supabase management API token
- `SUPABASE_PROJECT_REF` — your project ID (e.g., `mzqszgunhmkphfclkyip`)

---

## GitHub Secrets Configuration

Add the following secrets to your GitHub repository:

**Settings → Secrets and variables → Actions → New repository secret**

### For Netlify Deployment

| Secret Name | Value | Where to Find |
|-------------|-------|---|
| `NETLIFY_AUTH_TOKEN` | Personal access token | Netlify → User settings → Applications → Personal access tokens |
| `NETLIFY_SITE_ID` | Site ID | Netlify → Site settings → General → Site ID |

### For Vercel Deployment (Optional)

| Secret Name | Value | Where to Find |
|-------------|-------|---|
| `VERCEL_TOKEN` | Personal access token | Vercel → Settings → Tokens |
| `VERCEL_ORG_ID` | Organization ID | Vercel → Settings → General |
| `VERCEL_PROJECT_ID` | Project ID | Vercel → Project settings → Project ID |

### For Supabase Functions Deployment

| Secret Name | Value | Where to Find |
|-------------|-------|---|
| `SUPABASE_ACCESS_TOKEN` | Service role key or PAT | Supabase → Settings → API → Service role key (or create a PAT in Account settings) |
| `SUPABASE_PROJECT_REF` | Project reference ID | Supabase → Settings → General → Project reference (e.g., `mzqszgunhmkphfclkyip`) |

---

## Disabling Platform Auto-Deploy (Recommended)

To **avoid duplicate deployments**, disable auto-deploy on Netlify/Vercel platforms:

### Netlify

1. Go to **Site settings → Build & deploy → Deploy contexts**
2. Under "Deploy on push", choose:
   - **Branch deploys**: Disable
   - **Deploy previews**: Leave enabled (for PRs) or disable

### Vercel

1. Go to **Settings → Git**
2. Under "Deploy on Push", toggle **OFF**

This ensures **only GitHub Actions** trigger deployments.

---

## Code Quality Improvements Made

### TypeScript Strictness (tsconfig.json)

**Before:**
```json
{
  "noImplicitAny": false,
  "strictNullChecks": false,
  "noUnusedLocals": false,
  "noUnusedParameters": false
}
```

**After:**
```json
{
  "strict": true,
  "noImplicitAny": true,
  "strictNullChecks": true,
  "noUnusedLocals": true,
  "noUnusedParameters": true,
  "esModuleInterop": true,
  "forceConsistentCasingInFileNames": true
}
```

### ESLint Rules (eslint.config.js)

**Added:**
- `@typescript-eslint/no-explicit-any: warn` — warns on `any` types
- `@typescript-eslint/no-unused-vars: warn` — warns on unused vars/params (with `_` prefix exception)

### Code Fixes Applied

#### Console.log Guards

All `console.log` calls now wrapped in development checks:

```typescript
if (import.meta.env.DEV) {
  console.log('Debug info:', data);
}
```

This prevents production logging and keeps console clean.

#### Type Fixes

Replaced all `: any` with proper types:

- `metadata: any` → `metadata: Record<string, unknown>`
- `payload: any` → `payload: Record<string, unknown>`
- `error: any` → `error: AuthError | null` (with custom `AuthError` interface)

Files fixed:
- `src/main.tsx` — console.log
- `src/contexts/AuthContext.tsx` — console.log, `any` types
- `src/pages/DemoDataGenerator.tsx` — error handling
- `src/hooks/useOfflineSync.tsx` — `any` type
- `src/hooks/useActivityCache.tsx` — `any` type
- `src/components/StudyBuddy.tsx` — `any` type

---

## Quick Command Reference

### Development

```bash
# Start dev server
npm run dev

# Lint code
npm run lint

# Auto-fix ESLint issues
npm run lint:fix

# Type-check without emitting
npm run typecheck

# Build for production
npm run build
```

### Pre-Commit / Git

```bash
# Commit (triggers husky pre-commit hook)
git add .
git commit -m "feat: add feature"

# Skip hooks (not recommended)
git commit --no-verify
```

### Debugging GitHub Actions

If a workflow fails:

1. Go to your GitHub repo → **Actions** tab
2. Click the failed workflow run
3. Click the failed job
4. Expand step logs to see errors
5. Fix locally and push

---

## Troubleshooting

### Husky hooks not running after clone

**Solution:**

```bash
npm run prepare
```

### ESLint errors blocking commit

**Solution:**

Run locally to see the issue:

```bash
npm run lint
npm run lint:fix  # Auto-fix where possible
git add .
git commit -m "..."
```

### PR checks failing but local lint passes

**Solution:**

Ensure Node.js version matches (18):

```bash
node --version  # Should be v18.x.x
```

If different, update locally or use nvm:

```bash
nvm use 18
npm ci
npm run lint
```

### Netlify/Vercel deployment conflicts

**Problem:** Both platform auto-deploy AND GitHub Actions deploy.

**Solution:** Disable platform auto-deploy (see "Disabling Platform Auto-Deploy" above).

---

## Next Steps

1. **Install locally:**
   ```bash
   npm ci
   npm run prepare
   ```

2. **Test pre-commit hooks:**
   ```bash
   git add .
   git commit -m "test: verify husky hooks work"
   ```

3. **Create a test PR** to see GitHub Actions annotations in action.

4. **Add secrets to GitHub** (see configuration section).

5. **Optionally**: Enable Vercel deployment by uncommenting the job in `.github/workflows/deploy-frontend.yml` and adding secrets.

---

## References

- [ESLint Configuration](./eslint.config.js)
- [TypeScript Configuration](./tsconfig.json)
- [GitHub Actions Workflows](./.github/workflows/)
- [Husky Documentation](https://typicode.github.io/husky/)
- [lint-staged Documentation](https://github.com/okonet/lint-staged)
- [reviewdog Documentation](https://github.com/reviewdog/reviewdog)
