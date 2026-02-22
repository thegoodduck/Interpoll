# InterPoll Protocol Specification

> **Protocol codename:** **TurkeyLuck**  
> **Version:** 1.0  
> **Date:** 2026-02-21  
> **Authors:** @thegoodduck & @theendless11

> **Note** This spec describes the GunDB data schema and WebSocket relay protocol used by InterPoll, so that anyone can build a compatible client or server. **Sorry for not publishing prod backend, we are afraid of hackers :-(**
Hope you appreciate it, took a little while to write this(i am @thegoodduck for the record) this is my first time writing specs i read bunch of manuals and other specs before hand. ChatGPT cleaned up so all the nice linkbacks and all that stuff is it, i wrote it in plaintext it converted it to MD.
---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [GunDB Data Schema](#3-gundb-data-schema)
4. [WebSocket Relay Protocol](#4-websocket-relay-protocol)
5. [Blockchain (Local Audit Chain)](#5-blockchain-local-audit-chain)
6. [Nostr-Style Signed Events](#6-nostr-style-signed-events)
7. [Cryptography](#7-cryptography)
8. [HTTP API Endpoints](#8-http-api-endpoints)
9. [Server Discovery & Mesh](#9-server-discovery--mesh)
10. [Reference: Default Endpoints](#10-reference-default-endpoints)

---

## 1. Overview

InterPoll is a decentralised polling and community platform. Data is replicated through two complementary channels:

| Channel | Purpose |
|---------|---------|
| **GunDB** | Persistent, eventually-consistent graph database for communities, polls, posts, comments, and images. |
| **WebSocket relay** | Real-time fan-out of chain blocks, sync requests, peer discovery, and server-list gossip. |

Clients also maintain a **local append-only blockchain** (IndexedDB) for tamper-evident audit logging of votes and actions, signed with Schnorr signatures over secp256k1.

---

## 2. Architecture

```
┌──────────┐  GunDB (ws)   ┌──────────────┐
│  Client   │◄─────────────►│  Gun Relay   │
│  (Browser)│               │  Server      │
│           │  WSS          ├──────────────┤
│           │◄─────────────►│  WS Relay    │
│           │               │  Server      │
└──────────┘               └──────────────┘
      │  BroadcastChannel
      ▼
┌──────────┐
│ Other Tab│
└──────────┘
```

- **Gun Relay Server** — standard Gun.js relay (`gun` npm package) with `radisk: true`.
- **WS Relay Server** — lightweight WebSocket hub that registers peers, relays broadcasts, caches recent messages, and exposes HTTP endpoints for vote authorization and OAuth.
- **BroadcastChannel** — same-origin tab-to-tab sync (mirrors the WSS message types).

---

## 3. GunDB Data Schema

GunDB is used as the primary replicated data store. All data lives under top-level graph nodes. The Gun peer URL is of the form `https://<host>/gun` (upgrades to WebSocket internally).

### 3.1 Gun Client Configuration

```js
Gun({
  peers: ['https://interpoll2.onrender.com/gun'],
  localStorage: true,   // use browser localStorage as cache
  radisk: false,         // client-side disk persistence off
  axe: false             // disable AXE protocol logging
})
```

### 3.2 Communities

**Path:** `gun.get('communities').get('<communityId>')`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Unique ID, format: `c-<slugified-name>` |
| `name` | `string` | Raw community name |
| `displayName` | `string` | Human-readable display name |
| `description` | `string` | Community description |
| `creatorId` | `string` | Device fingerprint of creator |
| `createdAt` | `number` | Unix timestamp in milliseconds |
| `memberCount` | `number` | Current member count |
| `postCount` | `number` | Current post count |

**Rules** are stored as a numbered map:

**Path:** `gun.get('communities').get('<communityId>').get('rules')`

```json
{ "0": "Be respectful", "1": "No spam" }
```

### 3.3 Polls

Polls are stored in **two locations** (dual-write for fast community-scoped lookups):

1. **Global:** `gun.get('polls').get('<pollId>')`
2. **Community-scoped:** `gun.get('communities').get('<communityId>').get('polls').get('<pollId>')`

**Poll ID format:** `poll-<timestampMs>-<random9chars>`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Poll ID |
| `communityId` | `string` | Parent community ID |
| `authorId` | `string` | Device fingerprint of author |
| `authorName` | `string` | Display name of author |
| `question` | `string` | Poll question text |
| `description` | `string` | Optional description |
| `createdAt` | `number` | Unix timestamp (ms) |
| `expiresAt` | `number` | Expiry timestamp (ms) |
| `allowMultipleChoices` | `boolean` | Allow selecting multiple options |
| `showResultsBeforeVoting` | `boolean` | Show results before casting a vote |
| `requireLogin` | `boolean` | Require OAuth login |
| `isPrivate` | `boolean` | Require invite code |
| `totalVotes` | `number` | Running vote total |
| `isExpired` | `boolean` | Whether poll has expired |

#### 3.3.1 Poll Options

**Path:** `…get('<pollId>').get('options')`

Stored as a numbered map (keys are string indices `"0"`, `"1"`, …):

```json
{
  "0": { "id": "poll-xxx-option-0", "text": "Option A", "votes": 0 },
  "1": { "id": "poll-xxx-option-1", "text": "Option B", "votes": 0 }
}
```

**Option ID format:** `<pollId>-option-<index>`

#### 3.3.2 Voting

To cast a vote, increment the `votes` field on the chosen option(s) and increment `totalVotes` on the poll:

```
poll.get('options').get('<index>').get('votes')  ← current + 1
poll.get('totalVotes')                           ← current + N
```

Both the global and community-scoped copies must be updated.

#### 3.3.3 Invite Codes (Private Polls)

**Path:** `…get('<pollId>').get('inviteCodes')` — legacy numbered map  
**Path:** `…get('<pollId>').get('inviteCodesByCode')` — preferred keyed-by-code map

```json
// inviteCodesByCode
{ "A1B2C3D4": { "used": false }, "E5F6G7H8": { "used": true } }
```

Codes are 8-character uppercase alphanumeric strings. To consume: set `used: true`.

### 3.4 Posts

Dual-written like polls:

1. **Global:** `gun.get('posts').get('<postId>')`
2. **Community-scoped:** `gun.get('communities').get('<communityId>').get('posts').get('<postId>')`

**Post ID format:** `post-<timestampMs>-<random9chars>`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Post ID |
| `communityId` | `string` | Parent community ID |
| `authorId` | `string` | Device fingerprint |
| `authorName` | `string` | Display name |
| `title` | `string` | Post title |
| `content` | `string` | Post body text |
| `imageIPFS` | `string` | Optional IPFS CID for full image |
| `imageThumbnail` | `string` | Optional base64 thumbnail |
| `createdAt` | `number` | Unix timestamp (ms) |
| `upvotes` | `number` | Upvote count |
| `downvotes` | `number` | Downvote count |
| `score` | `number` | `upvotes - downvotes` |
| `commentCount` | `number` | Number of comments |

### 3.5 Comments

**Path:** `gun.get('comments').get('<commentId>')`

Each field is written individually (Gun.js field-per-put pattern):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Comment ID, format: `comment_<timestampMs>_<random9chars>` |
| `postId` | `string` | Parent post ID |
| `communityId` | `string` | Community ID |
| `authorId` | `string` | Device fingerprint |
| `authorName` | `string` | Display name |
| `content` | `string` | Comment text |
| `parentId` | `string?` | Parent comment ID (for threading) |
| `createdAt` | `number` | Unix timestamp (ms) |
| `upvotes` | `number` | |
| `downvotes` | `number` | |
| `score` | `number` | |
| `edited` | `boolean` | |
| `editedAt` | `number?` | |

**Comment index on post:**  
`gun.get('posts').get('<postId>').get('comments').set({ commentId, createdAt })`

### 3.6 Images

**Path:** `gun.get('images').get('<imageId>')`

| Field | Type | Description |
|-------|------|-------------|
| `data` | `string` | Base64-encoded image data |
| `thumbnail` | `string` | Base64-encoded thumbnail |
| `size` | `number` | Original file size in bytes |
| `uploadedAt` | `number` | Unix timestamp (ms) |

**Image ID format:** `img-<timestampMs>-<random9chars>`

---

## 4. WebSocket Relay Protocol

The WebSocket relay is a **JSON-over-WebSocket** message hub. All messages are JSON objects with a `type` field.

**Default URL:** `wss://<host>` (port 8080 in development)

### 4.1 Connection Lifecycle

```
Client                          Relay Server
  │                                  │
  │──── WebSocket connect ──────────►│
  │◄─── { type:"welcome" } ─────────│
  │──── { type:"register", peerId }─►│
  │◄─── { type:"peer-list", peers }──│  (broadcast to all)
  │──── { type:"join-room", roomId }►│
  │                                  │
  │  ... message exchange ...        │
  │                                  │
  │◄─── { type:"pong" } ────────────│  (in response to ping)
  │                                  │
  │     [connection closes]          │
  │◄─── { type:"peer-left", peerId }│  (broadcast to others)
```

### 4.2 Client → Server Messages

#### `register`
Register this client with the relay.

```json
{ "type": "register", "peerId": "<random-id>" }
```

`peerId` — random string generated via `Math.random().toString(36).substring(7)`.

#### `join-room`
Join a named room (currently only `"default"` is used).

```json
{ "type": "join-room", "roomId": "default" }
```

#### `ping`
Keep-alive heartbeat. Client sends every **25 seconds**.

```json
{ "type": "ping" }
```

#### `broadcast`
Relay a message to all other connected peers. The payload is wrapped:

```json
{
  "type": "broadcast",
  "data": {
    "type": "<message-type>",
    "data": { ... },
    "timestamp": 1708500000000
  }
}
```

The relay unwraps and sends `data` to all peers except the sender.

#### `direct`
Send to a specific peer (not commonly used):

```json
{ "type": "direct", "targetPeer": "<peerId>", "data": { ... } }
```

### 4.3 Server → Client Messages

#### `welcome`
Sent immediately on connection.

```json
{ "type": "welcome", "message": "Connected to P2P relay", "timestamp": 1708500000000 }
```

#### `peer-list`
Broadcast to all clients whenever a peer registers.

```json
{ "type": "peer-list", "peers": ["abc123", "def456", "ghi789"] }
```

#### `peer-left`
Broadcast when a peer disconnects.

```json
{ "type": "peer-left", "peerId": "abc123" }
```

#### `pong`
Response to client `ping`.

```json
{ "type": "pong", "timestamp": 1708500000000 }
```

### 4.4 Application-Level Broadcast Messages

These are sent via the `broadcast` wrapper (§4.2) or directly as top-level types. The relay forwards them to all other peers. The relay also **caches** content-bearing messages (`new-poll`, `new-block`, `sync-response`, `new-event`) and replays them to newly connecting clients.

#### `new-block`
A new chain block was created locally.

```json
{
  "type": "new-block",
  "data": {
    "index": 5,
    "timestamp": 1708500000000,
    "previousHash": "abcdef...",
    "voteHash": "123456...",
    "signature": "schnorr-sig-hex...",
    "currentHash": "789abc...",
    "nonce": 0,
    "pubkey": "x-only-pubkey-hex...",
    "actionType": "vote",
    "actionLabel": "Vote on poll-xxx"
  }
}
```

#### `request-sync`
Ask peers to send their chain blocks. Supports incremental sync via `lastIndex`.

```json
{
  "type": "request-sync",
  "data": {
    "peerId": "abc123",
    "lastIndex": 4
  }
}
```

- `lastIndex: -1` — request all blocks (full sync).
- `lastIndex: N` — only send blocks with `index > N`.

#### `sync-response`
Response containing chain blocks.

```json
{
  "type": "sync-response",
  "data": {
    "blocks": [ /* array of ChainBlock objects */ ],
    "peerId": "def456"
  }
}
```

#### `new-event`
A Nostr-style signed event (see §6).

```json
{
  "type": "new-event",
  "data": {
    "id": "sha256-hex-64chars",
    "pubkey": "x-only-pubkey-64chars",
    "created_at": 1708500,
    "kind": 101,
    "tags": [["poll_id", "poll-xxx"]],
    "content": "{\"choice\":\"Option A\",\"deviceId\":\"fp123\"}",
    "sig": "schnorr-sig-128chars"
  }
}
```

#### `new-poll`
Legacy/shortcut: a new poll was created (also synced via GunDB).

```json
{ "type": "new-poll", "data": { /* Poll object */ } }
```

#### `peer-addresses`
Peer shares its relay addresses for mesh discovery.

```json
{
  "type": "peer-addresses",
  "data": {
    "peerId": "abc123",
    "relayUrl": "wss://interpoll.onrender.com",
    "gunPeers": ["https://interpoll2.onrender.com/gun"],
    "joinedAt": 1708500000000
  }
}
```

#### `server-list`
Peer shares its known server list for federation.

```json
{
  "type": "server-list",
  "data": {
    "peerId": "abc123",
    "servers": [
      {
        "websocket": "wss://interpoll.onrender.com",
        "gun": "https://interpoll2.onrender.com/gun",
        "api": "https://interpoll.onrender.com",
        "addedBy": "abc123",
        "firstSeen": 1708500000000
      }
    ]
  }
}
```

### 4.5 Reconnection Strategy

Clients use **exponential backoff** on disconnect:

| Attempt | Delay |
|---------|-------|
| 1 | 1 s |
| 2 | 2 s |
| 3 | 4 s |
| 4 | 8 s |
| … | … |
| max | 30 s |

Formula: `min(1000 * 2^attempt, 30000)` ms. Attempts never stop (`maxReconnectAttempts = Infinity`).

### 4.6 Message Cache (Server-Side)

The relay server maintains an in-memory message cache (persisted to `message-cache.json` every 30 s). Cacheable message types: `new-poll`, `new-block`, `sync-response`, `new-event`. Maximum **500** cached messages. On peer registration, all cached messages are replayed to the new client.

---

## 5. Blockchain (Local Audit Chain)

Each client maintains a local append-only blockchain in **IndexedDB** (database name: `interpoll-db`, version `1`).

### 5.1 Block Structure (`ChainBlock`)

| Field | Type | Description |
|-------|------|-------------|
| `index` | `number` | Sequential block index (0 = genesis) |
| `timestamp` | `number` | Unix timestamp (ms) |
| `previousHash` | `string` | SHA-256 hex of previous block (genesis: `"0"×64`) |
| `voteHash` | `string` | SHA-256 of the action data (genesis: `"0"×64`) |
| `signature` | `string` | Schnorr signature (hex, 128 chars) |
| `currentHash` | `string` | SHA-256 hex of this block |
| `nonce` | `number` | Always `0` (reserved) |
| `pubkey` | `string?` | Signer's x-only public key (hex, 64 chars) |
| `actionType` | `string?` | `"vote"` \| `"community-create"` \| `"post-create"` |
| `actionLabel` | `string?` | Human-readable label |
| `eventId` | `string?` | Reference to NostrEvent that produced this block |

### 5.2 Block Hashing

The `currentHash` is computed as:

```
SHA-256(JSON.stringify({
  index,
  timestamp,
  previousHash,
  voteHash,
  signature,
  nonce,
  pubkey?,       // included only if present
  actionType?,   // included only if present
  actionLabel?   // included only if present
}))
```

JSON keys are serialized in the order listed above.

### 5.3 Block Signing

The signature covers:

```
Schnorr.sign(
  SHA-256(JSON.stringify({
    index,
    voteHash,
    previousHash
  })),
  privateKey
)
```

### 5.4 Vote Hash

```
SHA-256(JSON.stringify(voteData, sortedKeys))
```

Where keys are sorted alphabetically via `Object.keys(vote).sort()`.

### 5.5 Validation Rules

1. `block.index === previousBlock.index + 1`
2. `block.previousHash === previousBlock.currentHash`
3. `block.currentHash === computedHash(block)`
4. If `block.pubkey` is present: Schnorr signature must verify against `{index, voteHash, previousHash}`
5. Genesis block: `index === 0`, `previousHash === "0"×64`

### 5.6 Sync Protocol

1. On connect, client sends `request-sync` with `lastIndex` (highest local block index, or `-1`).
2. Peers respond with `sync-response` containing blocks where `block.index > lastIndex`.
3. Received blocks are validated against the local chain before being accepted.
4. Conflicts (same index, different hash) are ignored — first-write wins locally.

### 5.7 Vote Object

| Field | Type | Description |
|-------|------|-------------|
| `pollId` | `string` | Target poll ID |
| `choice` | `string` | Selected option text |
| `timestamp` | `number` | Unix timestamp (ms) |
| `deviceId` | `string` | Device fingerprint hash |

### 5.8 Receipt

After voting, a 12-word BIP-39 mnemonic is generated as a voter receipt:

| Field | Type | Description |
|-------|------|-------------|
| `blockIndex` | `number` | Chain block index |
| `voteHash` | `string` | Hash of the vote data |
| `chainHeadHash` | `string` | Chain head hash at time of vote |
| `mnemonic` | `string` | 12-word BIP-39 mnemonic |
| `timestamp` | `number` | Unix timestamp (ms) |
| `pollId` | `string` | Poll ID |

---

## 6. Nostr-Style Signed Events

InterPoll uses a Nostr-compatible event format (NIP-01) for cryptographic proof of authorship. Events are broadcast alongside chain blocks.

### 6.1 Event Structure

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | SHA-256 of canonical serialization (hex, 64 chars) |
| `pubkey` | `string` | x-only public key (hex, 64 chars) |
| `created_at` | `number` | Unix timestamp in **seconds** |
| `kind` | `number` | Event kind (see below) |
| `tags` | `string[][]` | Array of tag arrays |
| `content` | `string` | JSON-encoded payload |
| `sig` | `string` | Schnorr signature of `id` (hex, 128 chars) |

### 6.2 Event Kinds

| Kind | Name | Description |
|------|------|-------------|
| `100` | `POLL_CREATION` | A new poll was created |
| `101` | `VOTE_CAST` | A vote was cast |
| `102` | `POLL_UPDATE` | A poll was updated |
| `103` | `POST_CREATION` | A new post was created |

### 6.3 Canonical Serialization (NIP-01)

```
JSON.stringify([0, pubkey, created_at, kind, tags, content])
```

### 6.4 Event ID

```
id = SHA-256(canonical_serialization)   // hex
```

### 6.5 Signature

```
sig = Schnorr.sign(hexToBytes(id), hexToBytes(privateKey))  // hex
```

### 6.6 Verification

1. Recompute `id` from `{pubkey, created_at, kind, tags, content}` — must match `event.id`.
2. Verify `schnorr.verify(sig, id, pubkey)` — must return `true`.

### 6.7 Event Tag Conventions

| Tag | Used in kinds | Description |
|-----|---------------|-------------|
| `["poll_id", "<id>"]` | 100, 101, 102 | Reference poll |
| `["community", "<id>"]` | 100, 103 | Reference community |
| `["option", "<optionId>"]` | 101 | Selected option |
| `["post_id", "<id>"]` | 103 | Reference post |

### 6.8 Event Content Payloads

**Kind 100 (POLL_CREATION):**
```json
{
  "question": "What is your favorite color?",
  "description": "",
  "options": ["Red", "Blue", "Green"],
  "durationDays": 7,
  "allowMultipleChoices": false,
  "showResultsBeforeVoting": true,
  "requireLogin": false,
  "isPrivate": false
}
```

**Kind 101 (VOTE_CAST):**
```json
{ "choice": "Red", "deviceId": "sha256-fingerprint-hex" }
```

**Kind 102 (POLL_UPDATE):**
```json
{ "totalVotes": 42 }
```

**Kind 103 (POST_CREATION):**
```json
{ "title": "Hello World", "content": "My first post", "imageIPFS": "" }
```

---

## 7. Cryptography

### 7.1 Key Pair

- **Curve:** secp256k1
- **Signature scheme:** Schnorr (x-only public keys, BIP-340)
- **Private key:** 32 bytes, hex-encoded (64 chars)
- **Public key:** 32 bytes x-only, hex-encoded (64 chars)
- **Storage:** IndexedDB (`metadata` object store, key `"nostr-keypair"`)

### 7.2 Hashing

All hashing uses **SHA-256** producing hex-encoded output (64 chars).

### 7.3 Mnemonics

BIP-39 12-word English mnemonics are used as human-readable vote receipts.

### 7.4 Device Fingerprint

A SHA-256 hash of:
```
navigator.userAgent | navigator.language | timezoneOffset | colorDepth | screenResolution | hardwareConcurrency
```

---

## 8. HTTP API Endpoints

The WebSocket relay server also serves HTTP endpoints on the same port.

### 8.1 `POST /api/vote-authorize`

Server-side double-vote protection. The server maintains an in-memory `Set` of `pollId:deviceId` pairs.

**Request:**
```json
{ "pollId": "poll-xxx", "deviceId": "fingerprint-hash" }
```

**Response:**
```json
{ "allowed": true }
// or
{ "allowed": false, "reason": "already voted" }
```

### 8.2 `POST /api/receipts`

Append an audit receipt to the server's append-only log file (`storage.txt`).

**Request:** Any JSON object.

**Response:**
```json
{ "ok": true }
```

### 8.3 `GET /api/me`

Return the currently authenticated user (cookie-based session).

**Response:**
```json
{
  "user": {
    "provider": "google",
    "sub": "1234567890",
    "email": "user@example.com",
    "name": "John Doe",
    "picture": "https://..."
  }
}
```

### 8.4 OAuth Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/google/start` | GET | Redirect to Google OAuth |
| `/auth/google/callback` | GET | Google OAuth callback |
| `/auth/microsoft/start` | GET | Redirect to Microsoft OAuth |
| `/auth/microsoft/callback` | GET | Microsoft OAuth callback |
| `/auth/logout` | POST | Clear session cookie |

### 8.5 Gun Relay Health Check

On the Gun relay server (separate port, default `8765`):

| Endpoint | Method | Response |
|----------|--------|----------|
| `/health` | GET | `{ "status": "ok", "uptime": <seconds>, "peers": <count>, "timestamp": <ms> }` |

---

## 9. Server Discovery & Mesh

### 9.1 Known Server Object

```typescript
interface KnownServer {
  websocket: string;   // WSS relay URL (unique key)
  gun: string;         // Gun relay URL
  api: string;         // HTTP API base URL
  addedBy: string;     // peerId that reported this server
  firstSeen: number;   // Unix timestamp (ms)
}
```

### 9.2 Discovery Flow

1. On connect, client broadcasts `peer-addresses` with its own relay URLs.
2. On connect, client broadcasts `server-list` with all known servers.
3. When a `server-list` message is received, the client merges unknown servers into its local list.
4. Known servers are persisted in `localStorage` under key `interpoll_known_servers`.

### 9.3 Relay Configuration Override

Clients store relay overrides in `localStorage` under key `interpoll_relay_config`:

```json
{
  "websocket": "wss://custom-server.example.com",
  "gun": "https://custom-gun.example.com/gun",
  "api": "https://custom-server.example.com"
}
```

Empty strings fall back to built-in defaults.

---

## 10. Reference: Default Endpoints

| Service | Default URL |
|---------|-------------|
| WebSocket relay | `wss://interpoll.onrender.com` |
| Gun relay | `https://interpoll2.onrender.com/gun` |
| HTTP API | `https://interpoll.onrender.com` |

### 10.1 Running Your Own Relay

**Gun Relay Server:**
```bash
npm install express gun cors
node gun-relay.js
# Listens on port 8765 by default (set PORT env var to override)
```

**WebSocket Relay Server:**
```bash
npm install ws
node relay-server.js
# Listens on port 8080 by default
```

Both servers are stateless enough to run anywhere. The Gun relay persists data to disk via `radisk`. The WS relay caches messages in `message-cache.json` and vote-authorization state in memory (lost on restart).

---

## Appendix A: IndexedDB Schema

**Database:** `interpoll-db` (version `1`)

| Object Store | Key Path | Indexes |
|-------------|----------|---------|
| `blocks` | `index` | `by-hash` → `currentHash` |
| `votes` | `timestamp` | `by-poll` → `pollId` |
| `receipts` | `mnemonic` | `by-block` → `blockIndex` |
| `polls` | `id` | — |
| `metadata` | (out-of-line) | — |

## Appendix B: BroadcastChannel

Same-origin tab sync uses `BroadcastChannel('interpoll-sync')`. Message format is identical to the WSS application-level messages (§4.4). Messages include a `peerId` field; tabs ignore messages from their own `peerId`.

## Appendix C: Complete Message Type Reference

| Message Type | Direction | Cacheable | Description |
|-------------|-----------|-----------|-------------|
| `register` | C→S | No | Register peer |
| `join-room` | C→S | No | Join a room |
| `ping` | C→S | No | Keep-alive |
| `broadcast` | C→S | — | Wrapper for relayed messages |
| `direct` | C→S | No | Direct peer message |
| `welcome` | S→C | No | Connection acknowledgment |
| `peer-list` | S→C | No | Active peer list |
| `peer-left` | S→C | No | Peer disconnected |
| `pong` | S→C | No | Ping response |
| `new-block` | P2P | **Yes** | New chain block |
| `new-poll` | P2P | **Yes** | New poll created |
| `new-event` | P2P | **Yes** | Signed Nostr event |
| `request-sync` | P2P | No | Request chain sync |
| `sync-response` | P2P | **Yes** | Chain sync payload |
| `peer-addresses` | P2P | No | Relay address sharing |
| `server-list` | P2P | No | Known server gossip |
