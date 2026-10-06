# 25. Frontend toolkit 1: the web, Git, packages and build tools

## 25.1 How the web works

What happens when a customer types `pharmacare.co.tz`:

1. **DNS** — the browser asks the Domain Name System for the server's IP address (like a phone book). You manage this at your domain registrar or Cloudflare (A/AAAA/CNAME records).
2. **Connection** — TCP (or QUIC for HTTP/3) connection, then a **TLS** handshake for HTTPS (encryption + certificate check).
3. **HTTP request** — `GET /medicines` with headers (cookies, accepted languages).
4. **Server** — Next.js (or a CDN cache) builds or fetches the response.
5. **HTTP response** — status code + headers + HTML.
6. **Browser rendering** — parse HTML → DOM, CSS → CSSOM, run JS, calculate layout, paint pixels. Then hydrate React so the page becomes interactive.

**HTTP essentials:**

| Status | Meaning |
| --- | --- |
| 200 / 201 / 204 | OK / Created / No content |
| 301 / 302 / 304 | Moved permanently / temporarily / Not modified (cache) |
| 400 / 401 / 403 / 404 / 422 / 429 | Bad request / Not logged in / Forbidden / Not found / Validation failed / Too many requests |
| 500 / 502 / 503 | Server error / Bad gateway / Unavailable |

Other basics: **hosting** (where your server runs), **domain names**, **browsers** (Chrome, Safari, Firefox — test in each, especially Safari on iPhone), **APIs** (REST: resources + HTTP methods; GraphQL: one endpoint, client chooses fields), **WebSockets/SSE** for live updates.

## 25.2 Git and GitHub

**What it is:** Git records every change to your code, lets you go back, and lets a team work in parallel on branches. GitHub hosts repositories and adds pull requests, code review, issues and CI (GitHub Actions).

```bash
git init                                  # start a repo
git clone git@github.com:you/pharmacare-web.git

git status                                # what changed
git add .                                 # stage changes
git commit -m "feat(pos): add barcode scanning"
git push                                  # upload
git pull                                  # download others' changes

git switch -c feature/receipt-printing    # new branch
git switch main                           # change branch
git merge feature/receipt-printing
git rebase main                           # replay your branch on top of main (cleaner history)

git log --oneline --graph
git diff
git stash / git stash pop                 # park unfinished work
git restore file.tsx                      # discard changes to a file
git revert <commit>                       # undo a commit safely with a new commit
```

**Daily workflow:** branch from `main` → small commits → push → open a pull request → CI runs lint, type-check and tests → review → merge → automatic deploy. Use **Conventional Commits** (`feat:`, `fix:`, `chore:`, `refactor:`) for readable history. Never commit `.env.local` or `node_modules` (`.gitignore`).

## 25.3 Package managers

**What it is:** Tools that install libraries from the npm registry into `node_modules`, recorded in `package.json` (what you want) and a lockfile (exact versions installed).

| Command | npm | pnpm | Yarn | Bun |
| --- | --- | --- | --- | --- |
| Install all | `npm install` | `pnpm install` | `yarn` | `bun install` |
| Add | `npm i zod` | `pnpm add zod` | `yarn add zod` | `bun add zod` |
| Add dev | `npm i -D vitest` | `pnpm add -D vitest` | `yarn add -D vitest` | `bun add -d vitest` |
| Remove | `npm uninstall zod` | `pnpm remove zod` | `yarn remove zod` | `bun remove zod` |
| Run script | `npm run dev` | `pnpm dev` | `yarn dev` | `bun dev` |
| CI install (exact) | `npm ci` | `pnpm i --frozen-lockfile` | `yarn --immutable` | `bun i --frozen-lockfile` |
| Run a package once | `npx shadcn` | `pnpm dlx shadcn` | `yarn dlx shadcn` | `bunx shadcn` |

**pnpm** is fast and disk-efficient; **npm** comes with Node; pick one per project and commit its lockfile.

```json5
// package.json
{
  "name": "pharmacare-web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest",
    "test:e2e": "playwright test",
    "format": "prettier --write ."
  },
  "dependencies": { "next": "^16.0.0", "react": "^19.0.0", "react-dom": "^19.0.0", "zod": "^4.0.0" },
  "devDependencies": { "typescript": "^5.0.0", "vitest": "^3.0.0" }
}
```

**Semantic versioning:** `MAJOR.MINOR.PATCH`. `^16.1.0` allows minor and patch updates; `~16.1.0` allows patch only. Major versions may break things — read the upgrade guide (Next.js provides codemods: `npx @next/codemod@latest upgrade`). Check for risky dependencies with `npm audit` and keep them updated (Dependabot/Renovate).

## 25.4 Linters and formatters

**What it is:** A **linter** (ESLint) finds bugs and bad patterns (unused variables, missing hook dependencies, floating promises). A **formatter** (Prettier) makes code style consistent automatically. **Biome** does both in one fast tool.

```js
// eslint.config.mjs (flat config) — Next.js 16 runs ESLint directly; `next lint` was removed
import nextVitals from 'eslint-config-next/core-web-vitals';
import nextTs from 'eslint-config-next/typescript';

export default [
  ...nextVitals,
  ...nextTs,
  { ignores: ['.next/**', 'node_modules/**'] },
];
```

```json5
// .prettierrc
{ "singleQuote": true, "semi": true, "printWidth": 110, "plugins": ["prettier-plugin-tailwindcss"] }
```

Run them on save in VS Code and in CI. Optional: **Husky + lint-staged** to lint only changed files before each commit.

**Recommended VS Code extensions:** ESLint, Prettier (or Biome), Tailwind CSS IntelliSense, Pretty TypeScript Errors, Error Lens, GitLens.

## 25.5 Build tools and bundlers

**What it is:** Browsers cannot run TypeScript, JSX or thousands of npm imports efficiently. Build tools **transpile** (TS/JSX → JS), **bundle** (many files → few optimised chunks), **minify**, **tree-shake** (remove unused code) and run a dev server with **hot reload**.

| Tool | Role |
| --- | --- |
| **Turbopack** | Rust bundler built into Next.js; default for `next dev` and `next build` in Next.js 16 |
| **Vite** | Dev server + build for plain React/Vue apps; very fast |
| **webpack** | The older standard bundler; still available in Next.js (`next build --webpack`) |
| **esbuild**, **SWC** | Very fast transpilers (Go, Rust) used inside other tools |
| **Rollup / Rolldown** | Library bundlers (Vite's production build) |
| **PostCSS** | CSS transformations (Tailwind runs as a PostCSS plugin in Next.js) |
| **Turborepo / Nx** | Monorepo task runners (web app + admin app + shared UI package) |

You rarely configure these directly in Next.js; knowing what they do helps you read errors and optimise builds.
