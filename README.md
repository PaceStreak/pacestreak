# PaceStreak (workspace)

The umbrella repository for [PaceStreak](https://www.pacestreak.com), a workout
streak tracker. It holds no product code of its own: each component is its own
repository, pinned here as a git submodule, and this repo carries the shared
working notes.

| Path | Repository | Branch | What it is |
| --- | --- | --- | --- |
| `web/` | [`PaceStreak/web`](https://github.com/PaceStreak/web) | `main` | Public site, `www.pacestreak.com` (live) |
| `blog/` | [`PaceStreak/blog`](https://github.com/PaceStreak/blog) | `main` | Blog, `blog.pacestreak.com` (live) |
| `status/` | [`PaceStreak/status`](https://github.com/PaceStreak/status) | `master` | Upptime status page, `status.pacestreak.com` (live, public) |
| `app/` | [`PaceStreak/app`](https://github.com/PaceStreak/app) | `main` | The product PWA (built, not deployed) |
| `api/` | [`PaceStreak/api`](https://github.com/PaceStreak/api) | `main` | The backend (built, not deployed) |
| `infra/` | [`PaceStreak/infra`](https://github.com/PaceStreak/infra) | `main` | DNS, Cloudflare, runbook, decisions |
| `.github/` | [`PaceStreak/.github`](https://github.com/PaceStreak/.github) | `main` | Org profile and health files (public) |

## Documents here

- [`CLAUDE.md`](./CLAUDE.md): standing rules and traps. Read it first.
- [`HANDOFF.md`](./HANDOFF.md): the latest detailed handoff, with every
  decision and its reasoning, the questions asked and answers given (§5),
  and the owner's checklist (§6).

## Working with it

```bash
git clone --recurse-submodules git@github.com:PaceStreak/pacestreak.git
cd pacestreak

# Update every component to the tip of its tracked branch:
git submodule update --remote --merge

# Record those new commits here:
git add web blog app api infra status .github
git commit -m "chore: bump submodules"
```

A submodule is pinned to a commit, not a branch. **Committing inside `web/`
does not change what this repo points at** until you `git add web` here and
commit. Deploys are unaffected either way: pushing to `main` in `web` or `blog`
deploys them, exactly as before.

`status` is an Upptime repo that commits to itself every few minutes, so its
pin here will always lag; that's expected. Bump it only when it matters.
