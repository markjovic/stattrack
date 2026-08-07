<!-- stattrack/README.md -->
# StatTrack

StatTrack is a **single-file web app** — everything lives in `index.html`: markup, styles, and script, with no build step, no dependencies, and no service worker. It is the front end for the Basketball Victoria dataset maintained in [`markjovic/sports-players-stats`](https://github.com/markjovic/sports-players-stats): player search and season-by-season profiles, team rosters and fixtures, venue day-grids and calendars, leaderboards, single-game records, and on-demand box scores. Current release: **Beta 0.74** (the version badge in the left rail is the single version marker — bump it with every release).

## Two copies, one app

The canonical copy of `index.html` lives here. A mirror copy sits at the **root of `sports-players-stats`**, and that mirror is the one users actually load, served from the data repo's Pages site so the app and its data share a base URL. Both copies are maintained by hand. Publishing a new version means updating the mirror **and then running Deploy Pages in the data repo** — a commit is not a publication (trap T23 over there): the live site only changes when a Deploy Pages run completes.

## Where the data comes from

StatTrack fetches everything at runtime from three places, all defined as constants near the top of the script:

- **`BASE`** — the active Pages origin (`markjovic.github.io/sports-players-stats`): sports-index, player files, search shards, venue lookups, indexes, records, and the per-season files of seasons still in play.
- **`ARCH`** — the archive Pages origin (`markjovic.github.io/sports-players-stats-archive`): the per-season files (`games/bv`, `team-stats/bv`, `leaderboard/season`) of locked seasons. Same browser origin as `BASE` (same host), so no CORS is involved.
- **`WRKR`** — the Cloudflare Worker, which proxies box scores and quarter scores from the spectator API on demand. Box scores are never stored server-side.

## The split routing contract (v0.74)

Per-season fetches are split across the two Pages origins because the dataset outgrew the Pages artifact ceiling. The routing rules, which any future change to this file must preserve:

**All per-season fetches go through `sgj(dir, sid)`, never through `gj` with a hand-built URL.** `sgj` resolves the origin via `seasonBase(sid)` — the single choke point: a season whose sports-index entry carries `archivedAt` fetches from `ARCH`, everything else from `BASE`. `archivedAt` is written by the data repo's graduation workflow only *after* the season was probed live on the archive, so routing by it is safe. On a miss, `sgj` retries exactly once on the other origin, which covers a stale index in a long-lived tab in either direction; a season missing from both origins still returns `null`, so callers' existing null-handling is unchanged. If the archive ever shards by era, only `seasonBase` changes.

**Where a season is served from stays invisible in the interface.** The locked marker — the grey `LOCKED` pill and muted season name on player-view season cards, and the padlock on fully-locked season groups in the pickers — keys on `locked` only, never on `archivedAt`. A season must look identical the day it moves origins. (The pickers' existing dot on groups with an active season is unchanged and coexists with the padlock.)

## Other conventions worth knowing before editing

Player privacy renders from the `private` boolean on the player record (`isPrivate` / `privBadge`) — private profiles get the badge and lose the PlayHQ link, and a private stub may carry a real captured name, so never infer privacy from the placeholder-name pattern. Stored ids may be 13-character truncations, 10-character legacy truncations, or full uuids; the resolver near the top of the script maps all three back to canonical. Recent-search history is the only thing kept in `localStorage`. The fetch-path contract (every path the app requests, which drives the data repo's deploy strip) and the fuller design notes live in `stattrack_html_design.md` alongside the pipeline documentation — if a fetch path is added or removed here, that document and the deploy strip must be updated in the same pass, or the new path simply won't be in the published artifact.
