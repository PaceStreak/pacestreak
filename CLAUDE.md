# PaceStreak — working context

Handoff notes for a Claude Code session started anywhere under
`~/Documents/pacestreak/`. Everything here is either a decision already made or
a trap already hit. Read it before proposing changes; most of the obvious ideas
below were tried and rejected for a stated reason.

**PaceStreak** is a workout streak tracker. Log the session, keep the streak,
watch the grid fill. Nothing is launched. There is a public site, a blog and a
status page (all live), plus a **built but undeployed** product: `api`
(FastAPI, ~160 routes, 134 tests) and `app` (React PWA). `HANDOFF.md` in this
folder is the latest detailed handoff: every recent decision, and what's left.

Owner: AlzyWelzy (`welzyalzy@gmail.com`). GitHub org: `PaceStreak`.

---

## Layout

Local folder names match GitHub repo names exactly.

**The root folder is itself a repo**, `PaceStreak/pacestreak` (private), and
every component below is a **git submodule** of it. It holds only `CLAUDE.md`,
`HANDOFF.md` and `README.md`. Committing inside a component doesn't move the
root's pin: run `git add <component>` at the root and commit to record it.
Clone everything with `git clone --recurse-submodules`.

| Folder | GitHub | Branch | Visibility | Licence | State |
| --- | --- | --- | --- | --- | --- |
| `web/` | `PaceStreak/web` | `main` | private | AGPL-3.0 | **Live** at `www.pacestreak.com` |
| `blog/` | `PaceStreak/blog` | `main` | private | AGPL-3.0 | **Live** at `blog.pacestreak.com` |
| `status/` | `PaceStreak/status` | `master` | **public** | MIT | **Live** at `status.pacestreak.com` |
| `app/` | `PaceStreak/app` | `main` | private | AGPL-3.0 | Built (React 19 + Vite PWA), not deployed |
| `api/` | `PaceStreak/api` | `main` | private | AGPL-3.0 | Built (FastAPI + Postgres + Redis + worker), not deployed |
| `infra/` | `PaceStreak/infra` | `main` | private | AGPL-3.0 | Documentation, not automation |
| `.github/` | `PaceStreak/.github` | `main` | **public** | MIT | Org profile + health files |

`.github/` is a hidden directory — use `ls -A`.

`web-placeholder.bundle` is the archived history of a deleted placeholder repo.
Safe to delete; nothing depends on it.

**Two repos are public on purpose.** `status` because a status page behind a
login is useless. `.github` because a **private** `.github` breaks the org
profile page and stops the shared health files applying to public repos. Do not
"fix" either by making them private.

Not part of this org: `~/Documents/upptime` is `AlzyWelzy/upptime`, the owner's
personal Upptime instance monitoring `status.rajpoot.dev`. Separate thing.

---

## Topology

```text
pacestreak.com        301 → www          Cloudflare Redirect Rule
www.pacestreak.com    web    (live)      Cloudflare Pages, Git-connected
app.pacestreak.com    app    (no DNS)    the product, built, not deployed
api.pacestreak.com    api    (no DNS)    the backend, built, hosting undecided
blog.pacestreak.com   blog   (live)      Cloudflare Pages, Git-connected
status.pacestreak.com status (live)      GitHub Pages, DNS-only (grey cloud)
```

Full detail in `infra/TOPOLOGY.md`, `infra/RUNBOOK.md`, `infra/DECISIONS.md`.
**Read the runbook before debugging anything infrastructural** — it documents
eight failure modes, all of which actually happened here.

### Deployment

**Push to `main` deploys.** `web` and `blog` are Git-connected Cloudflare Pages
projects. There is no deploy workflow and no API token in either repo. CI exists
only to fail a PR before it reaches `main`.

Cloudflare deployment history is readable without a token:

```bash
cd web && npx wrangler pages deployment list --project-name=pacestreak
# blog's project is `pacestreak-blog`
```

The available Cloudflare API token is **`zone:read` only** — it cannot purge
cache or change Pages config. Those need the dashboard.

---

## Load-bearing constraints

Breaking any of these fails silently in production rather than loudly in the
build. That is why they are written down.

### `default-src 'self'`, no `unsafe-inline`

Every site ships it, enforced. It is not defence in depth — it is the reason
there are no third-party runtime dependencies. **The user explicitly chose to
stay on Cloudflare's free tier and add no third-party services**, so do not
propose analytics, a font CDN, or an embedded widget.

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
DNS tab. This is why `app` and `api` have no records.

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
- **No `Co-Authored-By: Claude` trailers.** The user had these purged from
  history across all repos. Do not reintroduce them.
- Commit as `AlzyWelzy <welzyalzy@gmail.com>`.
- `SECURITY.md` is duplicated per-repository on purpose: health files in a
  **public** `.github` repo do not apply to **private** ones.

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
cd api && make dev            # api :8000, postgres :5432, redis :6379, worker
cd app && npm run dev         # :5173, talks to :8000
```

- **`make test` truncates every table in the database it points at**, including
  the dev admin account. Run tests against `pacestreak_test` (see
  `api/README.md#tests`).
- The dev compose override now mounts `alembic/`; run migrations with
  `make migrate`.
- Dev admin: `admin@pacestreak.com` (password given in the session that created it, not stored in git). **Change
  it.** Reserved handles (anything spelling "pacestreak") can only be assigned
  through `POST /v1/admin/users/{id}/official`.
- Email is `console`: verification and reset links print to the API log. There
  is no numeric OTP; TOTP is the only numeric code.
- Dev containers run as the host user (`HOST_UID`, default 1000), so
  `api/keys/*.pem` stay 0600. Production takes keys as PEM environment
  variables and mounts no key files.

## Outstanding work

**Read `HANDOFF.md` for the full list and reasoning.** In short:

### What's left (details in HANDOFF.md §4)

1. **Where the API runs** (the user's decision). `api/compose.prod.yaml` is
   ready for any Docker host behind a TLS proxy: `make prod-config`, then
   `make prod-up`.
2. **Email provider** (the user's decision). SMTP is production-ready; set
   `SMTP_*` and DNS (SPF, DKIM, DMARC).
3. **Rotate the dev admin password:** `make set-password email=admin@pacestreak.com`.
4. After deployment: custom domains, Upptime monitors, a daily
   `make backup` with off-host copies, and naming the providers on `/privacy`.
5. **Cloudflare edge cache purge** for five stale files on `www` (dashboard
   only; the token is `zone:read`).
6. The new privacy and terms pages are live but need a qualified review.

The questions already asked (hosting, email, waitlist), the options offered
and the user's answers are in `HANDOFF.md` §5. Don't re-ask; "decide later"
means still open, not declined. The user's checklist is `HANDOFF.md` §6.

Done since the audit: items 5–9 (per-post OG images, JSON-LD, prev/next,
tags, privacy/terms). Item 10, the waitlist, was **skipped by the user**.

The blog has **thirteen posts**.

### Operations quick reference

- Admin tooling: `make create-admin|set-password email=…`,
  `python -m app.cli set-role`.
- Backups: `make backup` (dumps, then test-restores);
  `scripts/restore.sh FILE`.
- CI runs in `api`, `app`, `web` and `blog`; `api` CI also runs `alembic check`.

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
