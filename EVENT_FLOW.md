# din.fm Protocol Events

This file describes the **event protocol** used by din.fm: kinds, tags, payload shapes, and how events relate to each other.

## Kinds

- `40` - Room definition (NIP-28 channel metadata)
- `42` - Chat message (room stream + thread replies)
- `30078` - Avatar metadata (custom)
- `14578` - Room activity stats beacon (custom)
- `14579` - Room listing/alias beacon (custom)

## App Marker Conventions

din.fm uses app marker values for discoverability:

- marker tag: `["t", "gigachat-room-v1"]`
- marker field in JSON payloads: `"appKey": "gigachat_room_v1"`

These markers are used by custom events and listed rooms to distinguish din.fm traffic.

## Event Schemas

### `kind:40` Room Definition

Represents room metadata.

Content JSON:
- `name` (string)
- `about` (string, optional)
- `unlisted` (boolean)
- `appKey` (string, present for listed rooms)

Tags:
- `["client", "gigachat-web"]`
- `["t", "gigachat-room-v1"]` for listed rooms

Canonical room id is the event id of this `kind:40`.

---

### `kind:42` Chat Message

Represents chat content in a room.

Content:
- plain text message body

Tags:
- channel tag: `["e", "<roomId>", "", "channel"]`
- optional reply tag for threads: `["e", "<parentMessageId>", "", "reply"]`

Semantics:
- top-level message: only channel tag
- threaded reply: channel tag + reply tag

---

### `kind:30078` Avatar Metadata (custom)

Represents avatar configuration for a pubkey.

Content JSON:
- `e` (eyes index)
- `m` (mouth index)
- `hex` (face color hex)

Tags:
- `["client", "gigachat-web"]`
- `["t", "gigachat-room-v1"]`

Semantics:
- latest event by `created_at` for a pubkey is authoritative.
- message events do not embed avatar payloads.

---

### `kind:14578` Room Stats Beacon (custom)

Represents approximate room throughput snapshots.

Content JSON:
- `msgCount` (number in recent window)
- `windowSec` (window size in seconds)
- `msgsPerHour` (derived throughput)
- `lastMessageId` (message id used for rollover heuristic)
- `ts` (emit timestamp)
- `app` (`gigachat_room_v1`)

Tags:
- `["e", "<roomId>", "", "channel"]`
- `["t", "gigachat-room-v1"]`
- `["client", "gigachat-web"]`

Semantics:
- approximate, not canonical truth.
- last-write-wins by `created_at` for display.

---

### `kind:14579` Room Listing / Alias Beacon (custom)

Represents room discoverability + name-to-id mapping.

Content JSON:
- `roomId` (hex id of canonical room)
- `name` (display name)
- `alias` (shareable room alias/name)
- `appKey` (`gigachat_room_v1`)

Tags:
- `["e", "<roomId>", "", "channel"]`
- `["t", "gigachat-room-v1"]`
- `["client", "gigachat-web"]`

Semantics:
- used to discover rooms when `kind:40` is missing from local relay view.
- used to resolve `?room=<alias>` URLs to canonical room id.
- alias mapping is treated as latest-write-wins by timestamp in local index.

## URL Resolution Semantics

`room` query parameter supports:

- canonical id: `?room=<64-hex>`
- alias/name: `?room=<human-name-or-slug>`

Resolution order:
1. If `room` is 64-hex, use directly.
2. Else normalize alias and resolve via alias index built from `kind:40` + `kind:14579`.

If unresolved initially, clients can resolve later as more events arrive.

## Custom Event Emission + Fallback

Custom kinds in din.fm are **enhancers**. The base chat protocol remains `kind:40` + `kind:42`.

### When custom kinds are emitted

- `kind:30078` (avatar metadata)
  - emitted on app init, identity regeneration, and avatar save.
  - intent: share current avatar config for better cross-client rendering.

- `kind:14578` (room stats beacon)
  - emitted after successful `kind:42` publish.
  - not emitted continuously; gated by staleness/rollover heuristics.
  - intent: provide approximate room throughput (`messages per hour`).

- `kind:14579` (room listing/alias beacon)
  - emitted after successful `kind:42` publish when room is not already known/listed locally.
  - intent: improve discoverability and alias (`room name`) resolution.

### Why they are optional

If custom kinds are missing, delayed, or rejected by relays:

- Core chat still works via `kind:40` + `kind:42`.
- Rooms can still be joined by canonical room id (hex).
- Threading still works via `kind:42` tags.

### Fallback behavior

- Avatar fallback:
  - if no `30078` is available, clients use deterministic/default avatar rendering.

- Stats fallback:
  - if no fresh `14578` exists, clients operate without throughput hints (or show stale hints).

- Listing/alias fallback:
  - if no `14579` alias data exists, clients can still use canonical `?room=<hex>` links.
  - `kind:40` discovery remains valid independent of listing beacons.

## Ordering and Consistency Model

No central authority is assumed.

- Room identity: canonical `kind:40` event id
- Avatars: latest `kind:30078` per pubkey
- Stats: latest `kind:14578` per room
- Alias/listing: latest `kind:14579` per normalized alias

This is an eventually consistent model across relays.  
Conflicts are expected and resolved client-side with timestamp precedence.
