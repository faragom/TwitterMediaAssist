# CLAUDE.md — project context for Twitter Media Assist (faragom fork)

This file is the single source of truth for continuing work on this project with
Claude. It captures the architecture, conventions, history and tooling so a fresh
session (e.g. after switching Claude organisation) loses no context. The user's
general preferences live in Claude's cross-session memory; the project-specific
parts are duplicated here on purpose so the repo is self-sufficient.

## What this is

Personal fork of the Firefox extension **Twitter Media Assist** (adds a download
button to videos/images/GIFs on twitter.com / x.com).

- Fork repo: **https://github.com/faragom/TwitterMediaAssist** (git remote `origin`).
  The account was previously `faragouji` and was renamed to `faragom`; old
  `faragouji/...` URLs redirect, so references to either name mean this repo.
- Upstream (git remote `upstream`): https://github.com/Flkalas/TwitterMediaAssist (MIT).
- Extension source is in [`src/`](src/); `src/manifest.json` (MV2) is the manifest root.
- Current version: **3.3.5** (git tag `v3.3.5`, published as a GitHub release with
  the signed `.xpi` attached).

## Working conventions (for this repo)

- **Language: English everywhere** — code, comments, commit messages, PR text, and
  this doc. (This overrides the user's general Catalan/Spanish default, which does
  not apply to this third-party, English-based codebase. Chat with the user is in
  Spanish.)
- **Every change goes on a branch + PR** into this fork's `master` (base
  `faragom:master`, **never** a PR to the upstream Flkalas repo). Merge with
  **squash + delete branch** once the user confirms it works. Don't stack branches.
- **Explain before system actions** (installs, elevation, restarts) and narrate the
  diagnosis before fixing. Code changes are made directly in a branch/PR.
- **Credentials are the user's**: never type or handle AMO API keys / tokens. Give
  the user the command; they run the signing step themselves.
- Commits end with the Claude Code co-author line; PR descriptions end with the
  "Generated with Claude Code" line (per the active Claude Code attribution policy).

## Architecture (how a download happens)

MV2 extension. Two worlds cooperate:

1. `content.js` (content script, `document_start`) injects `twitter_video_downloader.js`
   and then `inject.js` **into the page** (parallel load, `async=false`, so the
   downloader's functions exist before the interceptor runs).
2. `inject.js` (page world) monkey-patches `fetch`/XHR and intercepts Twitter's
   **GraphQL** responses, extracts media, and stores them in `sessionStorage` under
   the key `rectifying@gmail.com`, **deduplicated by URL**.
   - `rectifying@gmail.com` here is just an internal namespace string (it happens to
     be the upstream author's id). It is **NOT** the extension id and must not be
     conflated with it.
3. `twitter_video_assist_client.js` (content script) injects the download button and,
   on click, runs `downloadMediaObject()`, which reads that `sessionStorage`, filters
   to the clicked tweet's media, de-dupes, and sends one message per media to the
   background. If nothing was captured for the tweet, it falls back to an on-demand
   fetch (see below).
4. `twitter_video_assist_server.js` (background) receives the messages and performs
   the actual `browser.downloads.download()` (and GIF→canvas conversion when enabled).

Key files: `src/content.js`, `src/inject.js`, `src/twitter_video_downloader.js`
(parsers + download helpers, loaded in **both** the content and background worlds),
`src/twitter_video_assist_client.js` (UI + click), `src/twitter_video_assist_server.js`
(background), `src/options.html` / `src/options.js` (settings).

## Changelog of fork fixes (all merged to master)

- **#1** Fix: video downloaded twice on a video post page. Cause: the same video
  appears across more than one GraphQL response with a differing URL query param
  (e.g. `?tag=12` vs `?tag=14`), so URL dedup kept two entries. Fix: dedupe by a
  stable identity (`type` + `readableFilename`) in `downloadMediaObject()` before
  dispatching.
- **#2** Fix: button did nothing until page reload (media not intercepted — cache/
  bfcache, or interceptor loaded after Twitter's request). Fix: on-demand fallback
  when `sessionStorage` is empty, plus the parallel `async=false` script injection in
  `content.js`.
- **#3** Signing prep: extension id set to `twitter-media-assist-fork@faragouji`
  (the upstream `rectifying@gmail.com` id belongs to the author's AMO-listed add-on
  and can't be signed under this account), `data_collection_permissions: ["none"]`.
- **#4** Docs + `sign.ps1`.
- **#5** Downloads named `handle-tweetId.ext` (account handle first) **by default**
  across video/TS/GIF/image paths (the "readable file name" option now defaults ON);
  robust `@handle` resolution from either `core.screen_name` or `legacy.screen_name`
  (never `unknown`).
- **#6** Twitter GIFs download **only as MP4** by default (`isConvertGIF` defaults OFF).
- **#7** Fix: timeline download sometimes did nothing. Cause: media not intercepted
  **and** the #2 fallback used an internal GraphQL query id that Twitter rotated (now
  stale). Fix: primary fallback via the public `cdn.syndication.twimg.com/tweet-result`
  endpoint (`extractSyndicationMedia()`, token derived from the tweet id, no login or
  rotating query id); the GraphQL path (`extractGraphQlMedia()`) is the secondary
  fallback for login-gated tweets.
- **#8** `release.ps1` + release-flow docs.

## Fragile parts / gotchas

- **Do not change the extension id** `twitter-media-assist-fork@faragouji`. Changing
  it breaks AMO signing continuity and makes installed copies a different add-on.
- `extractGraphQlMedia()` / `archiveTweetDetailJson()` use a **hardcoded GraphQL query
  id and bearer**. Twitter rotates the query id, so this path can silently fail; it is
  only the *secondary* fallback. Interception + the syndication fallback are the
  reliable paths.
- `syndicationToken()` derives the syndication token from the tweet id with the same
  scheme Twitter's embeds use. If Twitter changes it, the primary fallback for
  not-intercepted tweets breaks (interception still works).
- `web-ext lint`: expect **0 errors**, ~11 warnings (pre-existing `innerHTML` usage and
  the Chrome-only `downloads.onDeterminingFilename`). Warnings don't block signing.

## Build / sign / release

All from the repo root. Node + `npx web-ext` required; `gh` CLI authenticated for releases.

```powershell
# Build + lint (unsigned; loadable as a temporary add-on via about:debugging)
npx --yes web-ext@latest lint  --source-dir src
npx --yes web-ext@latest build --source-dir src --artifacts-dir dist --overwrite-dest

# Sign for permanent install (AMO self-distribution, unlisted). Credentials via env;
# the USER runs this with their own AMO key/secret from
# https://addons.mozilla.org/developers/addon/api/key/
$env:WEB_EXT_API_KEY = "user:XXXXX:XX"
$env:WEB_EXT_API_SECRET = "your-secret"
./sign.ps1            # -> signed .xpi in dist/

# Publish a GitHub release tagged v<version> with the signed .xpi attached
./release.ps1         # reads version from manifest; refuses to overwrite an existing release
```

Install the signed `.xpi`: Firefox → `about:addons` → gear ⚙ → **Install Add-on From File…**.

Update flow: branch+PR → merge to `master` → bump `version` in `src/manifest.json`
(AMO rejects re-uploading the same version) → `./sign.ps1` → reinstall → `./release.ps1`.
See `README.md` for the fuller write-up.

## Local state NOT in git (kept in the backup zip)

`.gitignore` excludes build/signing artifacts. These are **not** in the repo:

- `dist/` — built packages and the AMO-signed `.xpi` files (3.3.2, 3.3.4, 3.3.5). The
  3.3.5 signed `.xpi` is also attached to the GitHub release; older ones are local only.
- `src/.amo-upload-uuid` — local state created by `web-ext sign`; remembers the AMO
  upload UUID for this add-on. Keep it for signing continuity. (Regenerated on next
  sign if lost, but keeping it avoids surprises.)

A `TwitterMediaAssist-backup.zip` with these was created for safekeeping during the
Claude org migration. The AMO API key/secret are the user's and live only in their
environment (never in the repo or the zip).
