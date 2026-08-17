# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Bitcoin Education Archive (https://bitcoineducation.quest) — a static, framework-less
vanilla-JS single-page app served from GitHub Pages, fronted by Cloudflare Workers, with
Firebase (Auth / Firestore / Cloud Functions / Storage) as the backend. Content originated
as a curated Bitcoin Discord server and lives as JSON in `data/`.

There is no npm project at the repo root, no bundler, no transpiler, and no module system.
All browser code is plain ES5-ish script files that attach things to `window`.

## Commands

```bash
node tests/run-all.js            # full suite (syntax, HTML integrity, var scope, onclick)
node tests/test-syntax.js        # run one suite directly — that's the only granularity
./build.sh                       # rebuild bundle.src.js -> bundle.js (requires terser on PATH)
./deploy.sh "commit message"     # full production deploy (see caveats below)
git config core.hooksPath .githooks   # opt into the pre-commit test gate
```

`deploy.sh` is the maintainer's production path and is **not safe to run here**: it `cd`s to
a hardcoded `/root/simple-archive`, commits everything, force-pushes to `gh-pages` plus
codeberg/gitlab mirrors, and purges Cloudflare cache. Read it for the release contract, but
do the equivalent steps manually. Same for `tests/validate.js` (hardcoded `/root/simple-archive`,
not part of `run-all.js`).

Cloud Functions live in `functions/` (Node 22, `npm install` there); Workers each have their
own `wrangler.toml` under `workers/*/` and deploy with `wrangler deploy` from that directory.

## Build model — read before editing any `.js`

Three different kinds of `.js` file sit side by side in the repo root. Editing the wrong one
silently does nothing in production.

1. **Bundled sources.** The `SOURCES` list in `build.sh` (`channel_index.js utils.js
   ranking.js badges.js … app.js ux-patches.js`) is `cat`-ed in order into `bundle.src.js`,
   syntax-checked, then minified to `bundle.js`. Edit these files directly, then run
   `./build.sh`. Never hand-edit `bundle.js`, `bundle.src.js`, or `bundle.min.js`.
   Concatenation order is load order — a file may rely on globals defined by an earlier one.

2. **Minified standalone modules.** Files with a sibling `*.js.src` (`lightning.js`,
   `pvp.js`, `onboarding.js`, `irl-sync.js`, `nacho-closet.js`, `tips-integrations.js`,
   `wallet-router.js`, the `*-patches.js` set, …) are terser output. **Edit the `.src`, then
   re-minify over the `.js`** (`terser foo.js.src -o foo.js --compress passes=2 --mangle`).
   `build.sh` does not do this for you.

3. **Plain unminified modules** with no `.src` pair (`quests.js`, `forum.js`, `global-chat.js`,
   `features.js`, `learning-quests.js`, `nacho-qa.js`, `scholar.js`, …) — edit in place.

Watch out: `patches.js` and `timechain-tv.js` are minified but have **no committed `.src`**.
`timechain-tv.js.bak*` / `.working-backup` are stale unminified snapshots, not sources.

## Cache busting and the service worker

`index.html` references every script with a `?v=` stamp. `deploy.sh` bumps the stamp for any
file in its `LAZY_FILES` list that differs from HEAD, bumps `bundle.js`'s stamp, bumps
`CACHE_NAME` in `sw.js` (`btc-archive-vNNNN`), and copies `index.html` over `404.html`
(GitHub Pages serves 404.html for deep links, so the two must stay identical). If you change
a lazy-loaded file by hand, bump its `?v=` in `index.html` too or returning users keep the
cached copy. A stale stamp fails `deploy.sh`'s own verification step.

## Architecture

**Routing.** `window.go(id, btn, fromPopState)` in `app.js` (~line 3543) is the router. It
handles both content channels and app routes (`forum`, `marketplace`, `lightning`, `pvp`,
`timechain-tv`, `chat`, `dms`, `irl-sync`, `bitcoin-beats`, `learning-quests`, …). Clean
paths (`/channels/{id}`, `/app/{page}`) are parsed by `window._parseCleanUrl`, but pushState
uses hash routes so refreshes don't 404 on Pages. `_navGeneration` guards against
interleaved async navigations.

**Content.** `channel_index.js` defines the global `CHANNELS` map (id → `{cat, title, desc,
file}`); each entry points at a `data/*.json` file loaded on demand. Filename prefixes are
category groupings: `D_` = Bitcoin properties, `E_` = technical/economic topics, `H_` =
Discord-derived topic channels. Each JSON is `{cat, title, desc, msgs: [{text, link, img?}]}`.

**Everything is global.** HTML uses inline `onclick="fn()"`, so any function reachable from
markup must be a global (`window.foo = …` or a top-level `function`). This is enforced by
`tests/test-onclick-functions.js` and is why the CSP keeps `'unsafe-inline'` in `script-src`.
Two production outages came from touching this: adding a nonce to `script-src` makes browsers
drop `unsafe-inline` and kills every `onclick` in the page. Don't reintroduce one until the
onclick → `addEventListener` migration (Phase 2 in `workers/seo-router/worker.js`) is done.

**Auth + economy.** `ranking.js` owns Firebase init (config is inline and public by design),
the `LEVELS` ladder, `POINTS`, sign-in for every provider (Google GIS, Facebook SDK, GitHub,
Nostr NIP-07/nsec, Lightning LNURL-auth, email magic link), and `awardPoints(pts, reason,
channelId, tickets, streakFreezes, badgeId, extra)` — the single entry point for XP. Badges
are in `badges.js`. Daily gating everywhere must use `window.getDailyKey()` (day flips at
05:00 UTC); `functions/index.js` mirrors this with `getOffsetDateKey()` so server and client
agree.

**Firestore.** `firestore.rules` (~82 KB) is the real authorization layer and is written
defensively: `users/{uid}` is owner-only with a create-time field allowlist, economy fields
(points, sats, tickets, streak) are server-written only, and all leaderboard/profile lookups
go through the sanitized `public_profiles` mirror. Main collections: `users`,
`public_profiles`, `forum_posts`/`forum_replies`, `marketplace`, `dm_conversations`,
`global_chat_meta`, `beats_tracks`, `notifications`, `satoshi`, `pvp_matches`, `raid_bosses`.
Any new client write path needs a matching rule, and points must not be writable client-side.

**Cloud Functions** (`functions/index.js`, ~100 exports plus `functions/src/*` for PvP
resolution, raid bosses, Satoshi's Favor payouts, Telegram bridging). CORS is restricted to
`bitcoineducation.quest` / `btcedu.app` / `btcedu.quest`; admin endpoints use a timing-safe
`ADMIN_TOKEN` compare, and admin identity comes from a Firebase custom claim, not an email list.

**Cloudflare Workers** (`workers/*/`): `seo-router` sits in front of the site — it serves
pre-rendered HTML to bot user-agents, passes humans through to Pages, and sets the real
security headers including CSP (`firebase.json`'s headers only apply to Firebase Hosting).
Root `worker.js` is the Nacho AI worker (Workers AI + Brave Search, with prompt-injection
guards); `docs/nacho-*.js` are reference copies. Others: `embed-proxy`, `chat-bridge`,
`rss-feed`, `noderunners-proxy`, `pleb-underground`.

**Data feeds.** `scripts/*.sh` (ETF holdings/flows, MSTR, news ticker, X→Telegram) run from
cron on the maintainer's box and commit results — that's the source of the recurring
"Auto-update ticker headlines" commits. Note `.gitignore` lists `scripts/` even though the
scripts are tracked, so new files there won't be picked up by `git add` without `-f`.

## Tests — what they actually check

- `test-syntax.js` — `new Function(source)` over a hardcoded file list plus every inline
  `<script>` in `index.html`. Add new top-level JS files to its `JS_FILES` list.
- `test-html-integrity.js` — asserts specific element IDs (`usernameModal`, `hero`, `sidebar`,
  `msgs`, `leaderboard`, `forumContainer`, `questModal`) and that `bundle.js?v=` is
  referenced. Renaming those IDs breaks the build. Its direct-link route tests sit *after*
  `process.exit()` and never execute — dead code, don't be misled by them.
- `test-variable-scope.js` — catches specific production bugs where `isAnon`, `isRealUser`,
  or `settingsEmoji` leak out of function scope.
- `test-onclick-functions.js` — every `onclick=` handler in `index.html` must resolve to a
  function defined somewhere in the JS files.

## Conventions

- Security work is documented in `SECURITY.md` and the dated `SECURITY-AUDIT-*.md` files;
  historical fixes are marked inline with `[AUDIT FIX ...]` / `[SECURITY FIX]` comments —
  keep that tagging when fixing similar issues.
- Source files carry a `© 2024-2026 603BTC LLC` proprietary header (see `LICENSE`); preserve
  it when editing a file that has it.
- Production branch is `gh-pages` — that's what the Pages workflow (`.github/workflows/deploy.yml`)
  deploys and what `deploy.sh` pushes to. There is no `main`.
- User-facing HTML is built by string concatenation; anything user-supplied must be escaped
  (`utils.js` has the shared sanitizers, e.g. `_safeCover` for image URLs).
