# Skeleton

Starter template for Leverage Workshop web apps, copied from
[alpha-scout](https://github.com/Leverage-Workshop/alpha-scout) at commit `962b09e`. It's a working
Next.js app with a database, AI chat, tests and an agent harness already set up.

Everything still uses the **alpha-scout** name and holds alpha-scout's docs and demo features.
Follow [After you clone](#after-you-clone) before you build anything.

## What's included

| Area | What you get |
|---|---|
| App | Next.js 16.4 (App Router), React 19.3, TypeScript 5, Tailwind CSS 4, Base UI, Zustand |
| Database | PostgreSQL 18 + TimescaleDB 2.30 (local Docker, Aiven in prod), plain `pg`, SQL migration runner (`db/migrations`, `scripts/migrate.ts`) |
| AI | Vercel AI SDK + AI Gateway, `useChat` streaming chat rendered with Streamdown, an example tool |
| Health checks | `GET /api/health/db` and `GET /api/health/ai` |
| Quality gate | ESLint, `tsc`, Vitest (unit + DB integration), all run by `./init.sh` |
| Agent harness | `AGENTS.md` (+ `CLAUDE.md` → `@AGENTS.md`), `feature_list.json`, `progress.md`, `session-handoff.md`, PR template, a "ship every request as a merged PR" workflow |
| Agent skills | Vendored AI SDK / AI Elements / Streamdown / React skills in `.claude/skills` and `.agents/skills` (pinned in `skills-lock.json`) |
| Hosting | Built for Vercel, with Cloudflare DNS (see `docs/INFRASTRUCTURE.md`) |

## After you clone

Below, `my-app` stands for your project's name. Use kebab-case (`my-app`), snake_case (`my_app`) for
the DB, and camelCase (`myApp`) in code.

### 1. Create the repo

Either use GitHub's template feature (if this repo is marked as a template):

```bash
gh repo create Leverage-Workshop/my-app --private --template Leverage-Workshop/skeleton --clone
```

or copy it with fresh history:

```bash
git clone https://github.com/Leverage-Workshop/skeleton.git my-app
cd my-app
rm -rf .git && git init -b main
gh repo create Leverage-Workshop/my-app --private --source . --remote origin
```

### 2. Rename the project

The old name appears as `alpha-scout`, `alpha_scout`, `alphaScout`, `Alpha Scout` and `alphascout`.
Run this from the repo root to replace every form (vendored skills excluded):

```bash
grep -rIl --exclude-dir={node_modules,.next,.git,.claude,.agents} \
  -e alpha-scout -e alpha_scout -e alphaScout -e 'Alpha Scout' -e alphascout . \
  | xargs sed -i \
    -e 's/alpha-scout/my-app/g' -e 's/alpha_scout/my_app/g' -e 's/alphaScout/myApp/g' \
    -e 's/Alpha Scout/My App/g' -e 's/alphascout/myapp/g'
```

(macOS: use `sed -i ''`.) Then confirm nothing is left:
`grep -rIil --exclude-dir={node_modules,.next,.git,.claude,.agents} 'alpha.\?scout' .`

The command touches these files. Check the ones marked * by hand afterwards:

| File | What changes |
|---|---|
| `package.json`, `package-lock.json` | package `name` |
| `docker-compose.yml` | local DB user, password, database, healthcheck |
| `.env.example`, `init.sh`, `AGENTS.md` | local `DATABASE_URL` |
| `src/app/layout.tsx` | page `<title>` |
| `src/app/page.tsx`, `src/app/page.test.tsx` | heading and its test |
| `src/lib/db/pool.ts` | global pool key (`alphaScoutPool`) |
| `docs/INFRASTRUCTURE.md` * | Aiven project, Vercel project, domain: rewritten in step 5 anyway |
| `docs/PRODUCT.md` * | replaced in step 3 anyway |
| `feature_list.json` * | reset in step 4 anyway |

Also check:

- **Port**: `package.json` runs the app on **3100** (`dev` and `start`). Change it if another project
  already uses 3100 locally.
- **Local DB port**: `docker-compose.yml` maps Postgres to **5432**. If you run two skeleton-based apps
  at once, give one a different host port and update `DATABASE_URL` to match.
- **`LICENSE`** is Apache 2.0 with no name in it. Swap it if the new project needs another license.

### 3. Replace the product doc

`docs/PRODUCT.md` is alpha-scout's PRD. Replace it with the new project's PRD, keeping the idea of
a section for functional requirements. `AGENTS.md` tells agents to read it first and not to build
speculative features until it defines them, so update the first paragraph of `AGENTS.md` to match.

Also remove project-specific rules from `AGENTS.md` **Conventions** that no longer apply (e.g.
"Models narrate verified results...", the TimescaleDB-on-Aiven note if you drop TimescaleDB).

### 4. Reset the harness

- **`feature_list.json`**: replace the entries with a single `feat-001` for the new project's first
  approved piece of work (or a blocked "product definition" entry). Keep the shape: `id`, `name`,
  `description`, `dependencies`, `status`, `blocked_on`, `evidence`.
- **`progress.md`**: clear **Current State**, **Blockers / Risks** and **History**. Keep the
  **Decisions Made** entries that still hold (harness layout, `init.sh` behaviour, shipping workflow,
  ESLint ignoring `.claude/**` and `.agents/**`, PR template) and drop the alpha-scout ones.
- **`session-handoff.md`**: already a blank template; leave it.
- **`.github/pull_request_template.md`**: generic, but mentions Aiven and Vercel. Edit if the stack changes.

### 5. Decide what to keep from the demo features

The app ships with two alpha-scout experiments that make useful working examples:

- **Streaming chat** (feat-004): `src/components/prompt-box.tsx` + `POST /api/chat`
  (`src/app/api/chat/route.ts`, request validation in `src/lib/ai/chat-request.ts`). It's
  unauthenticated and not rate-limited; add both before exposing it publicly.
- **NFL team report tool** (feat-005): a sample AI tool calling ESPN's public API. To remove it:
  - delete `src/lib/sports/` and `src/components/team-report-call.tsx`
  - remove `getTeamReport` from `src/lib/ai/tools.ts`
  - remove the `tool-getTeamReport` branch in `src/components/prompt-box.tsx`
  - remove the NFL lines from the system prompt in `src/app/api/chat/route.ts`
  - update the affected tests (`route.test.ts`, `prompt-box.test.tsx`)

If you don't need TimescaleDB, change the image in `docker-compose.yml` to plain `postgres:18`, delete
`db/migrations/0001_enable_timescaledb.sql`, and drop or adapt `scripts/check-db.ts`, `npm run db:check`,
`src/lib/db/timescale.ts`, and the `/api/health/db` TimescaleDB check.

### 6. Set up infrastructure

Rewrite `docs/INFRASTRUCTURE.md` as you go; it currently describes alpha-scout's resources.

1. **Database (Aiven)**: create a PostgreSQL service (enable TimescaleDB if kept). Note the
   service URI and download the project CA certificate.
2. **Vercel**: import the GitHub repo (Next.js preset, Node 24.x, production branch `main`). Set
   server-only env vars for Production + Preview:
   - `DATABASE_URL`: Aiven service URI
   - `DATABASE_CA_CERT`: the Aiven CA PEM. The name must be exactly this; without it, `pg` fails
     with `SELF_SIGNED_CERT_IN_CHAIN`.
   - `AI_MODEL_ID`: a Gateway model ID like `provider/model`. It must exist in the AI Gateway catalog.

   AI Gateway authenticates with Vercel OIDC on deployments, so no `AI_GATEWAY_API_KEY` is needed there.
3. **DNS (Cloudflare)**: add a CNAME for the subdomain pointing at the Vercel target, **DNS only**
   (not proxied), so Vercel can issue the TLS cert. Add the domain in Vercel.
4. **Migrations**: run `npm run db:migrate` with `DATABASE_URL` pointing at Aiven.
5. **Verify**: `/api/health/db` and `/api/health/ai` on the deployed URL both return 200.

The shipping workflow in `AGENTS.md` waits on PR checks (`gh pr checks --watch`) and those checks
come from Vercel. Connect Vercel before relying on it, or agents will have no checks to wait for.

### 7. Run it locally and commit

```bash
nvm use
npm install
cp .env.example .env.local    # set AI_GATEWAY_API_KEY and AI_MODEL_ID for the chat
npm run db:up && npm run db:migrate
DATABASE_URL=postgresql://my_app:my_app@localhost:5432/my_app ./init.sh
npm run dev                   # http://localhost:3100
```

Once `./init.sh` passes, rewrite this README for the new project (the sections below can stay) and
make the first commit.

### 8. Optional: refresh agent skills

`.claude/skills` and `.agents/skills` hold the same vendored skills (one copy per agent tool),
recorded in `skills-lock.json`. Delete any the project won't use. Keep ESLint's ignore for both
directories, because the skill example scripts don't pass lint.

## Local setup

Requires Node 24 (`nvm use`) and Docker.

```bash
npm install
cp .env.example .env.local      # defaults point at the local Docker DB
npm run db:up                   # TimescaleDB 2.30.2 on Postgres 18, port 5432
npm run db:migrate              # applies db/migrations/*.sql
npm run db:check                # confirms timescaledb is enabled
npm run dev                     # http://localhost:3100
```

## Scripts

| Script | Purpose |
|---|---|
| `dev` / `build` / `start` | Next.js (port 3100) |
| `lint` / `typecheck` / `test` | ESLint, `tsc --noEmit`, Vitest |
| `db:up` / `db:down` | Local TimescaleDB via Docker Compose |
| `db:migrate` | Apply pending SQL migrations (tracked in `schema_migrations`) |
| `db:check` | Exit non-zero unless `timescaledb` is enabled |
| `./init.sh [--quick]` | The full pre-push gate: `npm ci` if needed, lint, typecheck, test, build |

DB integration tests in `tests/` run only when `DATABASE_URL` is exported (Vitest doesn't read
`.env.local`).

## Environment variables

All server-only. Never prefix them with `NEXT_PUBLIC_`. See `.env.example`.

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `DATABASE_CA_CERT` | CA PEM for TLS to Aiven (unset locally) |
| `AI_GATEWAY_API_KEY` | Local AI Gateway auth (Vercel deployments use OIDC) |
| `AI_MODEL_ID` | Gateway model ID, e.g. `provider/model`. No default |
