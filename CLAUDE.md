# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Hoop Chaos, a browser basketball game, shipped as one file. `index.html` holds the inline CSS, the game (one inline `<script>`, about lines 648–19717, wrapped in an IIFE with `"use strict"`), and two base64 MP3 music blobs in `<script type="text/plain">` tags (lines ~643 and ~645). Those two lines are about 4.5 MB together: never print them. Use `cut -c1-200` or line ranges that skip them when reading or grepping.

There is no `package.json`, build step, linter, or test suite. Line numbers below are approximate.

## Commands

- **Run:** open `index.html` in a browser. No server or build is needed. Online play loads PeerJS from jsdelivr and talks to Supabase, so it needs internet access.
- **Build / lint / test:** none exist. Verify a change by loading the page and playing a round.
- **Single test:** not applicable (no test runner).
- **Deploy:** code comments say the build is published on GitHub Pages (origin: github.com/lachameleon/Hoop-ChaosV4). Nothing in the repo automates this.

## Architecture (big picture)

**One closed scope.** All game state and functions are private to the IIFE. Nothing is global except DOM nodes. Section banners (`/* ===== NAME ===== */`) mark the major areas, from MAPS (~900) through INPUT (~18992). Use them to navigate.

**Loop and update order.** `loop(now)` (~12927) runs a fixed timestep: it accumulates real time and calls `step(1/120)` repeatedly, then `paintChrome()`, `draw()`, and `netHostTick(now)`. `step(rdt)` (~12738) is the update pipeline. Some states short-circuit it: a falling floor, a rising rite, rewind, and the stoppage clock. Otherwise it calls a long list of `tickX(dt)` functions, one per mechanic, then the core player, bot, and ball logic.

**Mechanics follow a pattern.** Each special ability or hazard is a group of functions:
- a trigger (`useAbility`, `throwX`, `sendX`, `openX`, `summonX`)
- `tickX(dt)` to advance it
- `drawX()` to render it, called from `draw()`
- `sfxX()` for its sound

Mechanics attach their state to `teams[]`, `chars[]`, `ball`, and `round`. `newRound()` (~3655) rebuilds the per-round state. `rollMods()` (~3643) picks random modifiers from `MODS` (~2324) each round.

**Data tables** sit near the top of the script and drive everything else:
- `UNLOCK_ALL` (~668) and `PLAYABLE` (~678) are build flags.
- `NEWS_ID` and `NEWS` (~691–692)
- `MAPS` (~1543)
- `MODS` (~2324)
- `TEAM_COLOURS` (~3459)
- `SHOP` (~15861) and the roster

**Rendering** is canvas 2D with hand-drawn pixel art. The logical size is `BW × BH` (`BH = 180`, `BW` set from the viewport aspect ratio and clamped). Drawing helpers such as `block`, `disc`, `ring`, `text`, and `pixText` live at ~742–900. Each map has its own backdrop function (`nileRoom`, `pyramidRoom`, `tombRoom`, and so on, ~7936–8138). In online matches the host switches the world width to `NET_BW = 400`.

**Audio** is Web Audio: a shared context (`audioInit`, ~2350) with SFX built from oscillators and noise. Music uses `<audio>` elements fed from the base64 blobs in the two `bgm` script tags. `bgmFor`, `playTrack`, and the boss-music fades manage playback.

**Input** (~18992) maps keyboard keys and the on-screen pads (`#pads`) to `down(i)` / `up(i)`, indexed by team.

**Online play is peer-to-peer.** The code comment at ~15451 describes this:
- Matchmaking uses PeerJS over WebRTC (`loadPeerJS`, peerjs@1.5.4 from jsdelivr). Peer IDs are prefixed `PEER_PREFIX = 'hoopchaos-v1-'`.
- The host (left, red) runs the full simulation. `netHostTick` streams snapshots (`snapshotOnce`, ~16223).
- The guest (right, blue) only sends its button state (`netSendInput`) and draws received snapshots (`netGuestApply`). `down()` and `up()` branch on `NET.role`.
- Finding a stranger uses a Supabase `lobby` table over REST (~15509–15534).

**Accounts and saves.** Progress lives in `localStorage` under `SAVE_KEY = 'hoopchaos'`, with `BAK_KEY` as a backup and `PRE_KEY` as a snapshot frozen the first time a new release opens an old save. `sessionStorage` (`hc_in`, `hc_note`) carries a message across reloads.

When `CLOUD.url` and `CLOUD.key` are set (~16536), the account is stored on Supabase through RPCs at `/rest/v1/rpc`. The server checks each save against the last one it accepted and restores the stored value on tampering. With the URL empty, the game is local-only. A `hoopchaos_cloud` key in `localStorage` overrides the project for testing.

## Gotchas

- **Before publishing a build:** `UNLOCK_ALL` must be `false`. It is for testing only: when `true`, every character and currency is unlocked for free. `PLAYABLE` must be `false` for the normal build. `true` disables all network features.
- **Posting news:** add an entry at the top of `NEWS` and increment `NEWS_ID`. The unread-dot logic depends on that counter.
- **Never rename `SAVE_KEY`.** Progress carries across releases through that key.
- **Schema file missing:** the code says the cloud setup needs `hoop-chaos-supabase.sql` "next to this file". That file is not in the repo, so the server side cannot be reproduced from this checkout.
- **Comments carry design intent.** Many mechanic comments explain why the numbers are what they are (for example, the rope-swing speed cap and the bot targeting notes). Read the comment above a block before tuning it.
- **Online sync depends on both peers running the same code.** Host and guest each run the same `index.html`, so changes to shared simulation rules affect both sides together. Changing `PEER_PREFIX` separates old clients from new ones.
