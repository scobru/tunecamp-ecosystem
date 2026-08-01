# TuneCamp Ecosystem

Reference repo for AI agents (and humans) working across the TuneCamp network of projects. Answer "what exists, what does it do, how does it connect, what's next" without cloning every repo first.

**For agents:** read this file top to bottom before proposing changes or new ideas for any TuneCamp-adjacent project. `docs/tunecamp/` = vendored snapshot of core repo docs — read it for server/API/architecture specifics. Find something stale here, fix it in same change.

---

## Components

| Component | Repo | Role | Stack | Status |
| --- | --- | --- | --- | --- |
| **TuneCamp** | [scobru/tunecamp](https://github.com/scobru/tunecamp) | Core: self-hosted federated music server. Streaming, Subsonic API, ActivityPub federation, payments (Stripe + on-chain), free sample/sample-pack library, Lab apps host, MCP server, Collab (multi-artist collaborative track building), in-instance Chat lobby + E2E DM (`peerChatEnabled` toggle). Example instance: [sudorecords.scobrudot.dev](https://sudorecords.scobrudot.dev) | Node/Express, SQLite (better-sqlite3), React/Vite frontend | **Stable core**, several areas Beta/New — see `docs/tunecamp/STATUS.md` |
| **Sidecamp** | [scobru/sidecamp](https://github.com/scobru/sidecamp) | Desktop companion app: P2P content acquisition (Soulseek/torrents) and peer file-sharing, kept off the server so core stays clean/compliant. npm-workspaces monorepo hosting Sidecamp + Graphofone + Sidecamp CLI + shared packages. | Electron, npm workspaces | **Beta** |
| **Graphofone** | `scobru/sidecamp` → `apps/graphofone` (same monorepo, not a separate repo) | Standalone live-performance app: import a folder, arrange tracks as a graph, beat-matched crossfade transitions, perform. No P2P, no server. Split out of Sidecamp to isolate low-latency audio thread from network/disk I/O. | Electron, Web Audio (`packages/audio-engine`), React (`packages/graph-ui`) | **Active** |
| **Sidecamp CLI** | `scobru/sidecamp` → `apps/sidecamp-cli` (same monorepo, not a separate repo) | Headless terminal client for the same Sidecamp functionality without Electron: multi-source search/download (YouTube, SoundCloud, Bandcamp, archive.org, torrent, Soulseek, instance library), track upload, and the P2P sharing daemon (`sidecamp share`, connects via WebSocket `/ws/peer`) — for servers/headless boxes or scripting. Auth via `sidecamp login` (stores JWT). | Node.js (`commander`, `axios`, `webtorrent`, `andrade-soulseek-downloader`), published as global bin `sidecamp` | **New** |
| **Audiofabric** (Lab app) | [scobru/tunecamp-audiofabric](https://github.com/scobru/tunecamp-audiofabric) (fork of [rolyatmax/audiofabric](https://github.com/rolyatmax/audiofabric)) | Real-time 3D WebGL music visualizer, streams from a TuneCamp instance's Subsonic API. Deployed at [tunecamp-audiofabric.vercel.app](https://tunecamp-audiofabric.vercel.app), embedded via iFrame (`lab_apps` DB table, id 2). | Three.js, Web Audio API | **Built-in default** |
| **4-Track Recorder** (Lab app) | [scobru/tunecamp-4-track-recorder](https://github.com/scobru/tunecamp-4-track-recorder) (fork of [andreboekhorst/4-track-recorder](https://github.com/andreboekhorst/4-track-recorder)) | Browser 4-track recorder, overdub, latency compensation, `.4trk` save/load. Client-only, no server. Deployed at [tunecamp-4-track-recorder.vercel.app](https://tunecamp-4-track-recorder.vercel.app), embedded via iFrame (`lab_apps` DB table, id 1). | SvelteKit, Web Audio API | **Built-in default**. Note: `lab_apps` seed's `source_url`/`author` still credit upstream `andreboekhorst/4-track-recorder`, not the actually-deployed maintained fork `scobru/tunecamp-4-track-recorder`. Known drift, not yet fixed in core. |
| **TuneCamp Iris** (Lab app) | [scobru/tunecamp-iris](https://github.com/scobru/tunecamp-iris) | Air-Gapped Optical File Transfer via Fountain Codes & WASM. Transfer stems and keys via light without a network. Deployed at [tunecamp-iris.vercel.app](https://tunecamp-iris.vercel.app), embedded via iFrame (`lab_apps` DB table, id 3). | Vanilla TS, WebAssembly (zxing), Canvas API | **Built-in default** |
| **TuneCamp Beam** (Lab app) | [scobru/tunecamp-beam](https://github.com/scobru/tunecamp-beam) | Zero-Server WebRTC P2P Data Drops. Sends massive DAW projects directly phone-to-phone via local network by scanning an LZ-compressed SDP QR Code. Includes TuneCamp Lab SDK integration for Subsonic audio streaming. Deployed at [tunecamp-beam.vercel.app](https://tunecamp-beam.vercel.app), embedded via iFrame (`lab_apps` DB table, id 4). | Vanilla TS, WebRTC DataChannels, lz-string | **Built-in default** |
| **Design System** | `scobru/tunecamp-design-system` | Deprecated. Design tokens are now inlined directly in app styles. | — | **Deprecated** |
| **TuneCamp Website** | `scobru/tunecamp-website` | Landing page, Community Directory (`community.html`), Community Player (`player.html`), and Global Zen SEA Identity Portal (`profile.html`, unifies cross-instance passports via `wss://delay.scobrudot.dev/zen`). | Static HTML + Tailwind CSS (CDN), `scobru/zen` | **Live** |
| **FID (Fediverse-ID)** | `scobru/fid` | Self-sovereign identity & SSO protocol for ActivityPub/Fediverse. Zero-knowledge auth via Zen SEA only — a secp256k1 keypair derived from `alias:passphrase`, so the same two strings reproduce the identity on any device or portal. Deterministic per-domain ActivityPub keypair derivation. Reference implementation in TuneCamp core. **v4 dropped the WebAuthn/passkey source**: a passkey is bound to one Relying Party domain, so it forked one human into a separate identity per portal. | TypeScript (ESM), Zen SEA | **Early, reference impl in TuneCamp** |

## Architecture at a glance

```
                     ┌─────────────────────┐
   Lab apps (iFrame) │      TuneCamp        │  Federation (ActivityPub)
   Audiofabric ──────►  core server + DB    │◄──── other TuneCamp instances
   4-Track Recorder   │  (Node/Express,      │      (HTTP gossip discovery)
                       │   SQLite, React)     │
                       └──────────┬───────────┘
                                  │ Subsonic API / REST / Zen Passports
                                  │
                       ┌──────────▼───────────┐
                       │  Sidecamp (Electron)  │──── P2P (Soulseek/torrents)
                       │  desktop companion     │
                       │  ├── Graphofone        │  (live performance, split out
                       │  ├── sidecamp-cli      │   for isolation, no network)
                       │  ├── audio-engine pkg   │
                       │  └── graph-ui pkg       │
                       └───────────────────────┘
                       sidecamp-cli: same monorepo, headless (no Electron),
                       same REST + /ws/peer talk to core, own P2P daemon

  Design tokens are now inlined directly in app styles instead of a shared package.

  TuneCamp Website ──── static, queries /api/community/sites + Zen P2P Relay (delay.scobrudot.dev)

  FID Identity Layer ──── cross-instance identity & SSO
  ├─ fid-portal.vercel.app — reference identity portal (Zen SEA; replaceable, holds no user record)
  ├─ TuneCamp core — /api/auth/zen/* endpoints (challenge, link, SSO, verify)
  ├─ fid_registry table — tracks linked instances per user
  ├─ MCP server — FID auth via `Authorization: FID <zen_pub_key>`
  └─ tunecamp-website/profile.html — unified cross-instance profile aggregation
```

## Facts worth knowing before suggesting network-wide ideas

Full list in `docs/tunecamp/ARCHITECTURE-DECISIONS.md` (vendored copy of TuneCamp core's own agent rules). Highlights:

- **Instance Autonomy & Zen SEA Passports (FID Protocol).** Instance auth remains local (username + password + JWT, per-instance). Cross-instance identity unification and zero-knowledge SSO are achieved via **FID (Fediverse-ID)** (`@scobru/fid`) and **Zen SEA & Instance Passports** (`GET/POST /api/auth/zen/*`), binding local accounts to a global `scobru/zen` `~pubKey` on `https://fid-portal.vercel.app/` (or `tunecamp.org`) via cryptographic proof of ownership or SSO tokens without central user databases.
- **ZEN role in identity.** ZEN (`scobru/zen`) is used client-side for Zen SEA self-sovereign graph identity (`~pubKey/linked_instances`) via P2P relay nodes (e.g. `wss://delay.scobrudot.dev/zen` or community-hosted relays), keeping core server dependencies lightweight while supporting cross-instance public profile/release aggregation. See `docs/tunecamp/FID-IDENTITY.md`.
- **Federation via ActivityPub, not logins.** Interactions federate (follows, shares) Mastodon/Funkwhale-style; purchases/collections stay local to the artist's instance.
- **SQLite only, single writer.** No Postgres/Redis assumption for TuneCamp core's data layer, unless the idea is explicitly about the multi-machine scaling problem.
- **Lab apps are DB-backed** (`lab_apps` table + admin API/panel), not a static frontend array. New Lab apps go through `POST /api/admin/lab-apps` or a seed row in `database.ts`. See `docs/tunecamp/LAB.md`.

## Core Innovations

Key technical and architectural innovations already powering the TuneCamp ecosystem:

1. **FID Protocol (Fediverse-ID) & Zero-Knowledge Passports:** Deterministic `secp256k1` keypairs derived from `alias:passphrase` via `scobru/zen` (ZEN SEA). Provides cross-instance identity and zero-knowledge SSO without central databases or domain-locked passkeys (`GET/POST /api/auth/zen/*`).
2. **P2P Acquisition & Audio Engine Isolation (Sidecamp & Graphofone):** Strict decoupling of core self-hosted streaming server from P2P content acquisition (Soulseek, WebTorrent). Graphofone isolates the Web Audio real-time DSP thread from disk and network I/O.
3. **Decentralized P2P Profile & Track Discovery:** `tunecamp-website` uses P2P WebSockets (`wss://delay.scobrudot.dev/zen`) to discover, aggregate, and play federated artist profiles and release graphs across instances without central indexers.
4. **ActivityPub Federated Music Network:** Full ActivityPub federation (follows, track shares) enabling sovereign music nodes to interact across the Fediverse.
5. **Lab Apps Micro-Frontend Architecture:** DB-driven Lab App registration (`lab_apps`) enabling web audio apps (`Audiofabric` WebGL visualizer, `4-Track Recorder`) to run isolated inside iFrames while binding directly to instance audio streams.
6. **First-Class MCP Server & Zero-Trust Auth:** Built-in Model Context Protocol server in TuneCamp core authenticated via FID headers (`Authorization: FID <zen_pub_key>`).

## Ideas / network suggestions

Running log. New entries: idea, rationale, what it would touch. Prune once decided (built, rejected, superseded) instead of leaving stale.

- **Outbound MusicBrainz metadata contribution.** Existing metadata pipeline (`/api/metadata/search`) only pulls in — MusicBrainz, Discogs, iTunes, TheAudioDB. Invert it: self-released/indie releases published on a TuneCamp instance get submitted back to MusicBrainz as new entries, with the returned MBID stored on the local release. Closes a real gap — indie/self-released catalogs are chronically underrepresented in MusicBrainz, and no competing platform contributes metadata outward instead of just consuming it. Would touch: `/api/metadata` module (new submit path), release publish flow (trigger + store MBID), and MusicBrainz's own submission API/auth requirements (needs research — may require a registered editor account per instance, rate limits, moderation queue before entries go live).

## Maintaining this repo

New TuneCamp component (Lab app, companion app, deployable service):

1. Add row to **Components** table above.
2. If it has real docs, vendor under `docs/<component-name>/` (copy, not symlink — this repo may be read or published standalone, separate from source repos).
3. Update architecture diagram if component-to-component talk changes.
4. Note deployed URL and actual repo — watch for upstream-vs-fork drift (see 4-Track Recorder row for how to document it honestly instead of silently picking one).

`docs/tunecamp/` is a snapshot, not a live mirror — will drift from `scobru/tunecamp/docs` over time. Re-sync after significant TuneCamp core doc changes; don't trust it for anything version- or time-sensitive (endpoint lists, current version number).
