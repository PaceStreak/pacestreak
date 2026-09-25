# PaceStreak: handoff (25 September 2026)

For the next person or Claude session working in `~/Documents/pacestreak/`.
Read `CLAUDE.md` first for the standing rules; this file records what was done
in the latest sessions, why each decision was made, and what is left.

---

## 0. State in one paragraph

The public site, blog and status page are live. The product is **built and
tested but not deployed**. The API (FastAPI, Postgres, Redis and a worker) has
about 110 routes and 87 passing tests. The app is a React 19 PWA. The
latest work is on branches, not `main`, so none of it is live yet:

| Repo | Branch | Commits on it |
| --- | --- | --- |
| `web` | `feat/product-site` | Site rebuild; copy for the new features; docs |
| `blog` | `feat/product-site` | Ten new posts (five committed by the user mid-session); changelog |
| `api` | `docs/current-state` | Pauses, recap, file import, calendar feed, official accounts; docs |
| `app` | `docs/current-state` | UI for all of the above; docs |
| `infra` | `docs/current-state` | Two decisions recorded |
| `.github` | `docs/current-state` | Org profile no longer says "not built yet" |

Some of these branches were pushed to origin by the user during the session.
None was merged. **Merging `web` or `blog` to `main` deploys it** (Cloudflare
Pages is Git-connected). The branch name `docs/current-state` undersells the
`api`/`app` branches, which carry real features, so rename or split them if
that matters for review.

`CLAUDE.md` has been updated to match reality. It had described `app` and
`api` as "docs only", and listed audit items 6–9 as open when they were done.

---

## 1. Decisions, and why

### Local environment (earlier in the session)

| Decision | Why |
| --- | --- |
| Explained that verification is a **single-use link**, not a numeric OTP | `EMAIL_BACKEND=console` prints the link to the API log. TOTP codes are the only numeric codes in the system. |
| Frontend on :5173 → API in Docker on :8000; stopped a stray API on :8001 | Two APIs against one database only cause confusion. |
| `chmod 644` on `api/keys/*.pem` | **Dev-only.** The container runs as uid 1001 and the files are owned by uid 1000. In production, fix ownership or inject secrets instead. |
| Created `admin@pacestreak.com` with a direct DB script, a temporary password (not stored in git) | Signup can't create an admin, and email verification can't complete without SMTP. **Rotate the password.** A `make create-admin` target is still worth adding. |
| Did **not** give the admin the handle `pacestreak` | Reserved handles block brand impersonation. That problem is now solved properly by official accounts (see below). |

### The website rebuild (`web`)

| Decision | Why |
| --- | --- |
| New pages `/features`, `/streaks`, `/social` and `/security` | The site described an idea, while the product had moved on. You asked for "an entire website telling all the features". |
| **All product claims read from `src/data/product.ts`**, with numbers checked against the code | Two claims had become false (a "47-day" streak, and "no feed, no leaderboard"). One file keeps claims auditable. Drafting against the code also caught three more errors before they shipped: a nonexistent `security@` address, "accounts start private" (the default is followers-only), and "the code is on GitHub for you to check" (the repos are private). |
| `/streaks` includes a **simulator** running `src/lib/chain.ts`, a TypeScript port of the engine's rules | Showing "rest doesn't break it" is more convincing than claiming it. The same module renders at build time, so the page is correct with JavaScript off. |
| The simulator script is a bundled external module; the mobile menu is a `<details>` element | `script-src 'self'` with no `unsafe-inline`. `check-html.py` still passes on every page. |
| The hero grid shows **weeks**, with rest days inside the streak | Streaks count kept weeks. |
| No pricing, no third parties, no new requests from `web` | Standing rules. |
| Checked with headless screenshots at 1400 px and 390 px | This caught the simulator overflowing on mobile, which was then fixed. |

### The blog (`blog`)

Ten new posts, each written from reading the code, not from memory:

1. Streaks count weeks
2. XP a heavier bar can't buy
3. Offline sync
4. Auth for health data
5. Social privacy
6. The worker and notifications
7. Data ownership
8. Pauses and file import
9. The website rebuild
10. Status report

Corrections made before publishing:

- One unverifiable "first version had this bug" anecdote was removed.
- The service-worker section had wrongly claimed an `/offline` fallback and
  tests using `fake-indexeddb`. Both were fixed to match the code.

Posts use markdownlint's underscore emphasis. They're dated 25 September at
distinct hours, so the ordering of previous/next links is deterministic.

### New features (the suggestion list you asked to implement)

| Feature | Decision | Why |
| --- | --- | --- |
| **Pause / injury mode** | A new week status, `paused`. It neither breaks nor extends the run, spends and earns no freeze, pays no XP, and is excluded from consistency. A pause covers every chain. | "Rest is training" has to survive a three-week injury, which freezes can't cover. Not extending the run keeps a pause from being worth anything on the leaderboards. |
| | A week is sheltered only if the pause covers **4 or more of its days** | Otherwise a one-day pause in every week would turn a target of 3 into 2. |
| | Starts up to 14 days back or 30 days ahead; lasts at most 12 weeks; **120 days per trailing year**; no overlaps; an open-ended pause is capped at 12 weeks | Without limits, a pause would hold a streak, and a leaderboard place, with no training. Backdating exists because people open the app days after getting hurt. |
| | A pause that has started can be **ended but not deleted** (except within 24 hours as an undo) | Deleting it would silently break a streak it had been holding. |
| | The worker sends **no nudges during a pause**; Today shows a "Streak paused" card | Telling an injured person to train is the behaviour the product exists to avoid. |
| | Included in the JSON export and import | Standing promise: all your data leaves with you. |
| **Planned rest on the grid** | Uses the existing `training_days` bitmask. Days outside the plan with no session are outlined as planned rest; paused days are hatched. Every state has a title and a legend entry. | Before, a 3-day plan looked like four failures a week. State is never shown by colour alone. |
| **Weekly recap** | `GET /v1/me/recap` plus a `/recap` screen. The existing Monday digest now calls the same builder and links to the screen. | One piece of code decides how the week went, so the notification and the screen can't disagree. It reports **no volume**, per positioning. |
| **GPX / FIT / CSV import** | `POST /v1/workouts/import`. GPX is parsed with `defusedxml`, FIT with `fitdecode`, CSV with forgiving column names. | Files instead of OAuth sync: no third party, no token store, no CSP change. |
| | Deterministic `uuid5` ids plus a 3-minute duplicate window | Re-uploading, or uploading the same run as GPX and then FIT, never double-counts. |
| | `source="import"`: counts for the streak and grid, never for challenges or rewarded records | Backfilling must not top a board. This follows the existing rule for JSON import. |
| | GPX elevation ignores climbs under 3 m; bare CSV dates are placed at local midday; a bare `distance` column is read as km | GPS noise inflates raw elevation gain. Midday means no timezone can shift the day. Common exports write distance in km. |
| | 15 MB and 5,000-session limits; rate limited; per-row problems are reported, never fatal | One bad lap in a five-year export shouldn't lose the other sessions. |
| **ICS calendar feed** | A token URL, shown once and stored only as SHA-256; create, rotate and revoke are recorded in the security history. It carries no notes, sets or metrics. Responses are `private`, rate-limited per token hash, and return 404 for both bad tokens and closed accounts. | Calendar apps can't send auth, so the URL is the credential. |
| | A shared RFC 5545 builder with proper escaping and 75-octet folding | The old export didn't escape backslashes or fold long lines. |
| | New setting `PUBLIC_API_URL`, which must be `https://` in production | The startup check fails loudly, not silently. |
| **Home-screen shortcuts** | Already existed ("Log a session", Progress, Rest timer). Added "Start a live workout" and "Last week's recap". | Only the gap needed filling. All shortcuts reuse one icon; distinct 96 px icons would be a small polish. |
| **Official @pacestreak account** | An admin-only, audited `POST /v1/admin/users/{id}/official`. It is the **only** path to a reserved handle. Revoking official status while holding a reserved handle requires moving to an ordinary handle. | The reservation is never loosened for anyone else. |
| | The brand check was **tightened**: it now matches the name anywhere in a handle or display name, after removing separators and folding look-alikes such as `5→s` and `0→o` | `real_pace_streak` and `Pace5treak` were previously allowed. |
| | A verified mark (icon plus text label) in the feed, people lists and profiles; an admin UI button | The admin UI uses `window.prompt` for the handle. That's functional but plain; a proper sheet would be nicer. |
| **Deliberately not built** | Volume or calorie leaderboards, third-party sync, analytics, pricing | Standing rules, and the "avoid" list in your message. |

### Engineering decisions made along the way

| Decision | Why |
| --- | --- |
| Tests run against a separate `pacestreak_test` database and Redis db 15 | `conftest.py` truncates `users … CASCADE`. Running it against the dev database would have wiped the admin account. Now documented in `api/README.md`. |
| Reverted Ruff's reformatting of six untouched files | The pre-existing files aren't Ruff-formatted. Formatting them would have buried the real diff. Only changed files were formatted. |
| The dev compose override now mounts `alembic/` | This was the root cause of the earlier `security_events` crash: the container only saw migrations baked into its image. |
| Every frontend read of a new stats field has a fallback (`?? []`) | The PWA caches `/me/stats` offline. A cached payload from an older build would otherwise crash the app. |
| A smoke-test import into the dev admin account was deleted afterwards | Leave the dev data as it was found. |
| No `Co-Authored-By: Claude` trailers; conventional commits; commits made as AlzyWelzy | Standing rules in `CLAUDE.md`. |

---

## 2. Verification done

- **API:**
  - 87 tests pass (67 existing, 20 new). They cover engine rules, pause rules
    and lifecycle, privacy, the recap, GPX, CSV and entity-bomb handling,
    idempotent import, the calendar lifecycle, ICS folding, brand
    reservation, and granting and revoking official status.
  - The migration upgrades, downgrades and upgrades again cleanly, and
    `alembic check` reports no drift.
  - The dev stack was rebuilt and migrated to `8a26a9caf99a`; health is OK.
  - Live smoke test passed for the pauses, recap, CSV import and calendar
    feed endpoints.
- **App:** `tsc -b`, `vitest` (6 pass) and `vite build` all succeed. The new
  screens were **not** visually checked in a browser against a logged-in
  session, so do that before merging.
- **Web:** built; `check-html.py` passes on 10 pages; screenshots checked at
  desktop and mobile widths.
- **Blog:** built; `check-html.py` passes on 44 pages; markdownlint is clean.

---

## 3. What's left

### A. Launch blockers (in order)

1. **Decide where the API runs.** This is your decision. It can't run on the
   Cloudflare free tier, and Python Workers rule out FastAPI. The options are
   a small VPS, a container platform, managed Postgres, or a rewrite onto
   Workers + D1. Everything below waits on this.
2. Deploy `app` to Pages and attach `app.pacestreak.com` to a real deployment.
   **Never create the DNS record by hand** (it would return 522). The
   `noindex` header, `Disallow: /` and the real 404 are already in place.
3. Production config: `COOKIE_DOMAIN=pacestreak.com`, `COOKIE_SECURE=True`,
   `DEBUG=False`, `TOTP_ENCRYPTION_KEY`,
   `CORS_ORIGINS=https://app.pacestreak.com`,
   `PUBLIC_API_URL=https://api.pacestreak.com`, `REQUIRE_VERIFIED_EMAIL=True`.
4. An email sender. This is a third-party question, so it needs your approval.
   Cloudflare Email is the option that stays within the rules.
5. **A privacy policy and terms written for accounts and health data**,
   published before the first real signup. The current pages cover only the
   website. They must also mention the calendar feed and file import.
6. Key ownership instead of `chmod 644`; rotate the admin password; add
   `make create-admin`.
7. Postgres backups with a tested restore; add `api./health` and `app.` to
   Upptime.
8. Purge the stale edge cache on `www` (five leftover files; dashboard only).

### B. Before merging these branches

- Log in to the dev app and click through: pausing and ending a pause, the
  grid legend, `/recap`, the Data page's import and calendar sections, and
  the official mark. None of this has been looked at in a browser yet.
- Decide whether to squash or rename `docs/current-state` in `api`/`app`.
- Commit or ignore `app/tsconfig.tsbuildinfo`, which is a build artifact
  showing as modified.

### C. Site audit leftovers

- **Item 5:** per-post OG images for the blog, generated at build time. Every
  post still shares `og.png`.
- **Item 10:** real waitlist capture. **Confirm with you first.** Once the app
  launches, "Sign up" may replace it.

### D. Engineering hygiene

- CI for `api` (pytest against Postgres and Redis services) and `app`. Neither
  repo has a workflow. Push workflow files over SSH (see `CLAUDE.md`).
- The outbox has no tests, although `fake-indexeddb` is already a dev
  dependency. The replace-on-edit and cursor logic deserves them.
- `57d4716` in `api` isn't a conventional commit.
- The admin UI uses `window.prompt` for official handles; replace it with a
  sheet.
- The whole of `api/app` isn't Ruff-formatted. Do that as one
  formatting-only commit, never mixed with other changes.

### E. Further feature ideas (not approved)

- A per-group mute for notifications.
- An accessibility pass on the charts: table views for BarChart and Heatmap.
- i18n scaffolding (units already convert only at display).
- Distinct shortcut icons.
- A pause history view in the recap.
- A "coming back" plan after a long pause: a lower target for the first two
  weeks, suggested rather than imposed.

**Still deliberately not recommended:** volume, calorie or body leaderboards;
Strava or Apple Health OAuth sync; analytics or embeds; anything about
pricing.
