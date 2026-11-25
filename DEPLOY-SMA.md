# Navidrome Tester Environment

Fork-owned deployment baseline. Canonical fork:
`https://github.com/sma-rl-dev/navidrome` (branch `tester-env-baseline`).
Upstream: `https://github.com/navidrome/navidrome` at v0.64.2
(`101145742f4762164202cb4858c9126258bdc463`).
Submodule: `apps/navidrome` in the training corpus.

## Quick Start

```bash
./tester-env deploy    # Build image from source and start the container (admin/admin ready)
./tester-env seed      # Generate the music library, scan it, add users/playlists/stars/radio/shares
./tester-env verify    # Assert the seeded state over the native API and the /app/ shell
./tester-env reset     # Remove container and both named volumes (keeps the image)
./tester-env stop      # Stop container, keep data (alias: down)
./tester-env status    # Container state, URL, credentials
./tester-env logs      # Tail container logs
./tester-env help      # Usage
```

Options: `--run-id <id>` scopes container and volumes, `--port <port>`
(default 53812). `IMAGE_TAG` selects the image; `RUN_ID` never enters the image
name.

## Deployment

- **Baseline:** v0.64.2, commit `101145742f4762164202cb4858c9126258bdc463`
  (latest stable GitHub Release, published 2026-09-24). `master` is 133
  commits ahead with no newer release, so the tag is pinned by SHA.
- **Dockerfile:** the upstream `Dockerfile`, unmodified, three real stages:
  `ui` (node:lts-alpine, `npm ci` from `ui/` then Vite 7
  `npm run build -- --outDir=/build`, workbox copied from node_modules, no
  network fetch), `build-alpine` (golang:1.27-alpine, CGO_ENABLED=1
  `go build -tags=netgo,sqlite_fts5` with the UI bundle bind-mounted, BuildKit
  cache mounts on `/go/pkg/mod` and `/root/.cache`), and the final
  alpine:3.22 image (curl/ffmpeg/mpv/sqlite/libwebp) with the binary at
  `/app/navidrome`. Upstream's own release pipeline builds this same file.
- **Image:** `${IMAGE_TAG:-tester-env-navidrome:dev}` (rl-env supplies a
  content-addressed tag). Identical trees collapse to one image ID.
- **Ports:** host `${PORT:-53812}` -> container 4533 (`ND_PORT=4533` is baked
  into the image).
- **Volumes:** `navidrome-data<run-suffix>` -> `/data` (SQLite `navidrome.db`
  + WAL, cache, JWT secret, config file) and `navidrome-music<run-suffix>` ->
  `/music` (generated fixture library, ~157 MB).
- **Services:** single container. No database, queue, cache companion or paid
  integration; SQLite plus the on-disk cache.
- **Environment:** `TZ=UTC`, `ND_ENABLEEXTERNALSERVICES=false` (no insights
  collector, deezer/lastfm/listenbrainz agents or remote login background),
  `ND_AUTHREQUESTLIMIT=0` (the default 5 logins / 20 s would punish repeated
  agent logins; Subsonic auth shares the limiter), `ND_PLUGINS_ENABLED=false`,
  `ND_DEVAUTOCREATEADMINPASSWORD=admin` (exact spelling: one `A` after
  `DEV`; `ND_DEVAAUTO...` is silently ignored), `ND_RECENTLYADDEDBYMODTIME=true`
  (paired with fixed fixture mtimes).
- **Credentials:** `admin / admin` (created on first start from
  `ND_DEVAUTOCREATEADMINPASSWORD`, so the create-admin wizard never appears).
  Non-admin fixtures: `maya / maya`, `theo / theo`.
- **Build cost (measured on this host):** cold build 2m51s end to end (npm ci +
  Vite 21 s, CGO Go build 57 s); Go-only mutation rebuild 10.5 s wall; `ui/`
  mutation rebuild 34.5 s. Two builds of the identical tree produced the same
  image ID.
- **Build topology note:** the SPA is embedded in the Go binary
  (`ui/embed.go` `//go:embed all:build`), so the UI stage must stay cached for
  cheap source mutations; Go-source scenarios only pay the Go build.

## Seed data

Theme: "Sundial Records", a small listening archive.

Music fixture, generated at seed time by the image's own ffmpeg (fixed tags,
fixed mtimes, 600x600 `folder.jpg` per album), so the volume is reproducible
per reset — 10 albums / 47 FLAC tracks / 6 album artists / 5 genres
(Jazz, Ambient, Funk, Folk, Electronic) / years 2017-2025 / durations
2:47-9:30:

| Album | Artist | Genre | Year | Tracks |
| --- | --- | --- | --- | --- |
| Blue Harbor | Marina Vale | Jazz | 2021 | 5 |
| Northern Lines | Oslo Field | Ambient | 2022 | 4 |
| Low Ceilings | Delta Groove | Funk | 2019 | 4 |
| Field Notes, Vol. 1 (disc 1/2 + 2/2) | Clara Voss & The Lantern Choir | Folk | 2020 | 6 |
| Night Bus | Delta Groove | Funk | 2023 | 5 |
| Paper Lanterns | Clara Voss | Folk | 2022 | 5 |
| Signal Bloom | Static Meridian | Electronic | 2023 | 5 |
| Tidal Works | Marina Vale | Jazz | 2024 | 6 |
| Static Meridian | Static Meridian | Electronic | 2017 | 4 |
| The Quiet Cartographer | Oslo Field | Ambient | 2025 | 3 |

File/folder mtimes are fixed 2025-01-06 .. 2025-10-06, so "recently added" is
`The Quiet Cartographer` first without any wall-clock dependence. Seed triggers
`GET /rest/startScan.view?fullScan=true` and waits for exactly 47 tracks in 10
albums.

App data, written through the app's own `/api` and `/rest` endpoints with the
admin JWT (no direct SQLite writes, no external service):

- Users: `admin` (admin), `maya` / Maya Chen and `theo` / Theo Alvarez
  (non-admin, `.example` emails, password == username, default library).
- Playlists: Late Night Drive (admin, public, 5 tracks: Last Stop, Downtown,
  Static Drift, Blue Harbor, First Snow), Sunday Morning (maya, public, 4:
  Riverbank, Treeline, Low Tide, Held Open), Focus Session (admin, private, 4:
  First Snow, Pale Horizon, Treeline, Compass Rose), Shop Floor Rotation (theo,
  public, 4: Low Ceilings, Brass & Velvet, Relay, Uptown Lights).
- Admin favourites/ratings: album stars Blue Harbor + Signal Bloom; song stars
  Last Stop + Riverbank + First Snow; album ratings Paper Lanterns 5, Tidal
  Works 4, Night Bus 3; song ratings Salt & Smoke 4, Treeline 2.
- Radio stations: Night Drive FM, Sunday Classical (reserved `example.net`
  URLs, inert data, nothing is fetched).
- Shares: `Night Bus (full album)` (downloadable) and
  `Late Night Drive playlist` (not downloadable), both expiring
  `2030-12-31T00:00:00Z`.

Ids are random per run and every step resolves rows by stable names/titles, so
`seed` is idempotent: users/radio/shares are skipped when present, playlists are
recreated to the exact track list, stars/ratings are reset to the same values.
Re-running seed wipes only the music volume.

Static manifest: `seed.manifest.json` (also mirrored at
`scenarios/navidrome/seed.manifest.json` in the corpus for the seed-gate
digest).

## Verify contract

`verify` logs in as `admin/admin` and asserts, over the same API the UI reads:
10 albums / 47 tracks / 6 artists, all 10 album names, the 6 artist names and 5
genres, a representative track title from every album, album metadata (Field
Notes, Vol. 1: 6 tracks across discs 1+2, Folk 2020; Tidal Works Jazz 2024 6
tracks; The Quiet Cartographer Ambient 2025 3 tracks), Recently Added first =
The Quiet Cartographer, Starred albums = Blue Harbor + Signal Bloom, Top Rated
order Paper Lanterns > Tidal Works > Night Bus, starred songs = First Snow /
Last Stop / Riverbank, longest song = The Quiet Cartographer 570 s, all 4
playlists with owner/public/track count, maya's non-admin login and 3-playlist
visibility (the private admin playlist is hidden), theo's non-admin login, 3
user rows, 2 radio stations, 2 shares with their contents labels, and that
`/app/` returns 200.

## Browser evidence

Attributed to browser-checker (the fork-integrator ran no browser and claims
none):

- Smoke (minimal 4-album seed, before the full seed): `admin/admin` at
  `http://172.16.0.1:53812/app/` reached the album grid with 4 albums / 8
  songs / 4 artists, opened an album detail and its song table, then favourited
  `Night Bus` and saw it under Favourites — a browser-visible state-changing
  workflow on the source-built stack.
- Populated state (full seed): the album table listed 10 albums with
  artist/year/song count, Night Bus showed its 5 tracks, Field Notes, Vol. 1
  showed Disc 1/Disc 2 with 3 tracks each, Album > Starred = Blue Harbor +
  Signal Bloom, Top Rated = Paper Lanterns 5*, Tidal Works 4*, Night Bus 3*,
  Song > Starred = First Snow/Last Stop/Riverbank, Playlists showed the four
  seeded playlists with owners/visibility (and maya saw only the three public
  ones), Sharing showed both shares, Radio showed both stations and Users
  (admin only) showed maya and theo as non-admin.
- Browser-side probes used during that session: toggling the `Night Bus`
  favourite and creating a disposable `Browser Probe` playlist. Neither
  artifact is part of the seeded state (the final visual check read starred
  albums Blue Harbor + Signal Bloom and exactly the 4 seeded playlists), and
  reset clears both.

Determinism: `reset -> deploy -> seed -> verify` was run twice and produced a
byte-identical visible-state fingerprint (albums/songs/artists with durations,
years, disc counts and flags; playlists with owner/public/count; users; radio;
shares with expiry; starred/top-rated/recently-added orders). A second `seed`
on the live stack was also clean.

## Known nondeterminism

Record ids are random strings and the SPA uses hash routes
(`/app/#/album`, `/app/#/playlist`, ...), so navigate through the UI and assert
visible text, never ids or id-derived URLs. The `random` album list is
intentionally random and must not be an oracle. The scan-progress indicator and
the dev activity panel are real async statuses but a 47-track library scans in
milliseconds — assert post-scan results instead. Play counts and
recently-played need `/rest/scrobble` traffic. The SPA is a PWA
(`vite-plugin-pwa`), so a stale browser profile can mask a rebuilt bundle: use
a fresh profile per run. Share expiry is a fixed date shown as a date, not a
relative countdown.

## Reset / cleanup safety

`reset` removes only `tester-env-navidrome<run-suffix>` and the
`navidrome-{data,music}<run-suffix>` volumes (name-guarded `docker volume rm`);
the image is kept for reuse (`scripts/rl-env gc --prune-images` reclaims tags).
All mutable state lives in those two volumes (SQLite + WAL, cache, JWT secret,
config file), so `reset && deploy` is a fresh admin/admin instance with an empty
library. No `alevern/chromium-browser` or `chromium-ls*` container is ever
touched and no broad prune is run.
