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

## 3. Second pass: the remaining list, implemented (25 September, later)

You asked for everything on the remaining list. You chose to **decide API
hosting and email later** and to **skip the waitlist**. Everything else is
done, except the cache purge, which needs the Cloudflare dashboard.

| Item | What was built | Why this way |
| --- | --- | --- |
| Production stack | `api/compose.prod.yaml`: required secrets, API on `127.0.0.1` only, a one-shot `migrate` service, read-only containers, `no-new-privileges`, memory limits, rotated logs. **Booted and smoke-tested in isolation** (migrate → healthy API → CLI admin → login with `__Secure-` cookies), then torn down. | Hosting is undecided, so it's host-agnostic: any Docker host behind any TLS proxy. |
| Key handling | Keys can be injected as PEM environment variables and are read once per process. The dev override runs containers as the host user; **private keys are back to 0600**. | This removes the `chmod 644` workaround properly. Production never mounts key files. |
| Email | SMTP: implicit TLS option, retries with backoff, Date/Message-ID/Reply-To/Auto-Submitted headers, RFC 8058 `List-Unsubscribe` plus a one-click POST endpoint. Production refuses `console` and refuses SMTP without TLS. | Provider-neutral, so choosing a provider later is configuration only. The docstring had cited RFC 8058, but no headers were actually sent. |
| Admin tooling | `python -m app.cli create-admin / set-password / set-role`, plus `make` targets. Passwords are prompted for, never passed as arguments. `set-password` signs out every session. | Replaces the throwaway scratchpad script. |
| Backups | `scripts/backup.sh` (checksummed, 0600, safe retention that never prunes the newest dump), `restore-check.sh` (restores into a scratch DB and compares the schema revision), and `restore.sh` (asks first). **Tested against the dev DB.** | A backup that has never been restored isn't a backup. |
| CI | `api`: ruff, pytest against Postgres and Redis services, `alembic check`, image build. `app`: `npm ci`, tests, build, and dist checks (404, noindex, no inline scripts). The blog also checks share cards. | Pushed over SSH, which workflow files require. |
| Code format | `api` Ruff-formatted in one formatting-only commit (`f9e558b`); 92 tests pass before and after. | Keeps `git blame` readable. |
| Privacy and terms | Rewritten on `web` for accounts and health data, written against the code: each table and why it exists, who sees what, age rules, 30-day deletion plus 14-day backup expiry, providers (API host and email sender "named before launch"), and terms covering health, use, moderation and liability. | Required before the first real signup. |
| Per-post share images | Built at build time with Satori and resvg into `dist/og/<slug>.png`. Noto Sans is vendored under the OFL. CI fails if a post has no card. | Audit item 5. Nothing runs in the browser. |
| Group mute | `group_members.muted` (migration `d32eee1ab7f7`). Push and email are skipped for that group; the inbox still receives everything. | Mute shouldn't lose anything. |
| Chart accessibility | A Chart/Table toggle on every bar chart and on the grid. The grid's toggle is sticky inside its horizontal scroller. | Values are available without hover or colour. |
| i18n | `src/lib/i18n.ts`: a typed catalog, `Intl.PluralRules`, locale-aware numbers, per-key English fallback. Navigation is migrated. | Adding a language becomes a data change. |
| Shortcut icons | Five distinct 96 px icons generated from Phosphor by `app/scripts/shortcut-icons.mjs`. | |
| Easing back after a pause | After a pause of 2+ weeks, a card suggests target − 1 for a fortnight. It's a suggestion only, never applied automatically. | |
| Admin "official" control | A sheet replaces `window.prompt`. | |
| Outbox tests | 9 tests against a real IndexedDB (`fake-indexeddb`), including an edit made while a push is in flight. | This was the most important untested logic. |
| Housekeeping | `tsconfig.tsbuildinfo` is untracked and ignored. The merged branches were deleted on GitHub after checking each was contained in `main`. | |
| Visual check | Logged in with headless Chromium and screenshotted Today, Training (pause), Progress, Recap, Data and Admin, and opened the pause sheet. This found and fixed the grid's off-screen toggle and the "SeptOct" label overlap. | |

**Not pushed.** All of this is committed on `main` locally in each repo. Pushing
`web` publishes the new privacy and terms. **Have someone qualified review
them first**: they're accurate to the code, but they aren't legal advice, and
they don't state a governing law.

---

## 4. What's left

### Needs your decision

1. **Where the API runs.** The stack is ready to go on any Docker host. Once
   you choose, fill `.env`, run `make prod-up`, and put a TLS proxy in front.
2. **Email provider.** Set `SMTP_*`, then add SPF, DKIM and DMARC for the
   sending domain.
3. **Change the dev admin password:** `make set-password email=admin@pacestreak.com`.

### Blocked on deployment

- Attach `app.pacestreak.com` and `api.pacestreak.com` as custom domains to
  real deployments (never hand-made DNS records).
- Then add both hosts to Upptime. Adding them before they exist would show a
  permanent outage, which happened once already with `pacestreak.net`.
- Schedule `make backup` daily and copy `./backups` off the host.
- Name the API host and email provider on `/privacy`.

### Needs the Cloudflare dashboard

- Purge the stale edge cache on `www` (five leftover files). The token is
  `zone:read` only.

### Deliberately not done

- The waitlist: skipped at your request.
- Volume, calorie or body leaderboards; third-party sync; analytics; pricing.
