# TuneCamp Ecosystem

Reference repo for AI agents (and humans) working across the TuneCamp network of projects. Answer "what exists, what does it do, how does it connect, what's next" without cloning every repo first.

**For agents:** read this file top to bottom before proposing changes or new ideas for any TuneCamp-adjacent project. `docs/tunecamp/` = vendored snapshot of core repo docs — read it for server/API/architecture specifics. Find something stale here, fix it in same change.

---

## Components

| Component | Repo | Role | Stack | Status |
|---|---|---|---|---|
| **TuneCamp** | [scobru/tunecamp](https://github.com/scobru/tunecamp) | Core: self-hosted federated music server. Streaming, Subsonic API, ActivityPub federation, payments (Stripe + on-chain), free sample/sample-pack library, Lab apps host, MCP server. Example instance: [sudorecords.scobrudot.dev](https://sudorecords.scobrudot.dev) | Node/Express, SQLite (better-sqlite3), React/Vite frontend | **Stable core**, several areas Beta/New — see `docs/tunecamp/STATUS.md` |
| **Sidecamp** | [scobru/sidecamp](https://github.com/scobru/sidecamp) | Desktop companion app: P2P content acquisition (Soulseek/torrents) and peer file-sharing, kept off the server so core stays clean/compliant. npm-workspaces monorepo hosting Sidecamp + Graphofone + shared packages. Consumes tunecamp-design-system for theming (5-theme picker: dark/light/grey/nordic/nordic-dark). | Electron, npm workspaces | **Beta** |
| **Graphofone** | `scobru/sidecamp` → `apps/graphofone` (same monorepo, not a separate repo) | Standalone live-performance app: import a folder, arrange tracks as a graph, beat-matched crossfade transitions, perform. No P2P, no server. Split out of Sidecamp to isolate low-latency audio thread from network/disk I/O. Same DS theming as Sidecamp. | Electron, Web Audio (`packages/audio-engine`), React (`packages/graph-ui`) | **Active** |
| **Sidecamp CLI** | `scobru/sidecamp` → `apps/sidecamp-cli` (same monorepo, not a separate repo) | Headless terminal client for the same Sidecamp functionality without Electron: multi-source search/download (YouTube, SoundCloud, Bandcamp, archive.org, torrent, Soulseek, instance library), track upload, and the P2P sharing daemon (`sidecamp share`, connects via WebSocket `/ws/peer`) — for servers/headless boxes or scripting. Auth via `sidecamp login` (stores JWT). | Node.js (`commander`, `axios`, `webtorrent`, `andrade-soulseek-downloader`), published as global bin `sidecamp` | **New** |
| **Audiofabric** (Lab app) | [scobru/tunecamp-audiofabric](https://github.com/scobru/tunecamp-audiofabric) (fork of [rolyatmax/audiofabric](https://github.com/rolyatmax/audiofabric)) | Real-time 3D WebGL music visualizer, streams from a TuneCamp instance's Subsonic API. Deployed at [tunecamp-audiofabric.vercel.app](https://tunecamp-audiofabric.vercel.app), embedded via iFrame (`lab_apps` DB table, id 2). | Three.js, Web Audio API | **Built-in default** |
| **4-Track Recorder** (Lab app) | [scobru/tunecamp-4-track-recorder](https://github.com/scobru/tunecamp-4-track-recorder) (fork of [andreboekhorst/4-track-recorder](https://github.com/andreboekhorst/4-track-recorder)) | Browser 4-track recorder, overdub, latency compensation, `.4trk` save/load. Client-only, no server. Deployed at [tunecamp-4-track-recorder.vercel.app](https://tunecamp-4-track-recorder.vercel.app), embedded via iFrame (`lab_apps` DB table, id 1). | SvelteKit, Web Audio API | **Built-in default**. Note: `lab_apps` seed's `source_url`/`author` still credit upstream `andreboekhorst/4-track-recorder`, not the actually-deployed maintained fork `scobru/tunecamp-4-track-recorder`. Known drift, not yet fixed in core. |
| **TuneCamp Design System** | `scobru/tunecamp-design-system` | Shared UI/design-token package. Ships real TuneCamp branding (OKLCH tokens, violet primary/cyan-magenta-orange accents) plus a canonical DaisyUI v5 theme (`daisyui-theme.css`) covering 5 themes — dark/light/grey/nordic/nordic-dark — kept in sync with core's own DaisyUI themes. Consumed by Graphofone and Sidecamp (npm `file:`/`github:` dependency), and now piloted inside TuneCamp core's own webapp (`Panel` component on a couple of pages) as the shared token source between the Tailwind/DaisyUI core app and the plain-CSS Electron apps. | React + TypeScript + Vite | **Early, in active use** — 4 components (Button, Panel, Slider, Typography), no forms/modals/tables yet; core webapp migration is incremental page-by-page, not a bulk rewrite. |
| **TuneCamp Website** | `scobru/tunecamp-website` | Landing page, Community Directory (`community.html`, queries `/api/community/sites`), Community Player (`player.html`, aggregates/plays tracks across discovered public instances). | Static HTML + Tailwind CSS (CDN), no build step | **Live** |

## Architecture at a glance

```
                     ┌─────────────────────┐
   Lab apps (iFrame) │      TuneCamp        │  Federation (ActivityPub)
   Audiofabric ──────►  core server + DB    │◄──── other TuneCamp instances
   4-Track Recorder   │  (Node/Express,      │      (HTTP gossip discovery)
                       │   SQLite, React)     │
                       └──────────┬───────────┘
                                  │ Subsonic API / REST
                                  │
                       ┌──────────▼───────────┐
                       │  Sidecamp (Electron)  │──── P2P (Soulseek/torrents)
                       │  desktop companion     │◄── tunecamp-design-system
                       │  ├── Graphofone        │  (live performance, split out
                       │  ├── sidecamp-cli      │   for isolation, no network)
                       │  ├── audio-engine pkg   │
                       │  └── graph-ui pkg       │◄── tunecamp-design-system
                       └───────────────────────┘
                       sidecamp-cli: same monorepo, headless (no Electron),
                       same REST + /ws/peer talk to core, own P2P daemon

  tunecamp-design-system also feeds the TuneCamp core webapp above (Tailwind/DaisyUI
  theme + a couple of migrated components) — same token source across the Tailwind
  app and the plain-CSS Electron apps.

  TuneCamp Website ──── static, queries /api/community/sites on any public instance
```

## Facts worth knowing before suggesting network-wide ideas

Full list in `docs/tunecamp/ARCHITECTURE-DECISIONS.md` (vendored copy of TuneCamp core's own agent rules). Highlights:

- **No portable cross-instance identity.** Auth = username + password + JWT, per-instance. No SSO, no roaming cryptographic identity. Any "network account" idea must account for this or propose reintroducing SSO deliberately.
- **ZEN/Gun.js fully removed** from core (PR #370, 2026-06-15) — actively removed, not just deprecated. Do not suggest re-adding for identity/auth/discovery/social. A narrow future reintroduction (Phase C, not started) is planned only for ephemeral presence and real-time collaborative playlists.
- **Federation via ActivityPub, not logins.** Interactions federate (follows, shares) Mastodon/Funkwhale-style; purchases/collections stay local to the artist's instance.
- **SQLite only, single writer.** No Postgres/Redis assumption for TuneCamp core's data layer, unless the idea is explicitly about the multi-machine scaling problem.
- **Lab apps are DB-backed** (`lab_apps` table + admin API/panel), not a static frontend array. New Lab apps go through `POST /api/admin/lab-apps` or a seed row in `database.ts`. See `docs/tunecamp/LAB.md`.

## Ideas / network suggestions

Running log. New entries: idea, rationale, what it would touch. Prune once decided (built, rejected, superseded) instead of leaving stale.

- **Outbound MusicBrainz metadata contribution.** Existing metadata pipeline (`/api/metadata/search`) only pulls in — MusicBrainz, Discogs, iTunes, TheAudioDB. Invert it: self-released/indie releases published on a TuneCamp instance get submitted back to MusicBrainz as new entries, with the returned MBID stored on the local release. Closes a real gap — indie/self-released catalogs are chronically underrepresented in MusicBrainz, and no competing platform contributes metadata outward instead of just consuming it. Would touch: `/api/metadata` module (new submit path), release publish flow (trigger + store MBID), and MusicBrainz's own submission API/auth requirements (needs research — may require a registered editor account per instance, rate limits, moderation queue before entries go live).

- **Collaborative music-making Lab app.** Real-time or async collaborative composition/DAW-style Lab app, prior art referenced: shogun-isle. Lowest-risk of the three — fits the existing Lab app pattern (DB-backed via `lab_apps` table, see `docs/tunecamp/LAB.md`) instead of a new subsystem.

- **Instance passport — signed ActivityPub actor badge.** No portable cross-instance identity is a fixed constraint (username+password+JWT, per-instance). Don't fight it with SSO — use a signed ActivityPub actor object to prove "this artist is verified on instance X" as a portable *badge*, not a login. Federation already carries interactions (follows, shares) via ActivityPub, so this rides the existing transport instead of adding a new one. Needs research before building: ActivityPub actor signing/verification spec, how a receiving instance validates the signature without a shared identity provider, and what "verified" even means per-instance (admin-approved artist link? see `VisibilityGuardian.canPublishContent()`). Would touch: ActivityPub actor object (add signed badge field), federation crawler/receiving side (verify signature), profile UI (display badge).

- **In-instance chat lobby / E2E messaging as a toggleable module.** Core already has the wire protocol — `peer.service.ts` relays a lobby broadcast (no `toUsername`) and per-user DMs, E2E-encrypted client-side (server only relays opaque public keys, Curve25519/XSalsa20-Poly1305 via `tweetnacl` on the Sidecamp side). Today this chat is only reachable through the separate Sidecamp Electron app (P2P peer network, Soulseek-style DMs) — regular webapp/listener users on an instance have no chat UI at all. Idea: expose the same lobby + DM relay directly in the TuneCamp webapp, scoped per-instance (not cross-network P2P), gated by an admin on/off toggle (parallel to the `lab_apps` DB-backed enable pattern). Would touch: webapp (new chat UI component, E2E key generation/relay client-side), `peer.service.ts` (already does lobby+DM relay — reuse, don't fork), admin settings (module toggle + panel), and scoping the lobby to instance-local sessions only.

## Maintaining this repo

New TuneCamp component (Lab app, companion app, deployable service):

1. Add row to **Components** table above.
2. If it has real docs, vendor under `docs/<component-name>/` (copy, not symlink — this repo may be read or published standalone, separate from source repos).
3. Update architecture diagram if component-to-component talk changes.
4. Note deployed URL and actual repo — watch for upstream-vs-fork drift (see 4-Track Recorder row for how to document it honestly instead of silently picking one).

`docs/tunecamp/` is a snapshot, not a live mirror — will drift from `scobru/tunecamp/docs` over time. Re-sync after significant TuneCamp core doc changes; don't trust it for anything version- or time-sensitive (endpoint lists, current version number).
