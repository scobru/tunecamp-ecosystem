# Zen SEA Unified Identity & Instance Passports

TuneCamp uses a **self-sovereign, decentralized identity model** powered by [Zen SEA](https://github.com/scobru/zen) (`@akaoio/zen`) and the P2P relay network (`wss://delay.scobrudot.dev/zen`).

This architecture allows users to unify their profiles across independent TuneCamp instances without relying on a centralized Single Sign-On (SSO) or shared database.

---

## 🏛️ Architecture Overview

```
                               ┌───────────────────────────┐
                               │     tunecamp.org          │
                               │  (Zen SEA Global Portal)  │
                               └─────────────┬─────────────┘
                                             │  WSS (Zen Graph)
                               ┌─────────────▼─────────────┐
                               │   wss://delay.scobrudot.dev│
                               │     Zen P2P Relay         │
                               └─────────────┬─────────────┘
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       │                                           │
          ┌────────────▼────────────┐                 ┌────────────▼────────────┐
          │   TuneCamp Instance A   │                 │   TuneCamp Instance B   │
          │ (sudorecords.scobru...) │                 │ (tunecamp.subterra...)  │
          └─────────────────────────┘                 └─────────────────────────┘
```

---

## 🔑 Endpoints

### 1. Generate Zen Challenge
- **Endpoint**: `GET /api/auth/zen/challenge`
- **Auth Required**: Yes (`requireUser`)
- **Response**:
```json
{
  "success": true,
  "challenge": {
    "instanceDomain": "sudorecords.scobrudot.dev",
    "username": "scobru",
    "nonce": "a4f891b2c3d4e5f67890123456789abc",
    "timestamp": 1721926658000
  }
}
```

### 2. Verify Challenge & Issue Passport Badge
- **Endpoint**: `POST /api/auth/zen/link`
- **Auth Required**: Yes (`requireUser`)
- **Body**:
```json
{
  "zenPubKey": "QmZenPubKey...",
  "challenge": { ... },
  "seaSignature": "SEA.sign_signature_data"
}
```
- **Response**:
```json
{
  "success": true,
  "passport": {
    "instanceDomain": "sudorecords.scobrudot.dev",
    "localUsername": "scobru",
    "zenPubKey": "QmZenPubKey...",
    "issuedAt": 1721926658000,
    "passportSignature": "HMAC_SHA256_SIGNATURE",
    "publicDataEndpoint": "https://sudorecords.scobrudot.dev/api/auth/zen/user/scobru/public"
  }
}
```

### 3. Public User Profile & Activity Export
- **Endpoint**: `GET /api/auth/zen/user/:username/public`
- **Auth Required**: No (Public Federation CORS enabled)
- **Response**: Returns public profile info, published albums, public playlists, and starred/liked items for cross-instance aggregation on `tunecamp.org`.
```json
{
  "success": true,
  "publicProfile": {
    "username": "scobru",
    "artistName": "Sudo Records",
    "bio": "Electronic music producer",
    "imageUrl": "https://sudorecords.scobrudot.dev/uploads/avatar.jpg",
    "joinedAt": "2026-01-01T00:00:00.000Z"
  },
  "publicReleases": [
    {
      "id": 1,
      "title": "Quantum Dub",
      "cover_url": "https://sudorecords.scobrudot.dev/uploads/cover.jpg",
      "release_date": "2026-07-01",
      "type": "album"
    }
  ],
  "publicPlaylists": [
    {
      "id": 1,
      "name": "Favorite Ambient Tracks",
      "cover_url": null,
      "created_at": "2026-07-15"
    }
  ],
  "publicLikes": [
    {
      "type": "album",
      "id": "10",
      "album_title": "Deep Space Echoes",
      "album_cover": "https://...",
      "created_at": "2026-07-20"
    }
  ]
}
```

---

## 🎨 Unified Identity Features (`tunecamp.org`)

1. **Multi-Artist Binding Per Instance**:
   A single Zen key (`~pubKey`) can bind to multiple local handles on the same instance (e.g. `@scobru` and `@artist_project` on `sudorecords.scobrudot.dev`). Accounts are keyed by `(instanceDomain, localUsername)`.

2. **Unified Activity Grids**:
   - **Releases**: Published albums and tracks fetched from instance SQLite `albums` table.
   - **Favorites**: Starred releases and tracks fetched from instance `starred_items` table.
   - **Playlists**: Public playlists created by linked handles.

3. **Direct Federation Links**:
   Every aggregated item features direct links to open the original item on the host TuneCamp node.

4. **Instance Unlinking**:
   Linked accounts can be unlinked dynamically with instant LocalStorage and state updates.
