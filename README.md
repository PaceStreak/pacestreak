# PaceStreak (workspace)

The umbrella repository for [PaceStreak](https://www.pacestreak.com), a habit and
streak tracker. It holds no product code of its own: each component is its own
repository, pinned here as a git submodule, and this repo carries the shared
working notes.

| Path | Repository | Branch | What it is |
| --- | --- | --- | --- |
| `web/` | [`PaceStreak/web`](https://github.com/PaceStreak/web) | `main` | Public site, `www.pacestreak.com` (live) |
| `blog/` | [`PaceStreak/blog`](https://github.com/PaceStreak/blog) | `main` | Blog, `blog.pacestreak.com` (live) |
| `status/` | [`PaceStreak/status`](https://github.com/PaceStreak/status) | `master` | Upptime status page, `status.pacestreak.com` (live) |
| `app/` | [`PaceStreak/app`](https://github.com/PaceStreak/app) | `main` | The product PWA, `app.pacestreak.com` (live) |
| `api/` | [`PaceStreak/api`](https://github.com/PaceStreak/api) | `main` | The API, `api.pacestreak.com` (live) |
| `infra/` | [`PaceStreak/infra`](https://github.com/PaceStreak/infra) | `main` | Topology, runbook and decision records |
| `.github/` | [`PaceStreak/.github`](https://github.com/PaceStreak/.github) | `main` | Org profile and health files |

## Documents here

- [`AGENTS.md`](./AGENTS.md): standing rules and traps for anyone working
  here, human or coding agent. Read it first. Each component also has its own
  `AGENTS.md` with that repo's commands. `CLAUDE.md` only imports `AGENTS.md`.
- [`HANDOFF.md`](./HANDOFF.md): the detailed history, with every decision
  and its reasoning, the questions asked and answers given (§5), and the
  owner's remaining checklist (§6).

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
commit. Deploys are unaffected either way: pushing to `main` in `web`, `blog`
or `app` deploys it on Cloudflare Pages, and pushing to `main` in `api`
publishes an image the server rolls out within a couple of minutes. Push a
component before the root commit that pins it.

`status` is an Upptime repo that commits to itself every few minutes, so its
pin here will always lag; that's expected. Bump it only when it matters.
