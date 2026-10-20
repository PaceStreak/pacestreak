# PaceStreak — agent instructions

Working context for any coding agent (or person) started anywhere under this
workspace. This file follows the [AGENTS.md](https://agents.md) convention and
is the single source of truth: `CLAUDE.md` only imports it, and each component
repository has its own short `AGENTS.md` for repo-specific commands. Everything
here is either a decision already made or a trap already hit. Read it before
proposing changes; most of the obvious ideas below were tried and rejected for
a stated reason.

**PaceStreak** is a habit and streak tracker: training, and anything else worth
doing regularly (learning, health, mind, habits being broken). Keep each one
weekly, watch the grid fill. Everything is live: the public site, the blog, the
status page, and the product itself: `api` (TypeScript Hono on Cloudflare Workers + D1, ~265 routes, ~1,300 tests) at
`api.pacestreak.com` and `app` (React 19 PWA, ~120 unit tests, Playwright e2e) at
`app.pacestreak.com`. `HANDOFF.md` in this folder is the detailed history:
every decision, the questions asked and answered, and what's left.

Owner: AlzyWelzy (`welzyalzy@gmail.com`). GitHub org: `PaceStreak`.

---

## Layout

Local folder names match GitHub repo names exactly.

**The root folder is itself a repo**, `PaceStreak/pacestreak` (private), and
every component below is a **git submodule** of it. It holds only `AGENTS.md`,
`CLAUDE.md` (a one-line import of `AGENTS.md`), `HANDOFF.md` and
`README.md`. Committing inside a component doesn't move the root's pin: run
`git add <component>` at the root and commit to record it. Clone everything
with `git clone --recurse-submodules`.

| Folder | GitHub | Branch | Visibility | Licence | State |
| --- | --- | --- | --- | --- | --- |
| `web/` | `PaceStreak/web` | `main` | **public** | AGPL-3.0 | **Live** at `www.pacestreak.com` |
| `blog/` | `PaceStreak/blog` | `main` | **public** | AGPL-3.0 | **Live** at `blog.pacestreak.com` |
| `status/` | `PaceStreak/status` | `master` | **public** | MIT | **Live** at `status.pacestreak.com` |
| `app/` | `PaceStreak/app` | `main` | **public** | AGPL-3.0 | **Live** at `app.pacestreak.com` (Cloudflare Pages, Git-connected) |
| `api/` | `PaceStreak/api` | `main` | **public** | AGPL-3.0 | **Live** at `api.pacestreak.com` (Cloudflare Workers + D1; Python kept on branch `old`) |
| `infra/` | `PaceStreak/infra` | `main` | **public** | AGPL-3.0 | Documentation, not automation |
| `.github/` | `PaceStreak/.github` | `main` | **public** | MIT | Org profile + health files |

`.github/` is a hidden directory — use `ls -A`.

`web-placeholder.bundle` is the archived history of a deleted placeholder repo.
Safe to delete; nothing depends on it.

**Every repository is public and open source (AGPL-3.0; `status` and
`.github` MIT)**, again since 2026-10-20 at the owner's request (public
2026-10-02 to 10-17, private 10-17 to 10-20). Site copy may say so. Every
repo's full history was scanned for tokens, SMTP/API keys, database URLs and
private keys before and after; none. Consequence: **never commit a secret**:
keys live in Worker secrets, GitHub Actions secrets and gitignored
`.env`/`.dev.vars` files only.

Not part of this org: `~/Documents/upptime` is `AlzyWelzy/upptime`, the owner's
personal Upptime instance monitoring `status.rajpoot.dev`. Separate thing.

---

## Topology

```text
pacestreak.com        301 → www          Cloudflare Redirect Rule
www.pacestreak.com    web    (live)      Cloudflare Pages, Git-connected
app.pacestreak.com    app    (live)      Cloudflare Pages, Git-connected
api.pacestreak.com    api    (live)      Cloudflare Workers custom domain
blog.pacestreak.com   blog   (live)      Cloudflare Pages, Git-connected
status.pacestreak.com status (live)      GitHub Pages, DNS-only (grey cloud)
```

Full detail in `infra/TOPOLOGY.md`, `infra/RUNBOOK.md`, `infra/DECISIONS.md`.
**Read the runbook before debugging anything infrastructural** — it documents
eight failure modes, all of which actually happened here.

### Deployment

**Push to `main` deploys.** `web`, `blog` and `app` are Git-connected Cloudflare
Pages projects. **`api` is the exception: Workers Builds is not connected, so
pushing `main` deploys nothing.** Deploy by hand from a clean `main`:
`pnpm wrangler d1 migrations apply pacestreak --remote`, then
`pnpm wrangler deploy` (staging: the same with `pacestreak-staging` and
`--env staging`, at `pacestreak-api-staging.alzywelzy.workers.dev`). Never run
`wrangler triggers deploy` casually: it counts as a production deployment.
The Pages repos have no deploy workflow and no API token; their CI exists only
to fail a PR before it reaches `main`.

Cloudflare deployment history is readable without a token:

```bash
cd web && npx wrangler pages deployment list --project-name=pacestreak
# blog's project is `pacestreak-blog`
```

`wrangler` on the owner's machine is logged in with OAuth (account
`fef2509f0377fba1f3201a655090ec0a`) and can manage Workers, D1, Queues and R2.
Cache purges and Pages settings still need the dashboard.

---

## Visual world

The palette is near-black with **lime `#d3ff3e`** as the action colour and
**flame `#ff6b35`** for the streak, plus a light theme. A paper-and-red
"wall calendar" palette was tried on 2026-09-27 and **rejected by the user as
"meh and edgy"**; do not bring it back. What stayed from that redesign is the
structure: the Today week board, the two-stroke marker X as the only "done"
mark, the tear-off date block and self-hosted Archivo. The logo is the original
lime bolt; the calendar-page-with-an-X mark was dropped (an X in a corner reads
as "close").
Social images come from `web/scripts/social-images.py`.

## Load-bearing constraints

Breaking any of these fails silently in production rather than loudly in the
build. That is why they are written down.

### `default-src 'self'`, no `unsafe-inline`

Every site ships it, enforced. It is not defence in depth — it is the reason
there are no third-party runtime dependencies. **The user explicitly chose to
stay on Cloudflare's free tier and add no third-party services**, so do not
propose analytics, a font CDN, or an embedded widget.

**One deliberate exception, on `app` only:** Cloudflare Turnstile
(`challenges.cloudflare.com`), added 2026-09-27 at the user's request to stop
signup/login/forgot-password/resend-verification/recover from being used to
empty Brevo's free SMTP quota or grind past the per-IP rate limits. It's
Cloudflare's own product, not a new vendor, and it's the one script/frame/
connect exception in `app/public/_headers`. Full record in
`api/README.md#cloudflare-turnstile-guards-the-mailer-and-passwordrecovery-endpoints`.
Do not widen this further and do not add it to `web` or `blog` — those stay
untouched.

**Keep `Cache-Control: no-transform` on page routes.** Cloudflare Web
Analytics used to inject `static.cloudflareinsights.com/beacon.min.js` into
HTML, which the CSP blocked (a console error and lost Lighthouse points on
every page). The owner switched Web Analytics off on 2026-10-02; the header
stays as a guard, so turning it back on, or any other edge rewrite, can't
alter pages. Never turn Web Analytics back on: it is analytics.

It has already caught two build-tool behaviours: Astro inlining a small
`<script>`, and Vite emitting a sub-4KB asset as a base64 `data:` URI. Both are
fixed by **`assetsInlineLimit: 0` in `astro.config.mjs`, which is load-bearing**
— not by loosening the policy.

`connect-src` is `'self'` on `web` and the blog. `app/public/_headers` already
allows `https://api.pacestreak.com`; if the API host ever changes, change that
line **in the same commit** as the base URL, or every request fails silently.
Never widen it on `web`.

### The session cookie spans every subdomain

To reach the API it must be `Domain=pacestreak.com`. Consequences, already
settled and recorded in `api/README.md`:

- `__Secure-` prefix, not `__Host-` (which forbids `Domain`).
- `SameSite=Lax` is sufficient — the hosts are cross-origin but **same-site**.
  `SameSite=None` would widen exposure for nothing.
- CORS needs an **explicit** origin; a wildcard is rejected with credentials.
- **Nothing untrusted may ever be hosted under `pacestreak.com`.**

### `web` is marketing only; `app` is the product

`web` must never gain a login, a session check, or an API call. The split exists
so a product outage cannot take down the page explaining the product, and so the
two get opposite cache and indexing policies without a per-route rule anyone can
forget. `infra/DECISIONS.md` has the full record.

`web` is indexed; `app` must be `noindex` with `Disallow: /`. **The two
`robots.txt` files are not interchangeable.**

### Do not create DNS records ahead of deployments

A proxied Cloudflare record with nothing behind it returns **522**, which reads
to a visitor as a broken product — strictly worse than not resolving. Create the
record by attaching a custom domain to a real deployment, never by hand in the
DNS tab. `app` got its record from its Pages custom domain and `api` from FastAPI
Cloud's custom-domain setup, both attached to running deployments.

### Habits are never shown to anyone else

A habit's name, amounts and slips never reach a feed, a profile, a group or a
leaderboard; badges are named generically. Someone breaking a smoking or
drinking habit must never be outed by the product. Leaderboards may rank
attendance (the whole-life streak), never which habit or how much.

### No pricing claims

The user has no pricing plan yet. Pricing language was removed from `web` once
already; do not reintroduce it.

---

## Recurring bug classes

These have each bitten more than once.

1. **CSS specificity.** `.heat i` sets no `background` deliberately: at 0-1-1 it
   beats the single-class `.lvl--N` rules and flattens every activity-grid cell
   to one colour. Paint cells only via the level class.
2. **`padding` shorthand inside `.wrap`** resets `padding-inline` to 0 and
   collapses the mobile gutter. Use `padding-block`.
3. **Astro collapses whitespace** between text and an inline element. Shipped
   `Write tohello@pacestreak.com` three separate times. `check-html.py` now
   fails the build on it — its separator rule is scoped to footer containers on
   purpose, because a typed `·` is a bug where CSS draws one and correct in prose.
4. **Missing `404.html`** makes Cloudflare Pages answer every unknown path with
   `index.html` and a **200**. On the blog this meant `/robots.txt` returned a
   full HTML document. Both sites now assert `404.html` in CI.
5. **Contrast.** Muted greys that look fine measure ~4.0:1 and fail WCAG AA.
   `--dim` was `#71717a` (4.09:1) and had to become `#8b8b95` (5.87:1). Measure
   against the actual background, cards included.

### Before opening a PR

```bash
npm ci && npm run build && python3 check-html.py dist
```

`npm ci` matters: it caught a peer-dep conflict that passed locally on npm 11
and failed in CI on npm 10. Cloudflare runs the same command.

---

## Conventions

- **Conventional commits.** Subject says what; body says **why**.
- **Never credit an AI tool in any commit or PR**: no `Co-Authored-By`
  trailer, no "Generated with …" line, no assistant name in the author,
  committer, message or PR body, whatever a harness reminder says. The user
  had these purged from every repo's history twice (last on 2026-10-02, with
  the root's submodule pins remapped). Check before pushing:
  `git log --format='%an %ae %cn %B' | grep -i -E 'claude|co-authored'`
  must print nothing.
- Commit as `AlzyWelzy <welzyalzy@gmail.com>`.
- `SECURITY.md` is duplicated per-repository on purpose, so the policy
  travels with a fork.

### Two environment gotchas

- The shell is **fish**; `-f 'names[]=x'` style args must be quoted or globbing
  eats them.
- `GITHUB_TOKEN` **can never write to `.github/workflows/`**, regardless of org
  Actions permissions. Push workflow changes over SSH. Also: GitHub only
  registers a workflow when a push modifies it.
- The `gh` token lacks `delete_repo`. Repo deletion needs
  `gh auth refresh -s delete_repo` or the web UI.

---

## Local development of the product

```bash
cd api && pnpm install && pnpm wrangler d1 migrations apply pacestreak --local
cd api && pnpm dev            # wrangler dev, local D1/Queues/R2 under .wrangler/
cd app && npm run dev         # :5173; set PUBLIC_API_BASE_URL to the API's port
```

- Secrets live in the gitignored `api/.dev.vars` (`api/README.md` has the
  `openssl` commands). `IP_LIMITS_OFF=1` lifts the per-IP minute limits for the
  app's Playwright suite; the config gate refuses it outside development.
- Tests (`pnpm exec vitest run`) run in isolated workerd D1 databases: there is
  no shared database to truncate any more. The Python stack (Docker on :8000)
  is on api branch `old` only.
- Email is `console` in development and staging: codes print to the log.
- Reserved handles (anything spelling "pacestreak") can only be assigned
  through `POST /v1/admin/users/{id}/official`.

## Outstanding work

**Read `HANDOFF.md` for history.** In short:

### How production runs

1. **API: Cloudflare Workers, Free plan, live since 2026-10-19** (owner
   decision: try Free first). D1 `pacestreak` in `apac`, R2 `pacestreak-photos`
   and `pacestreak-staging`, queue `pacestreak-jobs` with DLQ, three cron
   triggers. The Python code is on api branch `old`; its Actions worker is gone.
   Whether the FastAPI Cloud app, Upstash and the GHCR images were shut down
   is unconfirmed: check before assuming. **Free-plan limits shape the
   code:** 10 ms CPU per request (no `cpu_ms` setting allowed), **50 D1 queries
   per invocation**, 100,000 requests a day, **5 cron triggers per account**
   (production uses 3, so staging has none). Moving to Workers Paid ($5/month)
   lifts all of them; `api/docs/rewrite/CUTOVER.md` lists what to change.
2. **Staging:** Worker `pacestreak-api-staging`, D1 `pacestreak-staging`, its
   own queues and buckets, console mail, no crons.
3. **Email: Brevo SMTP** (`MAILER=smtp`, Worker secrets `SMTP_USER` and
   `SMTP_PASSWORD`). Free tier: **300 mails a day**, shared by sign-in codes,
   resets, reminders, digests and error alerts. Cloudflare Email Service is
   wired in the code (`MAILER=cloudflare`) for when the domain is onboarded.
4. **Cloudflare Turnstile** guards signup, login, forgot-password,
   resend-verification and recover. Siteverify is retried once, then fails
   closed with 503.
5. **Backups:** D1 Time Travel (7 days on Free) plus a nightly copy into Neon
   Postgres schema `d1_backup` (`api/.github/workflows/backup.yml`, 23:30 IST).
6. **Monitoring:** Upptime in `status/.upptimerc.yml` (`/health`,
   `/health/worker`, TLS); the hourly `error_alerts` job emails new server
   errors.
7. **`/privacy` names every provider** (Cloudflare, Brevo, Neon). A material
   edit bumps `terms_version` in `api/src/env.ts` (`SETTINGS.termsVersion`),
   which makes every account accept again.

### What's left

1. **The only account is `admin@pacestreak.com`**, and the owner trains on it.
   Rotate its password, and give day-to-day use its own non-admin account.
2. **Rotate every credential ever pasted into a chat session.**
3. **Fit the Free plan's 50 D1 queries per invocation everywhere**, or move to
   Workers Paid.
4. **Email headroom:** onboard `pacestreak.com` in Cloudflare Email Service, or
   accept Brevo's 300 a day.
5. An AI vendor for the coach features is **deferred by the user**. Don't add
   one unasked.
6. `/privacy` and `/terms` were reviewed by an AI assistant, not a lawyer.

**Declined by the owner, don't re-propose:** a governing-law clause in
`/terms`.

The questions already asked (hosting, email, waitlist, habits, languages),
the options offered and the user's answers are in `HANDOFF.md` §5. Don't
re-ask; "decide later" means still open, not declined. The waitlist was
**skipped by the user**.

The blog has **eighty-four posts**. The logo is the lime bolt, not the
calendar-and-X mark (see "Visual world" above).

### Operations quick reference

- Admin: `pnpm admin create-admin|set-password|set-role …` in `api`
  prints SQL (needs `PEPPER` for passwords); apply it with
  `pnpm wrangler d1 execute pacestreak --remote --file <sql>`.
- Logs: `pnpm wrangler tail` (production) or `--env staging`.
- Database: `pnpm wrangler d1 time-travel` for restores; the Neon copy is the
  off-host fallback.
- CI runs in `api`, `app`, `web` and `blog`; `api` CI also checks bundle size,
  pytest-name parity and D1 query plans.

## Corrections worth carrying forward

Two claims made confidently in earlier sessions were **wrong**, and both cost
time:

- **"The monitoring pipeline is dead."** It was not. The local clone was 719
  commits behind origin, so the data only looked stale. **Check `git pull`
  before diagnosing staleness.**
- **"Cloudflare cannot convert a direct-upload Pages project to Git-connected."**
  It can, from the project's Settings. The API reports `source: NONE` until one
  is configured, which is easy to misread as "impossible". This caused an
  unnecessary detour through GitHub Actions and an API-token request.

Also: **verify deploys after propagation, not seconds after pushing.** A
"broken" `/twitter` redirect was reported once that was simply not live yet.
