> **This repository is retired.** Flametender lives in the
> [sampo monorepo](https://github.com/safwyls/sampo) now — `core/` (the
> shared console framework), `games/enshrouded/`, `cmd/flametender`,
> `cmd/flameagent`, and `web/flametender`. The port was verified against
> the live Enshrouded server on 2026-08-16, and the
> `ghcr.io/safwyls/flametender` / `ghcr.io/safwyls/flameagent` images
> (including `:latest`) publish from sampo since then. History is
> preserved in sampo via subtree import. This repo is archived
> read-only; file issues and PRs against sampo.

# Flametender (flametender)

A management console for self-hosted **Enshrouded** dedicated servers —
the third console on the reusable base shared with
[palcon](https://github.com/safwyls/palcon) (Palworld) and
[wildskeeper](https://github.com/safwyls/wildskeeper) (RuneScape:
Dragonwilds), provisioned through
[ilmari](https://github.com/safwyls/ilmari), the shared host service.

In Enshrouded you are Flameborn, and survival means keeping the Flame
lit. That is also this dashboard's whole job.

Enshrouded has no RCON, no HTTP admin API and no server console; all
native administration is `enshrouded_server.json` plus the in-game player
menu. Flametender therefore *derives* everything: process liveness and
uptime from its flameagent sidecar, the live player list from a state
machine over the server's log tail, configuration from the JSON at rest.
Commands that have no transport (broadcast, kick, on-demand save) answer
HTTP 501 with the honest reason instead of pretending — the reasons say
where each ability actually lives (kick/ban are in the in-game menu for
anyone holding a kick/ban-capable role password; saves happen every 10
minutes and on shutdown by themselves).

## What works today (Phase 1)

- **Overview** — status, power controls through the agent (stop is a
  clean save: the game writes the world on SIGINT), uptime/player
  vitals, log preview
- **Flameborn** — who's online (log-derived: the join line carries the
  SteamID64, a login line the display name), join/leave history and
  playtime, and the banned-accounts editor
- **World saves** — snapshot, download, scheduled backups of the
  `savegame/` directory (world blob + rolling copies + index)
- **Configuration** — `enshrouded_server.json` editor over the top-level
  and `gameSettings` scalars (never adds or removes keys, type-validated
  against the file, one `.bak`, atomic swap, JSON-validated at the agent
  so a bad edit can never make the game regenerate an *open* default
  config), the role-group editor (names, capabilities, reserved slots,
  per-group join passwords), and one-click admin-role password rotation
- **Server log** — live tail through the agent
- **Raise a server** — one-click provisioning through Ilmari (or a
  generated compose stack to deploy by hand): places the flameagent
  container, installs the game via SteamCMD (app 2278520, Windows depot,
  runs under Wine), seeds a private-by-default `enshrouded_server.json`,
  and starts it
- Shared base: users/roles/permissions, audit trail, Discord
  notifications, scheduled restarts, crash watchdog, SteamCMD update
  jobs, public status page
- **Installable** — the console is a PWA: a service worker (`web/public/sw.js`)
  caches the shell and content-hashed assets, never the API, and an
  "Install" offer appears in the rail and the mobile menu once the browser
  says it can be installed
- **Cloudflare Access SSO** (optional) — behind a Cloudflare Tunnel, the
  identity Access already authenticated becomes a console session, with
  the account created on first sign-in and no permissions until you grant
  them. Tokens are verified cryptographically, so the header cannot be
  forged by anything that reaches the origin directly; password login
  stays as break-glass. See `docs/cloudflare-access.md`
- **Installable** — the console is a PWA: a service worker (`web/public/sw.js`)
  caches the shell and content-hashed assets, never the API, and an
  "Install" offer appears in the rail and the mobile menu once the browser
  says it can be installed
- **Cloudflare Access SSO** (optional) — behind a Cloudflare Tunnel, the
  identity Access already authenticated becomes a console session, with
  the account created on first sign-in. Tokens are verified
  cryptographically, so the header cannot be forged by anything that
  reaches the origin directly; password login stays as break-glass. See
  `docs/cloudflare-access.md`

`docs/enshrouded-recon.md` records every externally-sourced game fact
with its confidence level, and its verification ledger tracks what a real
server has confirmed. `docs/roadmap.md` is the phased plan from here
(A2S presence, save rollback, the 1.0 wave).

## Running it

```sh
cp .env.example .env && export $(cat .env | xargs)
go run ./cmd/flametender          # backend on :8080
cd web && npm install && npm run dev   # frontend dev server
```

Production: `cd web && npm run build`, then `go build ./cmd/flametender`
(the Go binary embeds the bundle), or use the `Dockerfile` /
`docker-compose.yml`. TrueNAS SCALE: `deploy/truenas-app.yaml`, including
how to register flametender as an Ilmari client.

The game server itself runs under the `flameagent` sidecar
(`Dockerfile.flameagent`, Wine included — Enshrouded ships no Linux
server binary). In supervisor mode the agent *is* the server: it installs
the game with SteamCMD and runs `enshrouded_server.exe` under Wine as a
child process, enforcing the server name, port and role passwords into
the config before every start. The Raise-a-server wizard sets all of this
up; `docs/sidecar-agent.md` has the reference stack.

## Provisioning is Ilmari's

This console deliberately holds no Docker rights and ships no provisioner
of its own. One host, one Docker-socket holder: Ilmari places, rebuilds
and destroys the per-server containers for all three consoles, each under
its own client token, data root and image allowlist. Adding Flametender
to a host is a client registration in Ilmari's config — no Ilmari code
changes, which is exactly the promise its README makes.

## Tests

```sh
go test ./...
cd web && npm test
```
