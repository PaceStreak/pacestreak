# PaceStreak: handoff (25 September 2026)

For the next person or Claude session working in `~/Documents/pacestreak/`.
Read `CLAUDE.md` first for the standing rules; this file records what was done
in the latest sessions, why each decision was made, and what is left.

---

## 0. State in one paragraph

The public site, blog and status page are live. The product is **built and
tested but not deployed**: the API (FastAPI, Postgres, Redis and a worker) has
about 210 routes and 188 passing tests; the app is a React 19 PWA with 98
unit tests, and every new screen was also driven end to end in a real browser. Everything is on `main` in every repo, and the root repo pins each
component. **Pushing `main` in `web` or `blog` deploys it** (Cloudflare Pages is
Git-connected). What blocks launch is two decisions only you can make, where
the API runs and which email provider sends mail, plus the checklist in §6.
The third and fourth rounds of features (§3b, §3c) were built on 25 September;
weigh-ins and weight trends (§3d) the fifth round (§3e) and the sixth (§3f) on 26 September.

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

**Pushed to `main`** in every repo at your request, including `web`, so the
new privacy policy and terms are live. They're accurate to the code, but
they aren't legal advice and don't state a governing law. **Have someone
qualified review them**, and update them the moment a hosting or email
provider is chosen.

---

## 3b. Third pass: pre-launch features (25 September, evening)

You asked what else was worth building before deploying, then: "Implement
all of them. Properly production/industrial/professional grade level."

Some of the list already existed and was **not rebuilt**: 2FA recovery codes,
new-device sign-in alerts, offline logging with a sync queue, streak
milestones, a 4-week consistency score, the app-icon badge, the rest timer and
"last time" hints. Where they had gaps, the gaps were closed.

| Feature | What was built | Key decisions |
| --- | --- | --- |
| Passkeys | WebAuthn registration and usernameless sign-in (button and email-field autofill), rename/remove, security events | py_webauthn, a library, not a service. RP id = `app.pacestreak.com`, not the apex, so no other subdomain can ask for the credential. Challenges in Postgres, single use. Password required to add or remove. A passkey sign-in satisfies 2FA. |
| Recovery codes | "New recovery codes" in Settings, low-count warning | The API supported it; the UI never exposed it. |
| Health | `/health` (liveness), `/health/ready`, `/health/worker`; worker heartbeat in Postgres; per-job results on the admin Metrics tab | Heartbeat in Postgres, not Redis (Redis is best-effort). Public endpoints show a verdict only, never job names or errors. |
| Monthly backup | Opt-in reminder on the 1st, 09:00 local; opens a one-tap download in the app | **No data and no token in the email or inbox.** The download still needs a signed-in session. |
| Balanced weeks | Chain requirements ("at least 2 runs and 1 strength day") | History like targets, so old weeks are never re-judged. A week's score is its weakest part. |
| Consistency | 12- and 52-week windows next to the 4-week one | The number that survives a broken streak. |
| Year in review, record history | `/review`, a timeline per personal record | Attendance only, never volume. |
| Travel mode | `travel` pause reason; Today offers to follow the phone's timezone or pause | Logged sessions keep their dates. Offered once per place. |
| Icon badge | Unread, days still to go this week, or off | Hidden while paused. |
| Training plans | 4 conservative templates, custom plans, week editor, Today card | Plans never change what the streak counts; completion is read from the log; a session moved within the week still counts. One plan at a time. The terms now say they are general suggestions. |
| Interval timer | Intervals, EMOM, Tabata | Position computed from the clock, so a locked phone comes back correct. Wake lock while running. |
| Progression hints | Now also for bodyweight reps and timed holds | Past the top of the range it suggests a harder variation, not a longer set. |
| Tags and search | Private tags; search over notes, titles, exercises and #tags | Search is on-device, so it works offline. |
| Gear | Shoes/bikes/other, replacement distance, per-discipline default | Mileage is summed from the log, never stored. |
| Buddy streaks | Invite, accept, shared streak, side-by-side progress | Only with an accepted follow either way. A buddy sees progress and "paused", never sessions or reasons. Blocking ends it. Ending notifies nobody. |
| Group streaks | Shared group streak, threshold 50-100% set by owner/admin | Same judgement for every viewer. |
| Encouragement | 6 preset messages | No free text, so nothing to moderate. Only to people who follow you, buddies or group mates, once a day each. |
| i18n | All signed-out screens and API errors through the catalog; `rich()` for sentences with links; guard test | The remaining ~900 in-app strings are migrated as screens are touched. There is no second language yet. |
| Accessibility | Automated control-labelling pass; contrast measured for every text token on every surface in both themes | One unlabeled link fixed; three light-theme colours lifted to AA. |
| Web | `/changelog`; features, security, privacy and terms updated | The changelog says plainly that nothing is live yet. |

Also: export version 2 carries tags, gear, plans, chain requirements and
buddies. `make recompute-all` backfills `user_stats.recent_weeks` after
deploying, and should be run once on first deploy.

## 3c. Fourth pass: pre-launch hardening and the rest of the list (25 September, night)

You asked "what are all the features we can implement", then "Implementing
them all", then "continue now properly … check everything properly".

| Area | Built | Key decisions |
| --- | --- | --- |
| Terms versions | `TERMS_VERSION`, a gate when it changes, `POST /me/terms` | The gate steps aside on the export screen. Existing acceptances backfilled. **Bump the version with any material change to /terms or /privacy after launch.** |
| Recovery | `/recover`: a 2FA recovery code sets a new password | Only for accounts with 2FA; everyone else goes through support. Same answer whether or not the account exists. |
| Email change | Security → Email address | Needs the password; takes effect only via the link to the new address; the old address is told. |
| Crash reports | Anonymous reports to our own API; a real error screen | Path only, never the query string; grouped; capped at 1000 groups. |
| Abuse view | Admin → Ops: failed attempts by IP and account, sign-ups by IP | Keyed hashes; the account shown only if it exists; 30-day retention. |
| Getting started | A checklist on Today | Ticks come from real state, never from a tap. |
| Smart reminders | Nudge an hour before your usual time | Mode, not model; never in quiet hours; falls back to the fixed hour. |
| Monthly goals, rest days | Progress card; "Resting today" in the log sheet; a dot on the grid | Neither touches the streak. |
| Comeback, deload, freeze note | Coach cards | Suggestions only; deload needs 8 rated sessions. |
| Supersets, warm-ups | Live workout | Rest waits for the last exercise in a superset; ramps from the empty bar in plate steps. |
| Splits | From GPX/FIT, on the session screen | Interpolated at each kilometre; private. |
| Plan sharing | Export/import as a file | Routines embedded by value; other people's custom exercises dropped. |
| Plan challenges | "One plan" challenge kind | Everyone gets a copy; app-logged sessions count; joining switches your running plan (the app says so). |
| Coach plans | Coach tab: progress and "Suggest a plan" | Needs the member's sharing consent; the member starts it. |
| Buddy nudge | 5pm, buddy short with ≤2 days left | Once per week, never about a paused buddy. |
| Announcements | Group owners/admins | Plain text, 5 a day, nothing between members to moderate. |
| Not built | Exercise illustrations | Needs real artwork; placeholders would make the app worse. |

**Found by driving the app in a browser, and fixed:** the local containers
were running an image without `webauthn` (a rebuild fixed it; CI and
production build fresh); a false "Travelling?" card for timezone aliases;
"just now ago"; "1 members"/"1 days"; one-box week strips; a double-indented
switch; a flaky sync test (passed 25 stress runs after the fix); the www
header button wrapping at 360px. The dev database has no test accounts left.

## 3d. Weigh-ins and trends (26 September)

You asked for weight tracked several times a day (after waking, before and
after training, before bed), sets/PRs with progression graphs and % change,
gamification, and research into requested features. Sets, PRs with % gain,
record history, e1RM charts and XP already existed; the gap was body weight.

| Feature | What was built | Key decisions |
| --- | --- | --- |
| Weigh-ins | New `weigh_ins` table: many per day, `moment` = waking / pre_workout / post_workout / bedtime / other. `PUT /v1/weigh-ins/{id}` (client id, so retries update), `GET`, `DELETE`. Backdate up to 30 days. | Weight swings a kilo or more within a day, so readings are only comparable within one moment. |
| One place for weight | Migration `02c71d3228fb` moves `body_metrics.weight_kg` to weigh-ins at local midday, moment `other`, and **drops the column**. | Safe because nothing is launched. Downgrade restores each day's mean. |
| Trend | Body screen: daily dots, 7-day average line (new `LineChart`, Chart/Table toggle), filter by moment, 7/30/90-day change in unit and %. | Changes compare averages; "Not enough history" instead of a false figure. Gain/loss is not coloured good or bad. |
| Exercises | % change in e1RM, top set and volume across the charted sessions. | |
| Export | Version 3 adds `weigh_ins` (JSON and `weigh_ins.csv`); v1/v2 daily weights import as weigh-ins. Imports dedupe. | |
| Privacy | Weight stays out of XP, badges, boards and other people. **Your choice this round.** The "Measured" badge counts distinct logged days across both tables. | Rewarding the number pushes people to cut. |

Research-backed ideas offered, not yet built, in ranked order: trend
projection and milestone goals (as Happy Scale and Libra do); private progress
photos; weekly hard sets per muscle against a target range; deload suggestions
on a stall; soreness and pump check-ins; offline queueing for body entries;
consistency-only quests (e.g. five morning weigh-ins in a row), PR streaks
per four-week block, and a monthly strength recap.

## 3e. Fifth pass: the researched list (26 September)

You asked for research into what the industry offers and what people want,
then "Add them all". Research sources were Hevy, Strong, Fitbod, Alpha
Progression, MacroFactor Workouts, Happy Scale, Libra, and writing on
streak design ("streak creep"). Four ideas were excluded by standing rules:
AI form checks and camera rep counting (a hosted model or third party),
wearable/Strava/Health sync (third-party sync; a PWA can't reach HealthKit),
watch apps and widgets (native only), and weight-based rewards (your call in §3d).

Two things I had told you wrongly: quests did **not** already exist (my search
matched "request"), and sets per muscle with a 10-20 band **did**.

| Feature | Where | Key decisions |
| --- | --- | --- |
| Stall resets | `app/src/lib/training.ts` `suggestNext` | Same top weight three sessions without a rep gained, or twice under the range floor: suggest -10%. Shown in orange. |
| Auto-fill | Setting, off by default | Ticking an empty set logs the suggestion instead of last time. |
| Stall card | `coach.ts` `stalledLift` | Four sessions over 14+ days, none beating the first by 1%: a lighter week on that lift only. |
| "What showing up did" | `coach.ts` `consistencyGain` | Streak of 4+ weeks and the most-trained lift up 2%+. |
| Recovery map, untrained muscles | Progress | Days since primary work, sets this week. Says plainly it isn't a recovery measure. |
| Strength standards | Records, opt-in | Men's/women's bodyweight-multiple tables for squat, bench, deadlift, OHP. Off until chosen; private. |
| Search | Exercise picker | One typo forgiven per word of 4+ letters; recents rank first; "At <gym>" filter. |
| Gyms | New `gyms` table, `workouts.gym_id`, Settings > Gyms | Equipment, plates, bar; first gym is default; plate calculator uses its plates. |
| Pinned notes | `exercise_notes` | On the live workout and exercise pages. |
| Offline body writes | IndexedDB v2 `requests` store | Only idempotent PUT/DELETE queue; 4xx is dropped, not retried. Flushed on every sync. |
| Typed sets | `parseShorthand` | 100x5, 100x5x3, 3x5@100, 5@100, 12. |
| Weight goal | `weight_goals` | Starts from the 7-day mean; milestones; projection from 4 weeks' regression. **No rewards.** |
| Progress photos | IndexedDB `photos` store | Never uploaded (nothing untrusted under pacestreak.com). Re-encoded, which strips EXIF location. Lost with browser data; the UI says so. |
| Weekly quests | `api/app/game/quests.py` | 3 a week from 7, same for everyone, 15 XP each, recomputed from history. Paused weeks offer none. Weigh-in quest only once you've weighed in before that week. Hidden when gamification is off. |
| PR streak | Same module; badge "Always improving" (3/6/13) | Four-week blocks anchored to 2024-01-01; the open block never breaks it. |
| Monthly recap | `/me/recap/month`, `/recap/month` | Best e1RM this month vs before it. No volume. |
| Streak wager | `streak_wagers`; `compute_chain(wagers=...)` | Target+1 days; kept earns a freeze within the cap; missed costs nothing. One a calendar month, placed before the week's first session. Main chain only. Exported, **not imported**. |
| Similar boards | `user_stats.weekly_days_4w`, `scope=similar` | Opted-in people within one day a week of your 4-week average. |

Also fixed: the XP screen recomputed XP separately and would have omitted
quest XP; it now reads `snapshot().xp_items`. The website had claimed 23
achievements while the code had 25; both pages now read `facts`.

Migration `0f73caa084e4` round-trips and `alembic check` is clean. Export v3
carries gyms, notes, goal and wagers. Screens were checked in headless
Chromium on a throwaway account, deleted afterwards.

## 3f. Sixth pass: programming depth, hybrid training, reach (26 September)

Researched again (Runna, RP Hypertrophy, Edge/Hypla, retention studies), then
"Implement them all".

| Feature | Where | Key decisions |
| --- | --- | --- |
| Race plans | `api/app/training/race.py`, `POST /plans/race` | Pure builder. Long run +10%/week, every 4th week 80%, and it holds before a lighter week so the week after returns to the same level (a bug found and fixed in testing). Taper 1-2 weeks, then a recovery week. Refuses build-ups shorter than 4/5/8/12 weeks; starts later rather than exceed 25 weeks. |
| Training blocks | `training_blocks`, `/blocks` | RIR 3 to 1 across working weeks, last week lighter (RIR 4, sets halved). Feeds `targetRpe = 10 - RIR` when a routine sets none. One at a time. Not imported. |
| Soreness/pump check-in | `workouts.soreness`, `pump` | Optional; two agreeing check-ins suggest a set more or fewer. |
| Adjust today | `AdjustToday.tsx` | Lighter / hot / unwell / short / another routine / rest. All still complete the plan day; sessions get a tag. Merges "not feeling 100%" and the heat/illness toggle. Weather data would need a third party, so it's manual. |
| Readiness | `readiness` table | Sleep, energy, soreness 1-5; low score offers Adjust today. Today or the last two days only. |
| Conflicts, training load | `app/src/lib/load.ts` | Load = minutes x effort (5 if unrated). Spike > 1.5x the 4-week average. |
| Heart rate | Importers, `workouts.avg_hr/max_hr/hr_zones`, `profiles.max_hr` | Zones at 50/60/70/80/90% of max; gaps count at most 30 s. |
| First fortnight | Getting started | Shown only while achievable. |
| Reflections | `week_reflections` | On the recap, searchable. |
| Share images | `app/src/lib/shareImage.ts` | Canvas on device, share sheet or download. Attendance only. |
| Languages | `app/src/lib/locales/{es,de}.ts` | Only the catalogued screens (auth, navigation, errors) are translated; the rest is English and the setting says so. **Needs native-speaker review.** A test keeps placeholders and link tags identical. |

Also fixed: the sign-in page showed "Something went wrong" on load when the
background passkey autofill request failed; it now fails quietly.

## 4. What's left

### Needs your decision

1. **Where the API runs.** The stack is ready to go on any Docker host. Once
   you choose, fill `.env`, run `make prod-up`, and put a TLS proxy in front.
2. **Email provider.** Set `SMTP_*`, then add SPF, DKIM and DMARC for the
   sending domain.
3. **Change the dev admin password:** `make set-password email=admin@pacestreak.com`.
4. **Review the Spanish and German translations** with a native speaker before
   launch (`app/src/lib/locales/`). Most in-app screens aren't in the catalog yet.

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

---

## 5. Every question asked, the options offered, and your answers

Kept here so the next session doesn't ask again, and doesn't treat a
"decide later" as a "no".

### Questions asked with the question tool

| # | Question | Options offered | Your answer | What it means now |
| --- | --- | --- | --- | --- |
| 1 | Where should the API (FastAPI + Postgres + Redis + worker) run in production? | **Small VPS (recommended)**: about €5 a month (e.g. Hetzner or DigitalOcean); a production compose stack with Caddy, backups and a deploy script. **Container platform**: Fly.io, Railway or Render; less ops, more cost, one more vendor. **Decide later**: build everything host-agnostic and leave deploy unwired. | **Decide later** | `api/compose.prod.yaml` runs on any Docker host behind any TLS proxy. Nothing is tied to a vendor. **Still open; it blocks launch.** |
| 2 | Which email sender for verification and password-reset mail? | **Generic SMTP (recommended)**: harden the existing SMTP backend; works with Zoho, which already handles `hello@`, or any provider. **Cloudflare Email**: stays within the Cloudflare-only rule, but its sending API is newer, so it would need a new HTTP backend. **Decide later**: leave `console`; make SMTP production-ready but unconfigured. | **Decide later** | SMTP is hardened and provider-neutral; picking one is configuration only. Production refuses to start on `console`. **Still open; it blocks launch.** |
| 3 | Build the real waitlist signup on www (Pages Function + KV)? It would be the first network request `web` ever makes. | **Skip it (recommended)**: keep `mailto:` until launch, when "Sign up" replaces it. **Build it**: Pages Function + KV, double opt-in, rate limited. | **Skip it** | Audit item 10 is closed as won't-do. `web` still makes no network requests. |
| 4 | Most of the request already existed; the gap was body weight (one per day, no moments, no % change). How to proceed? | **Build weigh-ins + write research (recommended)**. **Research doc only first**. **Build everything proposed**. | **Build weigh-ins + write research** | §3d. The research list is waiting for you to choose. |
| 5 | Should weight tracking stay outside gamification? | **Keep it private (recommended)**: reward logging, never the number. **Allow weight-goal badges**, still never public. | **Keep it private** | Standing rule reaffirmed: no weight-based XP, badges or goals rewards. |
| 6 | (In conversation) "Add them all" after the researched list | The ranked list of 18 in §3e | **All of them** | §3e. Excluded items stay excluded by standing rules. |

### Choices you made in conversation

| When | What you said | What was done |
| --- | --- | --- |
| Setting up locally | Asked how email verification ("OTP") works and how to log in | Explained that verification is a single-use **link**; with `console` email it prints to the API log. TOTP is the only numeric code. |
| Setting up locally | "Create an admin account with email admin@pacestreak.com" | Created, with a temporary password (not stored in git). Now also possible with `make create-admin`. |
| Setting up locally | "with port 8000 for backend api" / "my docker containers are running" | Standardised on the Docker API on :8000 and stopped a stray API on :8001. |
| Setting up locally | Hit "password must be at least 16 characters" | Kept the 16-character minimum and used a longer password. |
| Setting up locally | "That handle is reserved, I'm giving it pacestreak?" | Kept the reservation. Later solved properly with **official accounts**: an admin-only, audited grant that is the only path to `@pacestreak`. |
| Handoff 1 | "Write a detailed handoff…" | `HANDOFF.md` |
| Rebuild | "Update all the docs, rebuild the web repo as a full product site, write lots of blogs" | New site pages, 10 new posts, and every README, ARCHITECTURE and CHANGELOG updated. |
| Features | "Implement all these [suggested features] … production grade" | Pause/injury mode, planned rest on the grid, weekly recap, GPX/FIT/CSV import, ICS feed, shortcuts, official account. |
| Git | "Push all code to GitHub main, merge everything" | Fast-forward merges to `main` in all six repos; Cloudflare deployed `web` and `blog`. |
| Git | "Make this root a repo with submodules, named `pacestreak`, private" | `PaceStreak/pacestreak` (private), with seven submodules plus `CLAUDE.md`, `HANDOFF.md` and `README.md`. The dev admin password was removed from the docs before the first commit. |
| Remaining list | "Implement them all, production grade" | The whole of §3. |
| This push | "Push all to GitHub on main, update all md files and the handoff" | This section, §6, and the push, including `web`'s legal pages. |
| During that work | `/compact` typed mid-turn | It's a command you run yourself; it couldn't be run from inside the turn. |
| After that | "What can we do about deployment now? And mails?" | Recommended: app on Cloudflare Pages (with the API, not before); API on one small VPS with `compose.prod.yaml` behind a **Cloudflare Tunnel** (no open ports, free). Hosts offered: **Oracle Cloud Always Free** (free, fiddly sign-up), **Hetzner CX22** (~€4/month, recommended), DigitalOcean/Linode (~$6). Mail: Zoho's free plan can't send SMTP; offered **Zoho ZeptoMail** (recommended: Zoho is already on `/privacy`), Amazon SES (cheapest, sandbox approval), Resend/Brevo free tiers (one more vendor). **No answer yet: both still open.** |
| After that | "What about IONOS servers?" | Fine for this stack. Pick the **VPS** product (not Cloud Server or web hosting), the **4 GB** tier, an EU data centre, and check the minimum term and renewal price. Hetzner is still slightly easier (monthly, no commitment). **Still open.** |
| After that | "How does this project use the scheduler and worker?" | Explained `app/worker.py`: one process, six jobs per tick, a Redis lock, dedupe keys, `--once` for cron. |
| After that | "Before we deploy, what features can we implement?" then "Implement all of them" | §3b. |
| After that | "What are all the features we can implement?", then "Implementing them all" and "continue now properly … check everything" | §3c. Four groups offered (pre-launch, streaks, training, social) plus a "not recommended" list: native apps, Strava/Garmin sync (third party), search-indexed public profiles. All four groups built; illustrations skipped as needing real art. |

### Standing rules that shaped the answers (from `CLAUDE.md`)

- Cloudflare free tier, and no third-party services, analytics or embeds.
  That's why hosting and email are genuine decisions, not defaults.
- No pricing claims anywhere.
- No `Co-Authored-By: Claude` trailers; conventional commits; commits made as
  AlzyWelzy.
- Never create DNS records ahead of a real deployment (you'd get a 522).

---

## 6. Your checklist

In order. Nothing below needs code, only your decisions and accounts.

1. [ ] **Change the dev admin password:**
   `cd api && make set-password email=admin@pacestreak.com`.
2. [ ] **Get the privacy policy and terms reviewed** by someone qualified
   (`web/src/pages/privacy.astro`, `terms.astro`). Add a governing-law
   clause if the reviewer wants one.
3. [ ] **Decide where the API runs** (question 1 above). Then:
   - generate production keys: `openssl genrsa` for JWT, `make vapid` for push,
     and a Fernet key for `TOTP_ENCRYPTION_KEY`;
   - fill `.env` (mode 0600) and run `make prod-config` until it's clean;
   - `make prod-up`, behind a TLS proxy forwarding to `127.0.0.1:8000`;
   - run `make recompute-all` once (backfills buddy and group streak data);
   - leave `TERMS_VERSION` at `2026-09-25` for launch; bump it with any
     later material change to /terms or /privacy;
   - attach `api.pacestreak.com` via that host (not a hand-made DNS record).
   - Passkeys are bound to `app.pacestreak.com`. Don't set `WEBAUTHN_RP_ID`
     to anything else, and never change it after launch.
4. [ ] **Decide the email provider** (question 2 above). Set `SMTP_*` and add
   SPF, DKIM and DMARC DNS records for `pacestreak.com`.
5. [ ] **Name both providers** on `/privacy` (the "Service providers"
   section) and push `web`.
6. [ ] **Deploy the app**: create the Pages project for `PaceStreak/app` and
   attach `app.pacestreak.com` as a custom domain.
7. [ ] **Schedule backups**: a daily `make backup` via cron or a systemd timer,
   with `./backups` copied off the host.
8. [ ] **Add monitors** for `api.pacestreak.com/health/ready`,
   `api.pacestreak.com/health/worker` and `app.pacestreak.com`
   in `status/.upptimerc.yml`, only once they respond.
9. [ ] **Purge the Cloudflare cache** for `www` (Caching → Purge; five stale
   files).
10. [ ] After launch, flip `released` to `true` in
    `web/src/data/changelog.ts` so the changelog stops saying "not live".
11. [ ] Optionally, give the brand an official account: create it, then
    **Admin → People → Make official**, with handle `pacestreak`.
