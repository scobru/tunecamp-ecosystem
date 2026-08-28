# TuneCamp Ecosystem

Overview of the TuneCamp network of projects: **what exists, what it does, how the pieces talk, and what we're thinking of building next.**

This repo holds no code and no vendored documentation — only this file. It answers "what is there" without cloning nine repos first. For anything version- or time-sensitive (endpoint lists, current behaviour, threat models), go to the source repo's own docs; the canonical set lives in [`scobru/tunecamp` → `docs/`](https://github.com/scobru/tunecamp/tree/dev/docs), in English with an Italian mirror under `docs/it/`.

**For agents:** read this file top to bottom before proposing changes or new ideas for any TuneCamp-adjacent project. It records decisions that are easy to contradict by accident. Find something stale here — fix it in the same change.

## What exists

| Component | Repo | Role | Stack | Status |
| --- | --- | --- | --- | --- |
| **TuneCamp** | [scobru/tunecamp](https://github.com/scobru/tunecamp) | Core: self-hosted federated music server. Streaming, Subsonic API, ActivityPub federation, payments (Stripe + on-chain), free sample/sample-pack library, MCP server, Collab (multi-artist collaborative track building), Single Artist portfolio mode (alongside Record Label and Community). Example instance: [sudorecords.scobrudot.dev](https://sudorecords.scobrudot.dev) | Node/Express, SQLite (better-sqlite3), React/Vite frontend | **Stable core** at `5.5.0`, several areas Beta/New — see [`docs/STATUS.md`](https://github.com/scobru/tunecamp/blob/dev/docs/STATUS.md) |
| **Sidecamp** | [scobru/sidecamp](https://github.com/scobru/sidecamp) | Desktop companion app: P2P content acquisition (Soulseek/torrents) and peer file-sharing, kept off the server so the core stays clean and compliant. npm-workspaces monorepo hosting Sidecamp + Sidecamp CLI. | Electron, Capacitor (Mobile), npm workspaces | **`0.27.2`** |
| **Sidecamp CLI** | `scobru/sidecamp` → `apps/sidecamp-cli` (same monorepo, not a separate repo) | Headless terminal client for the same Sidecamp functionality without Electron: multi-source search/download (YouTube, SoundCloud, Bandcamp, archive.org, torrent, Soulseek, instance library), track upload, and the P2P sharing daemon (`sidecamp share`, connects via WebSocket `/ws/peer`) — for servers, headless boxes, or scripting. Auth via `sidecamp login` (stores a JWT). | Node.js (`commander`, `axios`, `webtorrent`, `andrade-soulseek-downloader`), published as the global bin `sidecamp` | **New** |
| **FID (Fediverse-ID)** | [scobru/fid](https://github.com/scobru/fid) | Self-sovereign identity & SSO protocol for ActivityPub/Fediverse. Zero-knowledge auth via Zen SEA only — a secp256k1 keypair derived from `alias:passphrase`, so the same two strings reproduce the identity on any device or portal. Deterministic per-domain ActivityPub keypair derivation. Reference implementation lives in TuneCamp core. **v4 dropped the WebAuthn/passkey source**: a passkey is bound to one Relying Party domain, so it forked one human into a separate identity per portal. | TypeScript (ESM), Zen SEA | **`4.0.0`**, early; reference impl in TuneCamp |
| **TuneCamp Website** | [scobru/tunecamp-website](https://github.com/scobru/tunecamp-website) | Landing page, Community Directory (`community.html`), Community Player (`player.html`), and Global Zen SEA Identity Portal (`profile.html`, unifies cross-instance passports via `wss://delay.scobrudot.dev/zen`). | Static HTML + Tailwind CSS (CDN), `scobru/zen` | **Live** |
| **TuneCamp Iris** | [scobru/tunecamp-iris](https://github.com/scobru/tunecamp-iris) | Air-gapped optical file transfer via fountain codes and WASM — move stems and keys with light, no network. Deployed at [tunecamp-iris.vercel.app](https://tunecamp-iris.vercel.app). Standalone — no longer embeddable in a TuneCamp instance. | Vanilla TS, WebAssembly (zxing), Canvas API | **Standalone** |
| **Wormhole** | [scobru/wormhole](https://github.com/scobru/wormhole) | Secure private remote file transfer (IPFS/WebRTC). Deployed at [wormhole.scobrudot.dev](https://wormhole.scobrudot.dev). Standalone — no longer embeddable in a TuneCamp instance. | — | **Standalone** |
| **Design System** | `scobru/tunecamp-design-system` | Deprecated. Design tokens are inlined directly in app styles now. | — | **Deprecated** |

## Architecture at a glance

```
                                ┌──────────────────────┐
                                │      TuneCamp        │  Federation (ActivityPub)
                                │  core server + DB    │◄──── other TuneCamp instances
                                │  (Node/Express,      │      (HTTP gossip discovery)
                                │   SQLite, React)     │
                                └──────────┬───────────┘
                                │ Subsonic API / REST / Zen Passports
                                │ /ws/peer
                     ┌──────────▼───────────┐
                     │ Sidecamp (Electron)  │──── P2P (Soulseek/torrents)
                     │ desktop companion    │
                     │ └── sidecamp-cli     │
                     └──────────────────────┘
                     sidecamp-cli: same monorepo, headless (no Electron),
                     same REST + /ws/peer to core, own P2P daemon

  TuneCamp Website ──── static; queries /api/community/sites
                        + Zen P2P relay (delay.scobrudot.dev)

  FID identity layer ──── cross-instance identity & SSO
  ├─ fid-portal.vercel.app — reference identity portal
  │                          (Zen SEA; replaceable, holds no user record)
  ├─ TuneCamp core — /api/auth/zen/* (challenge, link, SSO, verify)
  ├─ fid_registry table — tracks linked instances per user
  ├─ MCP server — FID auth via `Authorization: FID <zen_pub_key>`
  └─ tunecamp-website/profile.html — cross-instance profile aggregation
```

## Facts worth knowing before suggesting network-wide ideas

These are settled decisions. Contradicting one by accident is the most common way a good-sounding idea turns out to be unbuildable. The full list is in core's own [`docs/ARCHITECTURE-DECISIONS.md`](https://github.com/scobru/tunecamp/blob/dev/docs/ARCHITECTURE-DECISIONS.md).

- **Instance autonomy, with Zen SEA passports on top.** Instance auth stays local (username + password + JWT, per instance). Cross-instance identity and zero-knowledge SSO come from FID (`@scobru/fid`) binding a local account to a global `scobru/zen` `~pubKey`, by cryptographic proof rather than a central user database.
- **ZEN's role is client-side identity, not server infrastructure.** `scobru/zen` provides the self-sovereign graph (`~pubKey/linked_instances`) over P2P relays such as `wss://delay.scobrudot.dev/zen`. This keeps the server's dependency surface small while still allowing cross-instance profile and release aggregation.
- **Federation carries interactions, not logins.** Follows and shares federate Mastodon/Funkwhale-style; purchases and collections stay local to the artist's instance.
- **SQLite only, single writer.** No Postgres or Redis in core's data layer, unless the idea is explicitly about the multi-machine scaling problem.
- **Lab apps were removed from core, in full**, on 2026-08-28: the sandboxed-iFrame embedding mechanism (`lab_apps` table, `/api/lab-apps`, `/lab` route, admin panel, PostMessage bridge) is gone — low adoption did not justify the maintenance surface for a single-maintainer project. Audiofabric and 4-Track Recorder (the two apps built specifically for that integration) were dropped from the ecosystem entirely, local clones and all. Iris and Wormhole remain real standalone web apps (see **What exists**) — they never depended on Lab, just no longer embeddable in a TuneCamp instance. Do not propose re-adding a Lab-style embedding surface, or re-adding Audiofabric/4-Track Recorder, without revisiting why they were cut.
- **Chat is not TuneCamp's problem.** Peer chat — lobby, rooms, E2EE DMs and its cross-instance federation — was removed from core on 2026-08-24, along with the `@tunecamp/chat` library and Sidecamp's copies of it. Messaging belongs to [linda-pear](https://github.com/scobru/linda-pear), a serverless P2P messenger on the Holepunch stack. TuneCamp stays music, federation and publishing. Do not propose features that assume an in-instance chat surface.

## Core innovations

What is genuinely novel here, as opposed to assembled from parts:

1. **FID protocol and zero-knowledge passports.** Deterministic secp256k1 keypairs derived from `alias:passphrase` via Zen SEA. Cross-instance identity and SSO with no central database and no domain-locked passkeys.
2. **P2P acquisition kept off the server.** Strict separation between the self-hosted streaming core and P2P acquisition (Soulseek, WebTorrent), which lives in Sidecamp.
3. **Decentralized profile and track discovery.** `tunecamp-website` aggregates and plays federated artist profiles across instances over P2P WebSockets, with no central indexer.
4. **ActivityPub federated music network.** Full federation of follows and track shares, so sovereign music nodes interact with the wider Fediverse.
5. **First-class MCP server with zero-trust auth.** An MCP server built into core, authenticated by FID headers rather than a shared secret.

## Ideas and open threads

A running log — this is the half of the repo that is about the future rather than the present. Each entry states the idea, why it is worth doing, and what it would touch. **Prune an entry once it is decided** (built, rejected, superseded) rather than leaving it to rot.

### Ideas

- **Outbound MusicBrainz metadata contribution.** The metadata pipeline (`/api/metadata/search`) only pulls in — MusicBrainz, Discogs, iTunes, TheAudioDB. Invert it: a self-released record published on a TuneCamp instance gets submitted back to MusicBrainz as a new entry, with the returned MBID stored on the local release. This closes a real gap, since indie and self-released catalogs are chronically underrepresented there and no competing platform contributes metadata outward instead of only consuming it. Would touch the `/api/metadata` module (a new submit path) and the release publish flow (trigger plus MBID storage). Needs research first: MusicBrainz's submission API may require a registered editor account per instance, and has rate limits and a moderation queue.

### Open threads

Known, verified, not yet resolved. Unlike ideas, these are things already true about the system.

- **Routes and services bypass the manager layer.** `core/managers/` exists, yet `routes/auth/zen.ts` and `modules/auth/auth.service.ts` call `db.prepare` directly for columns like `zen_pub` and `subsonic_token`. Not a correctness or security problem — those statements are parameterised with literal column names — but the layer only holds if it is used.

## Maintaining this repo

When a new TuneCamp component appears (companion app, deployable service):

1. Add a row to **What exists**.
2. Update the architecture diagram if component-to-component communication changed.
3. Record the deployed URL and the actual repo — and watch for upstream-vs-fork drift.

Two standing rules:

- **Do not vendor documentation here.** This repo used to carry a `docs/` snapshot of core's documentation. It drifted — copies fell behind their originals while still reading as authoritative — so it was removed. Link to the source repo instead; a link cannot go stale in the way a copy can.
- **Record decisions, not just inventory.** The Components table says what exists; **Facts worth knowing** says what was decided and why. The second is what stops a plausible idea from being built against a constraint nobody remembered.
