# The Office — Product Requirements Document

**Version:** 1.5  
**Date:** October 22, 2026  
**Status:** Ready for sprint planning  
**Target:** One 3-day sprint, shipping a release every day  
**Team:** ~40 AI-assisted contributors in 5 build squads, plus the squad leads council (SQ0)  
**Ship model:** `dev` is production. Contributors merge into their **squad branch**; the **squad lead** reviews and tests it, then opens a **PR from the squad branch into `dev`**; a **second reviewer from another team** reviews that PR; on approval it merges into `dev` and is live. The only switch is per org and read at runtime: `organisations.office_settings.enabled`. A broken slice is reverted through the same path.

---

## 0. Change Log

| Version | Change                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0     | Initial PRD. Assumed a multi-week plan, contributor-helper login, static furniture, and reuse of a legacy 2D prototype.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| 1.1     | Rewritten after product decisions: Zedu-native auth; Zoe proxied now and ported later; shared building + private offices; channel/DM spatial mapping; grid-snap furniture with admin and personal persistence; one 4-day sprint with daily releases; legacy prototype assumptions removed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 1.2     | Merge-to-`dev` delivery with revert-based rollback and daily dev→production promotion. Workstreams renamed to squads. Rulings: personal-office guest invites cut; Zoe is stateless with no rate limiting; office presence syncs to 2D presence on transitions; avatar sources are CC0 packs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 1.3     | Lean schema: rooms are derived, not stored; `office_layouts` + `office_room_configs`; avatar config on `profiles`; audit via `audit_logs`; no building/room registry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| 1.4     | Final-consistency pass: feature flags restored per repo policy; 1:1 DM ownership per the requester's `dm_channels` row and group-DM ownership by group join order; team size fixed at 15–20; API path, draft, contract, and channel-naming fixes; PR-review allowlist called out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 1.5     | The building becomes a **tower**. Member floors of 40 (early joiners up) with one private office each, plus a dedicated top floor holding every channel in the org. Floor assignment is stored per org (`org_user_managements.floor_index`) and re-balanced on join/leave. New `area` room kind for per-floor facilities and extras. Tower view is 3D (click a window to fly in) and doubles as the lift ride. Person search tints the matching floor and pulses the matching office. Global feature flags dropped: `office_settings.enabled` is the only switch and it is read at runtime. `dev` is production, so the merge path is squad branch → lead-reviewed PR → cross-team review → `dev`. Sprint is 3 days with 5 build squads of ~8; SQ0 is the five squad leads and their assistants, and it owns the frozen artefacts rather than features. Each day runs from midnight: PR reviews start at 18:00 and the demo is due at 21:00 in the daily unblocking. |

### Locked decisions

| Area             | Decision                                                                                                                                                                                                                                                                                                                                                           |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Authentication   | Zedu login and session only. No NextAuth or Google login from contributor-helper.                                                                                                                                                                                                                                                                                  |
| Zoe backend      | Proxy to contributor-helper for this sprint, then port Zoe into office-zedu-fe later.                                                                                                                                                                                                                                                                              |
| Spatial model    | One **tower** per organisation: member floors of 40 people, each member with a private office, plus one general floor holding every channel in the org.                                                                                                                                                                                                            |
| Floor assignment | Stored per org on `org_user_managements.floor_index`. Filled **top-down**: the earliest joiners sit on the top member floor. A floor holds exactly **40**; the top member floor may be partially filled until someone joins. Re-balanced when a member joins or leaves.                                                                                            |
| Channel mapping  | Every Zedu channel in the org is a room on the **general floor**, reachable from any member floor through the portal door.                                                                                                                                                                                                                                         |
| DM mapping       | Zedu DMs map to private offices on their owner's floor.                                                                                                                                                                                                                                                                                                            |
| Furniture        | Grid snap + zones. Admins furnish the building; users furnish their personal office. Persist in Zedu Postgres.                                                                                                                                                                                                                                                     |
| Rooms in v1      | Channel rooms, private offices, meeting rooms, focus pods, Zoe reception, lounge/social, and `area` rooms for per-floor facilities and extras (visitors area, open pods, game area).                                                                                                                                                                               |
| Office           | Walled room with a door, a desk and a nameplate, on two stacked tiers per floor so 40 offices fit a half-length corridor.                                                                                                                                                                                                                                          |
| Tower view       | 3D, in the same engine: the tower seen from outside with cutaway walls; click a window to fly into the room. The tower overlay is also the lift ride.                                                                                                                                                                                                              |
| Avatar           | Tint + preset body/outfit/accessory parts, saved to Zedu user state.                                                                                                                                                                                                                                                                                               |
| Delivery         | One 3-day sprint in 5 build squads plus SQ0, ~40 AI-assisted contributors. `dev` is production: contributors merge into their squad branch, the squad lead reviews and tests it and opens a PR into `dev`, and a reviewer from another team approves that PR. The only switch is per org (`office_settings.enabled`, read at runtime). A broken slice is reverted. |
| Guest access     | Cut. Personal offices are enterable by the owner and active DM participants only; no invite flow.                                                                                                                                                                                                                                                                  |

---

## 1. Executive Summary & Goals

### 1.1 Vision

The Office is a WebGL/Three.js spatial layer over Zedu. Contributors log in with their existing Zedu account, enter their organisation's building, walk between channel rooms and private offices, see who is online, and talk to Zoe at her reception desk. Zoe is no longer a separate flat chat page; she is an in-world assistant triggered by proximity and backed by the existing contributor-helper knowledge base.

The product does **not** replace Zedu chat, auth, org roles, or calls. It gives those primitives a physical place.

### 1.2 Product thesis

1. **Reuse Zedu primitives.** Auth, org membership, channels, DMs, messages, RBAC, presence, and Buzz calls already exist. The Office adds geometry, furniture, proximity, and a spatial presence layer.
2. **Spatial rooms are views over existing conversations.** A channel room renders messages from that Zedu channel. A private office renders the DM with its owner. There is no second chat system.
3. **Only new state is spatial.** Spatial room overrides, furniture layouts, avatar config, and ephemeral positions are the only new data.
4. **Ship reviewed.** `dev` is production, so a release is a reviewed PR: the squad lead tests the squad branch, opens the PR into `dev`, and a reviewer from another team approves it. A broken slice is fixed by reverting its merge, not patched live.

### 1.3 End-of-sprint acceptance criteria

The sprint is successful when a logged-in Zedu user can:

1. Click **Enter Office** from the Zedu client and load their organisation's tower.
2. See the tower from outside: their floors, the general floor of channels, and the offices that are lit because someone is in them.
3. Click an office and fly into it, and see the nameplate, avatar and status of its occupant.
4. Move an avatar with WASD, arrow keys, or click-to-move, with collision against walls and furniture.
5. Enter a channel room on the general floor from any member floor, and see its members' messages as spatial chat.
6. Enter a private office on the owner's floor; only the owner and DM participant can enter.
7. Take the lift between floors, watching the tower overlay move, and arrive on the new floor.
8. Search for a person and have the tower tint their floor and pulse their office.
9. See other online users' avatars, nameplates, and status icons in real time.
10. Walk to Zoe's desk, press **E**, and receive a streamed answer grounded in the Zoe knowledge base.
11. As an admin, enter build mode, place/move/rotate/delete furniture in the building, publish, and have all clients update.
12. As a user, place/move/rotate/delete furniture in their personal office, reload, and find it persisted.
13. Customize avatar tint and preset parts, reload, and find the avatar persisted.

### 1.4 Success metrics

| Metric                       | Target                                  |
| ---------------------------- | --------------------------------------- |
| Login → controllable avatar  | < 3s p75 on desktop, < 6s p75 on mobile |
| Frame rate                   | 60fps desktop, ≥ 30fps mobile           |
| Presence update latency      | < 500ms p95                             |
| Zoe first streamed token     | < 2s p75                                |
| Furniture save round-trip    | < 300ms p75                             |
| Layout publish propagation   | < 2s p95 to all connected clients       |
| Tower to interior transition | < 1s p75                                |
| Days with a reviewed release | 3/3 days                                |
| P0 bugs at demo              | 0                                       |
| 20-user room stability       | 10-minute session without disconnect    |

### 1.5 Non-goals

- VR/AR.
- Voice synthesis or lip-sync for Zoe.
- Full character creator with body sliders and a clothing catalog.
- Custom 3D modelling pipeline; use primitives and licensed low-poly assets.
- Cross-organisation buildings.
- Persistent voice channels inside rooms (Buzz calls are launched from rooms; audio stays in the existing Buzz/Agora flow).
- Desk assignment in open-plan areas: `area` rooms such as open pods are extras, not allocated desks.
- A second message store, a second call stack, or a second presence system.

---

## 2. Users, Stories & Workflows

### 2.1 Personas

| Persona                | Needs                                           | Success looks like                                                              |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------- |
| **New contributor**    | Onboarding, task discovery, "where do I start?" | Spawns at reception, Zoe greets, asks a question, gets a grounded answer.       |
| **Active contributor** | Quick answers, awareness of teammates           | Walks to a channel room, sees who is present, reads/sends messages, visits Zoe. |
| **Team lead**          | See activity, manage space                      | Sees occupancy, opens a meeting room, asks Zoe for a summary.                   |
| **Org admin**          | Control the building                            | Enters build mode, arranges furniture, publishes, sees changes live.            |

### 2.2 User stories

**US-1 — Zedu login**
As a Zedu user, I open The Office from my organisation and I am already authenticated, with my name, avatar, role, and org loaded from my existing session.

_Acceptance:_ no second login screen; missing/expired token redirects to the existing Zedu login with a return URL; org slug stays correct.

**US-2 — Avatar**
As a user, I can choose a tint and preset body/outfit/accessory parts, and my choice persists across devices.

_Acceptance:_ saved through the Zedu API; visible to other users; default is deterministic from user ID.

**US-3 — Movement**
As a user, I can move with WASD/arrows or click the floor, and I cannot walk through walls, closed doors, or furniture.

_Acceptance:_ smooth acceleration, rotation, collision, and a camera that follows without clipping.

**US-4 — Channel rooms**
As a channel member, I can walk into the room mapped to my channel and see/send messages there.

_Acceptance:_ messages use the existing channel API and Centrifugo stream; non-members see a locked door; history loads on entry.

**US-5 — Private offices**
As a user, I have a private office mapped to my DMs. I can furnish it; only I can edit it; my active DM counterparts can enter as read-only guests.

_Acceptance:_ access enforced server-side; unauthorized users cannot subscribe to the room channel or load its layout.

**US-6 — Presence**
As a user, I see other online users in the same room as avatars with nameplates and status.

_Acceptance:_ join/leave in < 2s; positions interpolate; idle/away/call/DND states are visible.

**US-7 — Zoe**
As a user, I walk to Zoe's desk, press E, and ask a question. Zoe answers using the existing knowledge base and knows which room I am in and who is nearby.

_Acceptance:_ proximity prompt; streaming SSE; grounded answers; clear "I don't know" behavior; no private data leakage.

**US-8 — Personal furniture**
As a user, I can place, move, rotate, and delete furniture in my office, and it persists.

_Acceptance:_ grid snap; overlap and bounds validation; autosave; reload restores; guests see updates.

**US-9 — Admin building furniture**
As an admin, I can edit shared rooms and the building shell in-world and publish a new layout.

_Acceptance:_ build mode gated by role; draft is private; publish validates, increments version, persists, and broadcasts; all clients reconcile.

### 2.3 Core workflows

#### Workflow A — First entry

1. User clicks the **Office** item in the Zedu rail.
2. `/[org]/office` loads inside the existing client layout and `AuthGuard`.
3. Client reads `token` and `orgId` from `localStorage` via `AuthGuard` and profile/org from `DataContext`; `orgSlug` comes from the route param.
4. Building bootstrap request returns the building, the floor directory, the rooms, the channel/DM mapping, the occupants and the published layouts.
5. The tower opens: the general floor at the top, the member floors below, the offices that are lit.
6. User clicks an office and flies in. If no avatar config exists, the avatar modal opens with presets.
7. Zoe proximity trigger greets the user with a short onboarding prompt.

#### Workflow B — Channel room

1. User takes the portal door from any member floor to the general floor, then walks to a channel door, or clicks the channel in the room directory.
2. Client requests room access and layout.
3. Server checks channel membership, returns the room shell and layout, and issues/authorizes Centrifugo subscription.
4. Client loads the room scene and recent messages.
5. Messages sent in the room use the existing channel message endpoint.
6. Leaving the room unsubscribes and returns the avatar to the corridor.

#### Workflow C — DM private office

1. User opens a DM in the normal Zedu UI, or clicks a colleague's office in the tower and selects **Message**.
2. Client resolves the requester's DM row to the counterpart's private office (`dm_channels.participant_id`); for a group DM it resolves to the earliest-added participant's office. Every other participant enters as a read-only guest.
3. The office owner is the counterpart on the requester's row, so the requester enters as a read-only guest. The owner enters their own office from the directory (or when a counterpart's DM maps to it).
4. Messages use the existing DM endpoints; the 3D view renders them as spatial bubbles.

#### Workflow D — Between floors

1. User walks to the lift on their floor, or picks a floor in the tower rail.
2. Entering the lift opens the tower overlay with the lift moving to the target floor.
3. The overlay closes and the user stands on the new floor in the 3D view; only that floor is built.

#### Workflow E — Zoe

1. Avatar enters a 2m radius of Zoe's desk.
2. A billboard prompt appears: **Press E to talk to Zoe**.
3. Pressing E opens the Zoe dialog and pauses movement input.
4. The client posts to `POST /api/office/zoe` with the Zedu bearer token.
5. The office-zedu-fe route validates the token with office-zedu-be, builds a trusted office context, and proxies to contributor-helper with a service token.
6. The response streams back as SSE and renders with the existing typewriter pattern.
7. Closing the dialog returns control to the canvas.

#### Workflow E — Furniture edit

1. Owner/admin toggles **Edit** (or presses B).
2. Catalog opens; grid overlay and room bounds appear.
3. Place/move/rotate/delete items; client validates locally.
4. Personal office: autosave after 500ms idle. Admin building: save draft; **Publish** validates, increments version, persists transactionally, and broadcasts.
5. Other clients receive the published layout and reconcile without a reload.

---

## 3. Spatial Model

### 3.1 Zedu → Office mapping

| Zedu concept      | Office construct                                                                                                          | Access                                                  | Chat source                       | Furniture editor                      |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- | --------------------------------- | ------------------------------------- |
| Organisation      | Building (the tower)                                                                                                      | Org members                                             | —                                 | Org admins                            |
| Org member        | A private office on their floor                                                                                           | Owner + DM participants; org members see the office lit | —                                 | Self                                  |
| Channel           | A room on the **general floor**                                                                                           | Channel members                                         | Existing channel messages         | Org admins (MVP)                      |
| DM (1:1 or group) | Private office owned by the 1:1 counterpart on the requester's `dm_channels` row, or the earliest-added group participant | Owner + DM participants; non-owner read-only            | Existing DM messages              | Office owner                          |
| User              | Avatar + personal office                                                                                                  | Owner + DM participants; org members see avatar         | —                                 | Self                                  |
| Org role          | Build-mode permission                                                                                                     | `manage:organization` (owner/administrator)             | —                                 | —                                     |
| Buzz call         | Meeting room action                                                                                                       | Room occupants                                          | Agora/Buzz                        | Org admins                            |
| Focus pod         | DND booth                                                                                                                 | One occupant                                            | —                                 | Org admins                            |
| Zoe               | Reception NPC                                                                                                             | All org members                                         | Contributor-helper knowledge base | Org admins (desk is system furniture) |
| Floor             | A storey of the tower, 40 people                                                                                          | Org members may see the floor                           | —                                 | Org admins                            |

**DM rule (explicit):** a DM's spatial home is a private office. For a 1:1 DM, the office owner is the **counterpart on the requester's `dm_channels` row** (`participant_id`). `dm_channels` is a per-user view, so A's row maps to B's office and B's row maps to A's office; each is a read-only guest in the other's office. For a group DM, the owner is the **earliest-added participant** — earliest `channel_participants.created_at` for that DM, ties broken by lowest user id — and every other participant enters as a read-only guest. This satisfies both "personal office per user" and "DMs map to private offices" without creating a second room per pair.

### 3.2 Building layout (re-planned)

The building is a **tower**. Every storey is a floor; every floor holds people, except the general floor, which holds the org's channels.

```
                    +------------------------------+
   general floor    |  #general  #design  #dev ...  |   every channel in the org
                    |  reception, lounge, meeting,  |   reachable from any floor
                    |  focus pods                   |   through the portal door
                    +------------------------------+
   member floor 1   |  40 private offices           |   earliest joiners
                    |  visitors area, open pods,    |   (2 stacked tiers)
                    |  game area, facilities        |
                    +------------------------------+
   member floor 2   |  40 private offices           |   next 40 joiners
                    +------------------------------+
        ...         |                               |
                    +------------------------------+
   member floor N   |  40 private offices           |   newest joiners
                    +------------------------------+
```

- **Floor capacity:** 40 people, one private office each. The top member floor may be partially filled until someone joins.
- **Fill order:** **top-down** — the earliest joiners sit on the top member floor. The general floor is always above the member floors.
- **Offices:** walled room, door, desk, nameplate. Two stacked tiers per floor, so 40 offices fit half a corridor length.
- **Per-floor extras:** facilities and extras differ per floor (visitors area, open pods, game area) and are modelled as `area` rooms with a subtype.
- **General floor:** every channel in the org is a room here, so a conversation is always one portal door away regardless of the floor you stand on.
- Grid: **1 metre per tile**. Corridors: 2 tiles wide. Doors occupy reserved tiles and cannot be blocked by furniture.
- Only the **current floor** is rendered in 3D. The tower view is the same engine seen from outside, with cutaway walls, so a member floor shows a row of distinguishable offices with the occupied ones lit.
- Every position carries a `roomId` and a `floorIndex`. The lift and the portal door are rooms.
- Deep link: `/[org]/office?room=<url-encoded roomId>`.

### 3.3 Room lifecycle

Rooms are **derived from existing Zedu entities**, not stored as a parallel registry:

| Room kind   | Derived from                                              | Default layout                         |
| ----------- | --------------------------------------------------------- | -------------------------------------- |
| `lobby`     | Fixed building key, on the general floor                  | Seeded                                 |
| `reception` | Fixed building key, on the general floor                  | Seeded; Zoe desk system item           |
| `lounge`    | Fixed building key, on the general floor                  | Seeded                                 |
| `meeting`   | Fixed keys `meeting:1`, `meeting:2`, on the general floor | Seeded                                 |
| `focus`     | Fixed keys `focus:1..4`, on the general floor             | Seeded                                 |
| `channel`   | `channels.id`, on the general floor                       | Template chosen by channel name hash   |
| `personal`  | Owner's `users.id`, on the owner's member floor           | Template by avatar/tint hash           |
| `area`      | Per-floor extras: `area:<floor>:<key>`                    | Seeded; subtype label in the room name |

Only **spatial overrides** (bounds, capacity, enabled) and **layouts** are persisted. A channel rename, archive, or membership change is reflected on the next bootstrap without syncing room rows. `roomId` is an opaque composite: `channel:<channelId>`, `personal:<ownerUserId>`, `area:<floor>:<key>`, `lobby`, `meeting:<n>`, `focus:<n>`, `lounge`, `reception`. A DM deep link resolves through the requester's `dmOfficeMap` to `personal:<ownerUserId>`.

**Floor assignment** is stored per organisation on `org_user_managements.floor_index` and re-balanced when a member joins or leaves: fill the top member floor to 40, then the next one down. The general floor is not part of the assignment; it is derived from the org's channels.

---

## 4. Architecture & Codebase Touchpoints

### 4.1 System diagram

```
Browser (office-zedu-fe)
  ├─ AuthGuard + DataContext ............ existing Zedu session
  ├─ /[org]/office ...................... new route
  │   ├─ OfficeCanvas (Three.js)
  │   ├─ TowerScene / BuildingScene / RoomScene
  │   ├─ AvatarLayer / PresenceManager / OfficeSearch
  │   ├─ FurnitureEditor / Catalog
  │   └─ ZoeDialog
  ├─ existing channel + DM APIs ......... GetRequest/PostRequest
  ├─ existing Centrifugo connections .... chat/status
  └─ POST /api/office/zoe ............... office-zedu-fe server route (new)
        │  validates Zedu bearer token
        │  adds service token + trusted identity
        ▼
contributor-helper /api/chat ............ temporary Zoe brain
        │  OpenAI-compatible stream (OpenRouter)
        └─ knowledge.ts + clean-transcripts corpus

office-zedu-be (Go/Gin)
  ├─ /api/v1/office/* ................... new router/controller/service
  ├─ /api/v1/users/me/avatar ............ avatar config
  ├─ existing channels/DMs/messages ..... unchanged
  ├─ existing Centrifugo publish/auth ... extended for office rooms
  └─ Postgres ........................... office + furniture tables
```

### 4.2 Authentication flow (Zedu only)

1. `AuthGuard` (`src/app/(client)/[org]/_components/auth/auth-guard.tsx`) reads `token`, `orgId` from `localStorage`, dispatches `TOKEN`/`ORG_ID`, and fetches `/profile` and `/organisations/:orgId`.
2. The office route is inside `(client)/[org]`, so it inherits the guard and `client-layout.tsx`.
3. All office REST calls go through `GetRequest`/`PostRequest`/`PutRequest`/`DeleteRequest` from `src/utils/new-request.ts`, which attach `Authorization: Bearer <token>`.
4. All office server routes (only Zoe needs one) read the same bearer token and validate it against office-zedu-be before doing anything privileged.
5. There is **no** NextAuth session, Google OAuth, or contributor-helper login in the office path.

### 4.3 Zoe proxy flow (temporary)

- `office-zedu-fe/src/app/api/office/zoe/route.ts`:
  - Requires `Authorization: Bearer <zedu-token>`.
  - Calls office-zedu-be `GET /api/v1/users/me` (or the existing `/profile` endpoint) to resolve `userId`, `orgId`, display name, and role.
  - Builds the office context from a **server-validated** room membership check, not from raw client claims.
  - Calls contributor-helper `POST /api/chat` with `Authorization: Bearer ${ZOE_SERVICE_TOKEN}` and `x-zedu-user-id`, `x-zedu-org-id`, `x-zedu-role`.
  - Streams SSE back to the browser.
- `contributor-helper/src/app/api/chat/route.ts`:
  - Extend `getAuthenticatedUser()` to accept a constant-time-validated service token when `ZOE_ALLOW_SERVICE_AUTH=true`.
  - Keep the NextAuth path for standalone contributor-helper use.
  - Never trust identity headers unless the service token is valid.
- Secrets: `ZOE_SERVICE_TOKEN` is server-only in both deployments. It is never shipped to the browser.
- Context caps only (transport hygiene, not rate limiting): max 30 conversation messages, max 10 nearby users, max 20 tasks in context. No rate limiting this sprint — an accepted risk.
- Persistence: service-token chats are stateless. contributor-helper stores nothing for office traffic, so the Google-sub vs `x-zedu-user-id` key mismatch never materialises; the client supplies the conversation window on every request.
- **Port plan:** move the prompt and `knowledge.ts` into `office-zedu-fe/src/lib/zoe/`, move chat persistence to office-zedu-be, and keep the `POST /api/office/zoe` request/response shared schema unchanged. UI does not change.

### 4.4 Frontend structure

```
src/app/(client)/[org]/office/
├── page.tsx                     # office entry; dynamic import of the canvas
├── layout.tsx                   # office shell, providers
├── _components/
│   ├── OfficeCanvas.tsx         # renderer, scene lifecycle, resize, quality tier
│   ├── engine/
│   │   ├── useOfficeEngine.ts   # loop, input, movement, collision, camera
│   │   ├── useRoomLoader.ts     # floor bootstrap + scene swap
│   │   ├── useTowerLoader.ts    # tower directory, cutaway geometry, fly-in
│   │   ├── floorGenerator.ts    # procedural floor: 2 tiers, 40 offices, lift, portal
│   │   ├── grid.ts              # grid math, AABB, pathfinding
│   │   └── quality.ts           # desktop/mobile tiers
│   ├── scenes/
│   │   ├── TowerScene.tsx       # cutaway tower, lit offices, floor rail, lift
│   │   ├── BuildingScene.tsx    # current floor: corridor, doors, offices, extras
│   │   ├── RoomScene.tsx        # channel/personal/meeting interior
│   │   └── ZoeDesk.tsx          # reception NPC + interaction sphere
│   ├── avatars/
│   │   ├── Avatar.tsx           # GLTF, tint, nameplate, status icon
│   │   ├── AvatarLayer.tsx      # local + remote avatars, interpolation
│   │   └── avatarPresets.ts     # body/outfit/accessory presets
│   ├── presence/
│   │   ├── PresenceManager.tsx  # Centrifugo subscription + heartbeat
│   │   ├── presenceMath.ts      # throttle, interpolation, dead reckoning
│   │   └── OfficeSearch.tsx     # people search: tints the floor, pulses the office
│   ├── chat/
│   │   ├── RoomChat.tsx         # existing channel/DM message hooks
│   │   └── SpeechBubbles.tsx    # sprite bubbles, distance fade
│   ├── furniture/
│   │   ├── FurnitureEditor.tsx  # edit mode, selection, transform
│   │   ├── CatalogPanel.tsx     # searchable catalog
│   │   ├── gridOverlay.ts       # grid + reserved tiles
│   │   └── layoutValidation.ts  # bounds, overlap, doors, limits
│   ├── zoe/
│   │   ├── ZoeDialog.tsx        # shadcn Dialog + streaming chat
│   │   ├── useZoeChat.ts        # SSE client + typewriter
│   │   └── zoePrompt.tsx        # proximity billboard + E key
│   └── ui/
│       ├── OfficeHud.tsx        # room name, online count, controls
│       ├── Minimap.tsx          # rooms + avatars
│       ├── RoomDirectory.tsx    # channel/personal room list
│       └── LoadingScreen.tsx
```

The Zoe proxy is **not** nested under the client route. It is a top-level App Router handler:

```
src/app/api/office/zoe/route.ts   # server proxy only
```

### 4.5 Backend structure

```
pkg/router/office.go              # register /api/v1/office/*
pkg/controller/office/            # bind, validate, respond
services/office/                  # derive floors + rooms, access, layouts, publish, avatar
internal/models/office.go         # OfficeLayout, OfficeRoomConfig, floor DTOs
internal/models/migrations/       # additive migration + three nullable columns
tests/test_office/                # package test_office
```

No `office_buildings` or `office_rooms` tables. The building is the organisation; floors and rooms are derived from channels, DMs, and users. The only new stored state is the **floor assignment** on `org_user_managements.floor_index` (nullable int) plus the two layout tables and the two nullable JSONB columns.

`services/office` gains a floor module:

```
services/office/floors.go         # assign, re-balance, derive the floor directory
```

Floor assignment rules: a floor holds **40** people; members are assigned **top-down**, the earliest joiners first; the top member floor may be partially filled; a join fills the top member floor, a leave pulls the lowest-floor member up to keep floors contiguous. The general floor is derived from the org's channels and is not part of the assignment.

Register `Office(r, ApiVersion, validator, db, logger)` in `pkg/router/router.go` next to the other feature routers. Use `middleware.Authorize(db.Postgresql)` and existing deactivated/org checks. The router registers unconditionally; the **per-org switch** is `organisations.office_settings.enabled`, read at runtime and checked at the start of every office handler except `GET /office/status`, which reports it. A disabled org gets `403 office_disabled` from every other office route.

### 4.6 Reuse map

| Need                 | Reuse                                                                     |
| -------------------- | ------------------------------------------------------------------------- |
| Auth/session         | `AuthGuard`, `DataContext`, `localStorage` token, `new-request.ts`        |
| Org/role/permissions | `DataContext.orgData`, `src/types/rbac.ts`, `PermissionBoundary`          |
| Channel messages     | Existing channel message endpoints and Centrifugo chat connection         |
| DM messages          | Existing DM endpoints and hooks (`useOrganisationDms`)                    |
| Calls                | `src/lib/agora/`, `src/hooks/buzz/`                                       |
| Presence transport   | Existing Centrifugo connection/token setup                                |
| UI                   | `src/components/ui/dialog.tsx`, `button.tsx`, `avatar.tsx`, `tooltip.tsx` |
| Toasts               | `src/components/toast/sonner.tsx`                                         |
| Zoe knowledge        | `contributor-helper/src/lib/knowledge.ts` and chat prompt                 |
| Server HTTP          | `src/utils/new-request.ts` (client), `fetch` with `ZEDU_API_URL` (server) |

### 4.7 Key files to create or extend

| File                                                                                     | Type   | Purpose                                                                           |
| ---------------------------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------- |
| `office-zedu-fe/src/app/(client)/[org]/office/page.tsx`                                  | New    | Office entry, dynamic canvas import                                               |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/OfficeCanvas.tsx`              | New    | Renderer and scene lifecycle                                                      |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/engine/useOfficeEngine.ts`     | New    | Movement, collision, input, camera                                                |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/engine/floorGenerator.ts`      | New    | Procedural floor: two tiers, 40 offices, corridor, lift, portal door              |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/scenes/TowerScene.tsx`         | New    | Tower view: cutaway floors, lit offices, click to fly in                          |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/scenes/BuildingScene.tsx`      | New    | Building shell and doors                                                          |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/scenes/RoomScene.tsx`          | New    | Room interiors                                                                    |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/avatars/Avatar.tsx`            | New    | Avatar rendering                                                                  |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/presence/PresenceManager.tsx`  | New    | Presence sync                                                                     |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/presence/OfficeSearch.tsx`     | New    | Person search: tints the floor, pulses the office                                 |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/chat/RoomChat.tsx`             | New    | Channel/DM chat in room                                                           |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/furniture/FurnitureEditor.tsx` | New    | Placement and edit mode                                                           |
| `office-zedu-fe/src/app/(client)/[org]/office/_components/zoe/ZoeDialog.tsx`             | New    | Zoe UI                                                                            |
| `office-zedu-fe/src/app/api/office/zoe/route.ts`                                         | New    | Zoe proxy                                                                         |
| `office-zedu-fe/src/types/office.ts`                                                     | New    | Shared frontend schema (frozen, Day 0)                                            |
| `office-zedu-fe/src/lib/office/`                                                         | New    | API client, catalog, mapping, validation                                          |
| `office-zedu-fe/src/lib/office/use-office-status.ts`                                     | New    | Reads `GET /office/status` once per org; drives the rail item and the route guard |
| `office-zedu-fe/docs/office/PRD.md`                                                      | New    | This document                                                                     |
| `office-zedu-fe/src/components/layout/sidebar/`                                          | Extend | **Office** rail item — the entry point to the live office                         |
| `contributor-helper/src/app/api/chat/route.ts`                                           | Extend | Service-token auth path                                                           |
| `office-zedu-be/pkg/router/office.go`                                                    | New    | Routes                                                                            |
| `office-zedu-be/pkg/controller/office/office.go`                                         | New    | Handlers                                                                          |
| `office-zedu-be/services/office/office.go`                                               | New    | Business logic                                                                    |
| `office-zedu-be/services/office/floors.go`                                               | New    | Floor assignment and re-balance                                                   |
| `office-zedu-be/internal/models/office.go`                                               | New    | Models and DTOs                                                                   |
| `office-zedu-be/internal/models/audit_trail.go`                                          | Extend | Add `ResourceOfficeLayout` and office layout actions                              |
| `office-zedu-be/pkg/router/router.go`                                                    | Extend | Register office router                                                            |

---

## 5. Feature Specifications

### 5.1 Authentication & session

| Requirement        | Spec                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Login              | Existing Zedu login only. The office is inside the authenticated client shell.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Token              | Bearer token from `localStorage.token`; `orgId` and `orgSlug` from storage/context.                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Expiry             | 401 triggers the existing `handleSessionExpired()` and returns to `/auth/login?redirect=...`.                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Org switch         | Switching org in Zedu reloads the office with the new org's building.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Server auth        | Office routes use `middleware.Authorize`; room access checks run after auth.                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Contributor-helper | Not a login surface. Only the server-to-server Zoe proxy uses it.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Feature gate       | `organisations.office_settings.enabled` is the only switch, and it is read at **runtime**. The client asks `GET /api/v1/office/status`; when it is false the Office rail item is hidden and `/[org]/office` redirects to the organisation landing `/[org]`, and the office API answers `403 office_disabled`. There is no env var and no build-time switch: enabling an org is a database change that takes effect on the next request. Owners and administrators flip the switch in the organisation settings, under **The Office**. |

### 5.2 Avatar & profile

- **Data source:** Zedu profile (`display_name`, `username`, `avatar_url`, `role`) + `profiles.avatar_config`.
- **Customization:** tint color; body preset (3); outfit preset (4); accessory (none, headphones, glasses, cap).
- **Defaults:** deterministic from `userId` hash so users never appear blank.
- **Persistence:** `GET/PUT /api/v1/users/me/avatar`; optimistic UI with rollback on failure.
- **Rendering:** low-poly rigged GLTF; idle/walk/run clips; nameplate sprite; status ring; LOD for far avatars.
- **Tint clamp:** clamp saturation/lightness so dark or washed-out choices stay visible against the floor palette.
- **2D app:** unchanged; avatar config is additive.

### 5.3 3D engine and movement

| Item             | Spec                                                                                                            |
| ---------------- | --------------------------------------------------------------------------------------------------------------- |
| Renderer         | `WebGLRenderer({ antialias: true, powerPreference: "high-performance" })`; shadows on desktop, off on low tier. |
| Camera           | Third-person follow inside a floor, FOV 60, smooth lerp; an exterior camera for the tower view.                 |
| Input            | WASD/arrows relative to camera; click/tap floor to pathfind; Shift to run.                                      |
| Movement         | Acceleration/deceleration; max walk 3 m/s, run 5 m/s; rotation via quaternion slerp.                            |
| Collision        | Grid AABB against walls, closed doors, and furniture footprints; sliding along walls.                           |
| Pathfinding      | A\* on the room grid; click target snapped to walkable tile; path cancels on manual input.                      |
| Room transition  | Door interaction or click → fade → load target room → spawn at its entry tile.                                  |
| Floor transition | The lift → the tower overlay plays the ride → the new floor is built and the user lands on it.                  |
| Tower view       | The same scene from outside with cutaway walls; no interiors are built for the floors you are not on.           |
| Quality tiers    | Desktop: shadows, 2× DPR cap. Mobile/low: no shadows, 1.5× DPR cap, fewer decorative items.                     |
| Pause            | Tab hidden pauses rendering; presence heartbeat continues at reduced rate.                                      |

**Performance budget**

| Metric           | Desktop              | Mobile |
| ---------------- | -------------------- | ------ |
| Draw calls       | < 100                | < 60   |
| Triangles        | < 60k                | < 30k  |
| Initial shell JS | < 2.5MB gzip         | same   |
| Room assets      | Lazy-loaded per room | same   |
| Avatars visible  | 30                   | 15     |
| Position updates | 10Hz                 | 5Hz    |

### 5.4 Building, floors, rooms, and access

- **Building bootstrap:** `GET /api/v1/office/bootstrap` returns the organisation's office settings, the **floor directory**, the rooms per floor, the channel/DM mapping, and published layouts for visible rooms. The building id is the organisation id; there is no separate building row.
- **Org switch:** `GET /api/v1/office/status` returns `{ enabled }` for the caller's org. It is the only switch: the rail item is rendered from it, the route redirects to the organisation landing `/[org]` when it is false, and every office handler answers `403 office_disabled`. The setting itself is edited in the organisation settings, under **The Office**: `GET /api/v1/office/settings` reads it and `PUT /api/v1/office/settings` writes it for owners and administrators. Enabling an org is a database change, not a deployment.
- **Floor directory:** the general floor (every channel in the org, plus reception, lounge, meeting rooms and focus pods) and one entry per member floor with its 40 offices and its `area` extras.
- **Room access rules:**

| Room kind   | Floor   | Who can enter                  | Who can subscribe          | Who can edit furniture |
| ----------- | ------- | ------------------------------ | -------------------------- | ---------------------- |
| `lobby`     | general | Org members                    | Org members                | Org admins             |
| `reception` | general | Org members                    | Org members                | Org admins             |
| `lounge`    | general | Org members                    | Org members                | Org admins             |
| `meeting`   | general | Org members; capacity 8        | Org members up to capacity | Org admins             |
| `focus`     | general | One occupant; capacity 1       | Occupant                   | Org admins             |
| `channel`   | general | Channel members                | Channel members            | Org admins (MVP)       |
| `area`      | member  | Org members                    | Org members                | Org admins             |
| `personal`  | owner's | Owner + active DM participants | Same                       | Owner                  |

- **Capacity:** room-level `capacity`; entering a full room is blocked with a toast.
- **Locked doors:** show channel name and "Join channel in Zedu to enter".
- **Portal door:** every member floor has a door to the general floor. It is the only way to reach a channel room, and it is a room in its own right so presence and chat follow it.
- **Lift:** present on every floor. Riding it is the only way to change floors, and the ride is played in the tower view (§5.6).
- **Meeting rooms:** an in-room **Start Buzz** action calls the existing Buzz flow with the room's occupants.
- **Focus pods:** entering sets status `dnd`; leaving restores previous status.
- **No desk assignment:** `area` rooms are extras, not allocated desks.

### 5.5 Presence and status

- **Transport:** Centrifugo. One presence channel per room, namespaced by organisation so two orgs can never see each other: `office:org:{orgId}:floor:{floorIndex}:room:{encodedRoomId}:presence`, where `encodedRoomId` is the URI-encoded `roomId` (`channel%3Ach_1`) so the colon inside the room id does not collide with the channel separators. The subscription token endpoint rejects a channel whose organisation does not match the caller's token.
- **Client → server:** position/rotation/status at 10Hz (5Hz mobile), throttled and validated.
- **Server validation:** room membership, bounds, max speed (anti-teleport), timestamp monotonicity.
- **Server → clients:** join, leave, update events; clients interpolate remote avatars.
- **Status values:** `online`, `idle` (>2 min no input), `away` (tab hidden), `call` (Buzz active), `dnd` (focus pod).
- **Per-office presence:** a member's presence is anchored to their office, so an occupied office is lit in the tower and shows a nameplate, avatar and status icon (🟢 Available / 🔴 Busy / 💤 Away) from the outside.
- **Activity string:** each member carries a live activity ("On a call with Musa Garba", "In the lift to F1"), shown on hover, in their office label and in the People tab.
- **2D presence sync:** office enter/leave writes through via the existing `POST /api/v1/profile/presence` endpoint, which broadcasts `user_presence_changed`; position updates never write through. `call`/`dnd` are shown in the office status ring and are mapped to the existing status text/icon endpoint where supported; otherwise they stay office-local. `idle`/`away` remain office-local.
- **Nameplates:** display name + status icon; distance-based scaling and culling.
- **Reconnect:** exponential backoff; resubscribe; resync full room presence snapshot.
- **Grace:** remove an avatar 30s after disconnect unless presence returns.

### 5.6 The tower view

The tower is the same 3D engine seen from outside. There is no second renderer.

| Interaction    | Behaviour                                                                                                                                                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Glancability   | Every office shows a floating nameplate, avatar and status icon. Occupied offices are lit; empty ones are dark. The general floor of channels is always visible at the top.                                                    |
| Hover          | Hovering an office highlights its outline, brings the avatar into focus and shows the person's role.                                                                                                                           |
| One-click find | Clicking an office flies the camera into the room and opens a DM window for that person. A second click moves your avatar to their door.                                                                                       |
| Search         | A search bar for **people**. On a match, the tower **tints the matching floor** and **pulses the matching office**; a result can be clicked to fly straight in.                                                                |
| Lift           | Entering the lift in the 3D view opens the tower overlay and plays the lift moving to the target floor, then drops the user back into the 3D view on that floor. The overlay is a mode of the same route, not a separate page. |
| Floor switch   | The rail of the tower view selects a floor; the 3D view rebuilds only that floor.                                                                                                                                              |

The tower view is also the demo surface: the standalone demo shows a 3-floor tower and its floor interior with no backend.

### 5.7 Spatial chat

- **No new message store.** Channel rooms use existing channel messages; private offices use existing DM messages.
- **Send:** existing `PostRequest`/message hooks; the 3D client supplies the channel/DM id from the room mapping.
- **Receive:** existing Centrifugo chat connection; the office filters events to the active room.
- **Bubbles:** show the latest message above the sender for 8s; fade by distance beyond 8m; mute in focus pods.
- **Proximity:** client-side visibility radius (default 10m) within the same room; the message is still persisted normally.
- **History:** existing channel/DM history endpoints; last 30 messages rendered in the room log.
- **Moderation:** inherits channel permissions, edits, deletes, and restrictions.
- **Safety:** render message text as sanitized Markdown/plain text; no raw HTML.

### 5.8 Zoe spatial assistant

| Item        | Spec                                                                                                                                                     |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Placement   | Reception desk, fixed system furniture, interaction radius 2m.                                                                                           |
| Prompt      | Billboard above desk when in range: “Press E to talk to Zoe”.                                                                                            |
| Dialog      | shadcn `Dialog`; header “Zoe — Zedu Contributor Assistant”; close returns to canvas.                                                                     |
| Streaming   | SSE from `/api/office/zoe`; typewriter reveal; stop button aborts the request.                                                                           |
| Inputs      | Text only in v1; quick chips: “My tasks”, “Team status”, “Deadlines”, “How do I…?”                                                                       |
| Context     | Room kind/name, nearby users (max 10), user role/timezone, user tasks if available. No private room content.                                             |
| Knowledge   | Existing Zoe system prompt + `knowledge.ts` summaries and transcript matching.                                                                           |
| Answers     | Grounded in Zedu docs + cohort call knowledge; say “I couldn't find that” rather than invent.                                                            |
| Proactive   | Stretch only: one greeting on first entry and one tip per session. No unsolicited spam.                                                                  |
| Persistence | Stateless: office chats are not persisted by contributor-helper; the client sends the 30-message window each request. No rate limiting in v1 (accepted). |
| Port later  | Same shared schema; move prompt/knowledge into office-zedu-fe and persistence to office-zedu-be.                                                         |

**Zoe context shape**

```json
{
  "location": "office",
  "buildingId": "org_...",
  "room": { "id": "channel:ch_...", "kind": "channel", "name": "#design" },
  "nearbyUsers": [
    { "displayName": "Ada", "status": "online" },
    { "displayName": "Tunde", "status": "dnd" }
  ],
  "user": { "role": "user", "timezone": "Africa/Lagos" },
  "tasks": []
}
```

### 5.9 Furniture system

**Placement model:** grid snap, 1m tiles, 90° rotations, room zones.

**Catalog (MVP):** desk, chair, meeting table, sofa, plant, whiteboard, screen, rug, lamp, bookshelf, reception counter, Zoe desk (system), focus booth (system). Each item declares footprint, allowed room kinds, collision, max per room, and variant.

```json
{
  "id": "desk_oak",
  "name": "Oak desk",
  "category": "desk",
  "footprint": { "w": 2, "d": 1 },
  "rotatable": true,
  "allowedRoomKinds": ["channel", "personal", "meeting"],
  "collision": true,
  "maxPerRoom": 8,
  "variants": ["oak", "white", "black"]
}
```

**Layout document**

```json
{
  "roomId": "channel:ch_1",
  "version": 7,
  "grid": { "width": 12, "height": 10, "tile": 1 },
  "items": [
    {
      "id": "f_01",
      "catalogId": "desk_oak",
      "x": 4,
      "y": 7,
      "rotation": 90,
      "variant": "oak"
    }
  ],
  "updatedAt": "2026-10-19T12:00:00Z"
}
```

**Validation rules**

- Item footprint must be inside room bounds.
- No overlap with another item's footprint.
- Reserved tiles (doors, spawn, Zoe interaction area) cannot be blocked.
- Item must be allowed in the room kind.
- Rotation ∈ {0, 90, 180, 270}.
- Per-room item cap: 100; per-catalog cap enforced.
- Publish requires `If-Match: <version>`; mismatch returns `409` with the current layout.
- Server re-validates everything; client validation is only for UX.

**Editing**

| Context               | Who                 | Save model                 | Sync                                                         |
| --------------------- | ------------------- | -------------------------- | ------------------------------------------------------------ |
| Building/shared rooms | Owner/administrator | Draft → Publish            | Publish increments version and broadcasts `layout.published` |
| Personal office       | Owner               | Autosave (debounced 500ms) | Broadcast `layout.updated` to occupants                      |
| DM guest              | Non-owner           | Read-only                  | Receives updates                                             |
| Channel room          | Org admins (MVP)    | Draft → Publish            | Same as building                                             |

**Editor UX:** toggle build mode (B), catalog drawer, grid overlay, ghost preview with valid/invalid tint, select, move, rotate (R), delete (Del), undo/redo (Ctrl+Z/Y), reset room, publish/cancel. Admin drafts are saved server-side per admin (`status = draft`) and do not affect the live room. Personal office autosaves.

**Real-time sync:** Centrifugo `office:room:{encodedRoomId}:layout` carries a notify payload `{ type: "published" | "updated", version, roomId, updatedBy }`; clients fetch the new layout on notify and ignore stale versions. Publish is transactional: validate → write version → broadcast notify.

**Persistence:** Zedu Postgres is the source of truth. No localStorage as source of truth; localStorage may cache the last layout for instant paint.

### 5.10 Admin building editor

- Visible only to `owner`/`administrator` (or the existing `manage:organization` permission) via `PermissionBoundary`.
- In-world edit mode covers lobby, lounge, reception, meeting rooms, focus pods, and channel rooms.
- Draft is per admin; only one publish at a time per room (`409` on version conflict).
- Publish writes a new `office_layouts` row with `status = published` and marks the previous published version `superseded`.
- “Reset to template” restores the seeded layout.
- Audit fields: `updated_by`, `published_at`, `version`.

### 5.11 Personal office & access

- One personal office per user, derived from `personal:<userId>` (no room row is created).
- Access check: owner OR active DM participant. For a 1:1 DM, the office owner is the counterpart on the requester's `dm_channels` row (`participant_id`). For a group DM the office owner is the earliest-added participant (earliest `channel_participants.created_at`, ties broken by lowest user id); other participants are read-only guests.
- Guests see the owner's published/autosaved layout and cannot edit.
- Personal office chat uses the DM conversation the visitor arrived from (1:1 or group); non-owner participants are view-only guests. If a guest enters from the directory with no DM context, use their 1:1 DM with the owner; if the owner enters with no visitor, show an empty room with a prompt to open a DM.
- Door labels show the owner's display name and a lock state.

### 5.12 UI, HUD, onboarding

- **HUD:** room name, floor name, online count, controls hint, quality toggle, mute toggle, exit to Zedu.
- **Minimap:** the current floor's room shapes, doors, local avatar, nearby avatars, Zoe marker.
- **Tower view:** the building from outside, with the floor rail, the lift, the lit offices and the general floor at the top (§5.6). It is the map and the lift ride, in the same route.
- **Search:** a people search in the HUD and in the tower view. A match tints the floor and pulses the office.
- **Room directory:** searchable list of channels/rooms with lock state and occupancy.
- **Onboarding:** first entry shows a 3-step card (move, enter a room, talk to Zoe), dismissible and persisted.
- **Loading:** progress for assets + room bootstrap; skeleton HUD.
- **Errors:** toasts via `showError`; retry on bootstrap failure; safe fallback to 2D Zedu.
- **Accessibility:** keyboard-only movement and room entry; focus trap in Zoe dialog; reduced-motion option; readable nameplates; color-blind-safe status icons (not color alone).

### 5.13 Mobile

- Touch joystick for movement; tap floor to move; tap avatar/room for actions.
- Lower quality tier by default; 30fps target.
- Zoe dialog full-screen sheet on small screens.
- Furniture editor supports tap-select and drag; rotate/delete buttons.
- If WebGL is unavailable, show a clear fallback with a link back to Zedu chat.

---

## 6. Data Models & APIs

### 6.1 Frontend Shared Schema (`office-zedu-fe/src/types/office.ts`)

This file is the **frozen contract**. Day 0 merges it with the room kinds, the floor model, the layout schema, the API DTOs and the Centrifugo channel names. Only the squad leads and their assistants may change it, and the Go side in `office-zedu-be/internal/models/office.go` mirrors it by hand.

```ts
export interface Vec3 {
  x: number;
  y: number;
  z: number;
}
export interface Quat {
  x: number;
  y: number;
  z: number;
  w: number;
}

export type RoomKind =
  | "lobby"
  | "reception"
  | "lounge"
  | "meeting"
  | "focus"
  | "channel"
  | "personal"
  | "area";

export type PresenceStatus = "online" | "idle" | "away" | "call" | "dnd";

export interface AvatarConfig {
  tint: string;
  body: "a" | "b" | "c";
  outfit: "casual" | "smart" | "hoodie" | "lab";
  accessory: "none" | "headphones" | "glasses" | "cap";
}

export interface UserPresence {
  userId: string;
  displayName: string;
  username: string;
  avatar: AvatarConfig;
  roomId: string;
  floorIndex: number;
  position: Vec3;
  rotation: number;
  status: PresenceStatus;
  activity?: string; // "On a call with Musa Garba", "In the lift to F1"
  updatedAt: number;
}

export interface OfficeRoom {
  id: string; // composite: "channel:ch_1", "personal:usr_2", "area:3:visitors", "lobby"
  kind: RoomKind;
  name: string;
  refId?: string; // channel_id, dm_channel_id, user_id, or "meeting:1"
  floorIndex: number; // 0 = general floor, 1..n = member floors (top-down)
  capacity: number;
  isLocked: boolean;
  bounds: { x: number; y: number; w: number; h: number };
}

export interface FurnitureItem {
  id: string;
  catalogId: string;
  x: number;
  y: number;
  rotation: 0 | 90 | 180 | 270;
  variant?: string;
}

export interface RoomLayout {
  roomId: string;
  floorIndex: number;
  version: number;
  grid: { width: number; height: number; tile: number };
  items: FurnitureItem[];
  updatedAt: string;
}

export interface OfficeFloor {
  index: number; // 0 = general floor, 1..n = member floors, top-down
  kind: "general" | "member";
  name: string; // "General", "Engineering", "Sales"
  capacity: number; // 40 for member floors
  occupancy: number;
  rooms: OfficeRoom[];
}

export interface BuildingBootstrap {
  building: { id: string; orgId: string; name: string };
  settings: { enabled: boolean; floorCapacity: number };
  floors: OfficeFloor[];
  rooms: OfficeRoom[];
  occupants: UserPresence[];
  layouts: Record<string, RoomLayout>;
  channelRoomMap: Record<string, string>;
  dmOfficeMap: Record<string, string>; // built per requester; 1:1 entries point to the counterpart's office
}

export interface OfficeStatus {
  enabled: boolean; // organisations.office_settings.enabled, read at runtime
}

export interface OfficeSearchResult {
  userId: string;
  displayName: string;
  role: string;
  floorIndex: number;
  roomId: string;
}

export interface ZoeRequest {
  messages: Array<{ role: "user" | "assistant"; content: string }>;
  context: {
    location: "office";
    buildingId: string;
    floorIndex: number;
    room: { id: string; kind: RoomKind; name: string };
    nearbyUsers: Array<{ displayName: string; status: PresenceStatus }>;
    user: { role: string; timezone: string };
    tasks: string[];
  };
}
```

### 6.2 Schema strategy: extend before duplicating

**Rule:** if Zedu already has the entity, key the office record to it. Create a new table only when the data has its own lifecycle (versioned layouts), is too large for a hot row (furniture JSON), or is a genuinely new relation.

| Existing table                                                        | Office change                                                                                                                                                     | Why not a new entity table                                                                                                                                                            |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `organisations`                                                       | Add `office_settings JSONB NULL` (`enabled`, `theme`, `floorCapacity`) — or a 1:1 `organisation_office_settings` row if you want zero impact on org serialization | The building **is** the organisation. `office_buildings` would duplicate `id`, `name`, and timestamps.                                                                                |
| `org_user_managements`                                                | Add `floor_index INT NULL` — the member's floor (1 = top member floor, 2 = next down). Null means unassigned.                                                     | The floor assignment is 1:1 with org membership and is read on nearly every office request. A separate table would need a join on every lookup and a second write path on join/leave. |
| `profiles`                                                            | Add `avatar_config JSONB NULL` (`tint`, `body`, `outfit`, `accessory`)                                                                                            | Avatar is 1:1 with the user's profile. `user_avatar_configs` adds a join for no benefit. If avatars must be identical across orgs, put the column on `users` instead.                 |
| `channels` / `dm_channels` / `channel_participants` / `user_channels` | No change                                                                                                                                                         | Channel, 1:1-DM, and group-DM membership already define room access. Do not duplicate rooms, members, or names.                                                                       |
| `channel_invitations`                                                 | Reuse for private channel access                                                                                                                                  | Already models channel-scoped invites.                                                                                                                                                |
| `audit_logs`                                                          | Reuse with new `ResourceOfficeLayout` and `ActionOfficeLayoutPublish` values                                                                                      | `AuditLog` already stores actor, resource, old/new JSON, and timestamps. `office_layout_events` is redundant.                                                                         |
| Centrifugo / Redis                                                    | Presence only                                                                                                                                                     | Positions are ephemeral; do not persist them in Postgres.                                                                                                                             |

**New tables (2)**

| Table                 | Key columns                                                                                                                                                       | Notes                                                                                                                                                            |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `office_layouts`      | `id`, `organisation_id`, `room_kind`, `ref_id`, `version`, `status`, `draft_owner_id?`, `layout JSONB`, `updated_by`, `published_at?`, `created_at`, `updated_at` | Versioned furniture. `ref_id` is `channel_id`, `user_id`, or a fixed key such as `meeting:1` or `area:3:visitors`. `status`: `draft`, `published`, `superseded`. |
| `office_room_configs` | `organisation_id`, `room_kind`, `ref_id`, `bounds JSONB`, `capacity`, `template`, `enabled`, `updated_by`, `updated_at`                                           | The table always exists; rows are written only for rooms that override the derived default (bounds, capacity, template).                                         |

Group DM access and ownership resolve through `channel_participants`; do not infer group membership from `dm_channels` alone. 1:1 DM ownership resolves through `dm_channels.participant_id` on the requester's row, so the mapping is built per requester.

**Indexes and constraints**

- `office_layouts`: partial unique `(organisation_id, room_kind, ref_id, version) WHERE status IN ('published', 'superseded')` — version history.
- `office_layouts`: partial unique `(organisation_id, room_kind, ref_id) WHERE status = 'published'` — one live layout per room.
- `office_layouts`: partial unique `(organisation_id, room_kind, ref_id, draft_owner_id) WHERE status = 'draft'` — one draft per admin per room.
- `office_layouts`: index `(organisation_id, room_kind, ref_id, status)`.
- `office_room_configs`: primary key `(organisation_id, room_kind, ref_id)`.
- `organisations.office_settings` and `profiles.avatar_config`: no index needed; nullable, default `NULL`.

**Why not add room columns to `channels`?**

- `channels` is a hot, wide table read by every channel list and message query. Adding `office_bounds`, `office_capacity`, etc. changes every read and every API payload.
- DMs live in `dm_channels`, not `channels`; personal offices map to `users`. Expanding `channels` would require parallel changes in three tables.
- A polymorphic `office_room_configs` keyed by `(room_kind, ref_id)` covers all room types without touching chat tables.

**Why keep layouts in a new table?**

- Multi-row, versioned, draft/published, and potentially large JSON. It does not belong on `channels` or `organisations`.
- The feature can be removed without altering core chat/org tables.
- Existing clients and mobile builds are unaffected.

**Migration rules**

- Add nullable/defaulted columns first; backfill only if needed; never rename or drop existing columns in this sprint.
- Use additive migrations in `internal/models/migrations/`.
- GORM models live in `internal/models/office.go`.
- Wrap `office_settings` and `avatar_config` with `json:"-"` in the base models and expose them through dedicated response DTOs so existing API payloads do not change.

### 6.3 API endpoints

**office-zedu-be (`/api/v1`, authorized)**

`roomId` in these routes is the opaque composite defined in §3.3 (`channel:<channelId>`, `personal:<ownerUserId>`, `area:<floor>:<key>`, `lobby`, `meeting:<n>`, `focus:<n>`, `lounge`, `reception`). URL-encode `:` as `%3A` in path segments (for example `channel%3Ach_1`). The service resolves it to `(room_kind, ref_id)` and checks access against existing `user_channels`, `dm_channels` + `channel_participants` (group DMs), and `org_user_managements` rows. Every handler first checks `organisations.office_settings.enabled` and answers `403 office_disabled` when the org is not enabled.

| Method | Path                                   | Purpose                                                                                                                                                                                          |
| ------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `GET`  | `/office/status`                       | `{ enabled }` for the caller's org. The client uses it to show the Office rail item; cheap enough to call on every org load                                                                      |
| `GET`  | `/office/settings`                     | The organisation's office options (`{ enabled, theme, capacity }`). Readable by anybody in the organisation and answered even when the office is off, so the switch can always be turned back on |
| `PUT`  | `/office/settings`                     | Save the office options. Owners and administrators only                                                                                                                                          |
| `GET`  | `/office/bootstrap`                    | Office settings + floor directory + rooms + occupants + mapping + published layouts                                                                                                              |
| `GET`  | `/office/floors`                       | Floor directory only: floors, occupancy, and each floor's room ids                                                                                                                               |
| `GET`  | `/office/floors/:floorIndex`           | One floor: its rooms, its `area` extras, and the lift/portal rooms                                                                                                                               |
| `GET`  | `/office/rooms/:roomId`                | Derived room metadata + access result                                                                                                                                                            |
| `GET`  | `/office/rooms/:roomId/layout`         | Published layout                                                                                                                                                                                 |
| `GET`  | `/office/rooms/:roomId/layout/draft`   | Read the requesting admin's draft (`status = draft`)                                                                                                                                             |
| `POST` | `/office/rooms/:roomId/layout/draft`   | Save admin draft (`office_layouts`, `status = draft`)                                                                                                                                            |
| `POST` | `/office/rooms/:roomId/layout/publish` | Validate + publish + broadcast (`If-Match` version)                                                                                                                                              |
| `POST` | `/office/rooms/:roomId/layout/reset`   | Reset a shared room to its seeded template (admin)                                                                                                                                               |
| `PUT`  | `/office/rooms/:roomId/layout`         | Autosave personal office layout (owner only)                                                                                                                                                     |
| `GET`  | `/office/rooms/:roomId/occupants`      | REST fallback snapshot                                                                                                                                                                           |
| `GET`  | `/office/people`                       | People search: name, role, floor index, office room id                                                                                                                                           |
| `GET`  | `/users/me/avatar`                     | Read avatar config from `profiles.avatar_config`                                                                                                                                                 |
| `PUT`  | `/users/me/avatar`                     | Save avatar config                                                                                                                                                                               |
| `POST` | `/office/rooms/:roomId/enter`          | Access check + Centrifugo subscription authorization                                                                                                                                             |
| `POST` | `/office/rooms/:roomId/leave`          | Leave + status cleanup                                                                                                                                                                           |

**office-zedu-fe server route**

| Method | Path              | Purpose                                                             |
| ------ | ----------------- | ------------------------------------------------------------------- |
| `POST` | `/api/office/zoe` | Validate Zedu token, build context, proxy SSE to contributor-helper |

**contributor-helper (extension)**

| Method | Path        | Change                                                                    |
| ------ | ----------- | ------------------------------------------------------------------------- |
| `POST` | `/api/chat` | Accept `ZOE_SERVICE_TOKEN` + trusted identity headers; keep NextAuth path |

### 6.4 Centrifugo channels

| Channel                                                               | Payload                                                        | Notes                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `office:org:{orgId}:floor:{floorIndex}:room:{encodedRoomId}:presence` | `{ type: "join"\|"leave"\|"update", presence: UserPresence }`  | Position/status; server-validated. `encodedRoomId` is URI-encoded (`channel%3Ach_1`). The organisation id is part of the name so two orgs can never share a channel, and the token endpoint rejects a channel whose org does not match the caller. |
| `office:org:{orgId}:floor:{floorIndex}:room:{encodedRoomId}:layout`   | `{ type: "published"\|"updated", version, roomId, updatedBy }` | Clients fetch the new layout. Published on admin publish, on reset, and on a personal office autosave.                                                                                                                                             |
| `office:user:{userId}:zoe`                                            | `{ type: "tip", content }`                                     | Stretch: proactive Zoe                                                                                                                                                                                                                             |

Redis presence keys are namespaced the same way: `office:presence:{orgId}:{floorIndex}:{roomId}`.

Chat is **not** re-published on an office channel. The client reuses the existing channel/DM Centrifugo events and filters them to the mapped conversation id. Channel rooms live on the general floor, so a channel's messages are the same wherever the reader is standing.

Subscribe authorization must check room access server-side. Do not rely on client-side checks.

### 6.5 Permissions matrix

| Action                         | Owner/Administrator | Channel member | Personal office owner | DM guest            |
| ------------------------------ | ------------------- | -------------- | --------------------- | ------------------- |
| Enter building                 | Yes                 | Yes            | Yes                   | Yes                 |
| Enter a member floor           | Yes                 | Yes            | Yes                   | Yes                 |
| Enter the general floor        | Yes                 | Yes            | Yes                   | Yes                 |
| Enter channel room             | Yes                 | Yes            | Yes                   | No                  |
| Enter personal office          | Yes                 | No             | Yes                   | DM participant only |
| Edit shared furniture          | Yes                 | No             | No                    | No                  |
| Publish shared layout          | Yes                 | No             | No                    | No                  |
| Edit own office furniture      | Yes                 | Yes            | Yes                   | No                  |
| Start Buzz in meeting room     | Yes                 | Yes            | Yes                   | Yes                 |
| Ride the lift to another floor | Yes                 | Yes            | Yes                   | Yes                 |

Gate admin UI with `PermissionBoundary permission="manage:organization"` from `src/components/rbac/`. A dedicated `manage:office` permission is a later enhancement, not a sprint dependency.

### 6.6 Validation & concurrency

- Layout save validates grid bounds, overlap, reserved tiles, catalog rules, and version.
- **Admin publish:** transaction — validate → insert new `office_layouts` row with `status = 'published'`, `version = max(non-draft version for the room) + 1`, and `published_at = now()` → mark the previous published row `status = 'superseded'` → enqueue Centrifugo broadcast. Retain the last 10 superseded versions per room (cleanup job or on-publish prune).
- **Personal autosave:** update the single published row in place and increment `version` (and `published_at`); no draft row. `PUT` uses `If-Match`; on mismatch return `409` with the latest layout and a “reload” toast.
- **Admin draft:** one `status = 'draft'` row per `(room, admin)`, enforced by the partial unique index. Drafts use `version = 0` and do not participate in the published/superseded version index. Saving a draft does not affect the live room.
- `office_room_configs` writes are admin-only back-office operations this sprint (seeded or updated outside the in-world editor); they do not version furniture.
- Chat and presence are not stored in Postgres as layout data; presence is ephemeral in Centrifugo/Redis.
- Publish events are written to the existing `audit_logs` table with `ResourceType = office_layout` and `Action = office.layout_publish`; no separate event table.

---

## 7. 3-Day Sprint Plan

### 7.1 Operating model

- **One sprint, three days of shipping.** `dev` is production, so every merge into it is a release. The flow is: a contributor merges into their **squad branch** → the **squad lead** reviews and tests the squad branch and opens a **PR from the squad branch into `dev`** → a **second reviewer from another team** reviews that PR → it merges into `dev` and is live.
- **~40 contributors in 5 build squads**, ~8 per squad, each with a squad lead. SQ0 is those five leads plus their assistants: it owns the frozen artefacts and the release mechanics, and it ships no features. Squads run parallel sub-tasks inside their own branch.
- **Task size:** 2–6 hours per task, one owner, one PR, one reviewer. No task spans two days.
- **Model first.** The tower model, the floor rules and the shared schema are frozen before the feature squads start, because every squad builds on them. Only SQ0 changes the frozen artefacts.
- **No global kill switch.** `organisations.office_settings.enabled` is the per-org switch and it is read at runtime: enabling or disabling an org takes effect on the next request, with no deployment. That is also the fast way to take the office away from the pilot org if a slice misbehaves. Owners and administrators change it in the organisation settings, under **The Office**.
- **Branches, not file walls.** Each squad works on its own branch and merges into the squad branch; the lead opens the PR that promotes it to `dev`. File ownership is agreed per task on the board rather than enforced by a hard boundary.
- **Mocks before backend.** Frontend squads build against the mock bootstrap fixture (3 floors: two member floors of 40 and the general floor) until the API lands.
- **Daily integration:** the day runs from 00:00. Integration check at 12:00 inside each squad branch, freeze at 16:30, PR reviews start at 18:00, reviewed PRs merge into `dev` by 20:00, the demo is due at 21:00 in the daily unblocking, and the release notes are posted by 23:00. A squad PR that is not green and cross-team approved by 21:00 waits for the next midnight.
- **Cut line:** if a feature is not green by 16:30, it does not ship that day; do not hold the day's PRs.

### 7.2 Squads and staffing

**Five build squads plus SQ0.** SQ0 is not a sixth build squad: it is the council of the five squad leads and their assistants, and every member of it also leads one of the squads below. It owns the frozen artefacts and the release mechanics, and it ships no features of its own.

| Squad                                                                 | Scope                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SQ0 Squad leads council** (the five squad leads + their assistants) | The frozen artefacts: `src/types/office.ts` (room kinds, floor model, layout schema, API DTOs, Centrifugo names), the migration spec, the mock bootstrap fixture, the asset list and attribution, the release and revert runbook. Arbitrates cross-squad contract changes and owns the revert decision.                           |
| **SQ1 Frontend**                                                      | The `/[org]/office` route, the rail entry and `use-office-status`, the HUD, the minimap, the room directory, the floor rail, the people search UI, the nameplate and status icon overlay, the DM window, onboarding, mobile and touch layout, the furniture editor controls, and the `src/lib/office/` API client.                |
| **SQ2 Backend**                                                       | `org_user_managements.floor_index` and the join/leave re-balance, the floor-keyed layout tables, the bootstrap and floor directory, access rules per floor, `GET /office/status`, the people search endpoint, the avatar and layout APIs, draft/publish and version checks.                                                       |
| **SQ3 3D & Rooms**                                                    | The renderer and scene lifecycle, the procedural floor generator (two tiers, 40 offices, corridor, lift, portal door), the per-floor `area` extras, the tower scene with cutaway walls, the exterior camera and the fly-in transition, movement, collision, click-to-move, and the furniture meshes with their grid-snap preview. |
| **SQ4 Realtime & AI**                                                 | The org-namespaced Centrifugo presence and layout channels, the token endpoint channel authorisation, per-office presence and status, activity strings, chat bubbles and distance fade, the Zoe proxy and streaming, and the Zoe context builder.                                                                                 |
| **SQ5 QA & Release**                                                  | The scripted scenario harness, the two-browser presence tests, the floor and lift regression pass, the smoke checklist, the release checklist, bug triage, the demo script and the rollback drill.                                                                                                                                |

The split on the frontend is deliberate: **SQ1** owns the web surface, **SQ3** owns what is rendered in 3D, and **SQ4** owns what is driven by realtime events. An element is owned by the squad that ships it, agreed per task on the board, so nobody waits on a squad that is not chartered to build it.

### 7.3 What the tower needs on day 1

The floor interior already exists, so the tower is the first milestone: the same engine seen from outside, with cutaway walls, a lit office per occupant, and a click that flies the camera into the room. It ships on day 1 and every later slice lands on top of it.

### 7.4 What is frozen, and how it changes

Frozen on day 0, by SQ0, with lead consensus:

- `src/types/office.ts` — the room kinds (including `area`), the floor fields, the layout schema, the API DTOs and the Centrifugo channel names
- the migration spec — `org_user_managements.floor_index`, the layout tables, the two nullable JSONB columns
- the Centrifugo channel name format, including the organisation id and the floor index
- the mock bootstrap fixture shape

A change to any of them is a PR from SQ0 with the leads' agreement, and it lands before the squads that depend on it.

### 7.5 Daily cadence

The working day runs from **midnight** and closes with the demo at **21:00**, inside the daily unblocking; the release notes go on the board by **23:00**, so the next day starts with the unblocking decisions already written down.

| Time  | Activity                                                                                       |
| ----- | ---------------------------------------------------------------------------------------------- |
| 00:00 | Day starts; build block 1 opens, and every PR already has its reviewer named                   |
| 12:00 | Integration check; task branches merge into the squad branch                                   |
| 16:30 | Code freeze for the day's PRs; only release-blocking fixes                                     |
| 18:00 | **PR reviews start.** Squad leads open their PRs into `dev`; cross-team reviewers pick them up |
| 20:00 | Reviewed PRs merge into `dev` and go live                                                      |
| 21:00 | **Demo and next-day unblocking:** what shipped, what is blocked, what starts at midnight       |
| 23:00 | Release review notes posted; unblocking decisions recorded on the board                        |

### 7.6 Task template

Every task must include:

- **Outcome:** one sentence.
- **Files owned:** exact paths.
- **Shared schema:** request/response or props.
- **Acceptance:** 1–3 checks a reviewer can run.
- **Mock:** what to use if the dependency is not ready.
- **Revert unit:** the PR(s) to revert if this misbehaves.
- **Estimate:** 2–6 hours.
- **Reviewers:** the squad lead first, then one reviewer from another team on the squad's PR into `dev`.

### 7.7 AI-assisted contributor guardrails

Because most contributors are using AI:

1. **One task, one folder.** Do not let AI refactor unrelated code.
2. **Shared schema is read-only** unless SQ0 approves.
3. **No new dependencies** without SQ0 approval; `three` is the only planned addition.
4. **Generated code must pass** `pnpm check-format`, `check-lint`, `check-types`, `build`; Go must pass `go build ./...` and `go test ./tests/... -p 1`.
5. **Two gates before `dev`:** the squad lead reviews and tests the squad branch, then a reviewer from another team approves the PR into `dev`. No self-merge.
6. **Screenshots or a 30s clip** for UI tasks.
7. **No secrets in code.** Zoe service token stays server-side.
8. **Ship behind the org switch.** `office_settings.enabled` is the per-org switch and is read at runtime; a broken slice is reverted, not patched live. Keep PRs independently revertable.
9. **Ask before touching** protected files (`.github/`, AGENTS.md, CONTRIBUTING.md, tooling config, `internal/config/`).
10. **Keep PRs under ~400 meaningful lines.** Split larger work.

---

## 8. QA, Release & Risk

### 8.1 Test matrix

| Test                                               | Type                          | When   |
| -------------------------------------------------- | ----------------------------- | ------ |
| Login redirect and org switch                      | Manual                        | Daily  |
| Enter the tower, fly into an office, move, collide | Manual + Cypress smoke        | Daily  |
| Two users see each other                           | Manual, two browsers          | Day 2+ |
| Channel room access vs locked door                 | Manual + Go test              | Day 2+ |
| DM private office access                           | Manual + Go test              | Day 2+ |
| Lift ride lands on the right floor                 | Manual, two browsers          | Day 2+ |
| Search tints the floor and pulses the office       | Manual                        | Day 2+ |
| Floor re-balance on join and leave                 | Go test                       | Day 2+ |
| Cross-org channel isolation                        | Go test                       | Day 2+ |
| Zoe streaming + grounding                          | Manual + API test             | Day 2+ |
| Zoe service-token rejection                        | contributor-helper API test   | Day 2+ |
| Furniture place/rotate/delete                      | Manual + Go test (validation) | Day 3+ |
| Publish + real-time reconciliation                 | Manual, two browsers          | Day 3+ |
| Personal autosave + reload                         | Manual + Go test              | Day 3+ |
| Version conflict 409                               | Go test                       | Day 3+ |
| 20-user presence stress                            | Manual                        | Day 3  |
| Mobile 30fps                                       | Manual                        | Day 3  |
| Non-office regression (2D paths)                   | Manual                        | Daily  |
| Rollback drill                                     | Manual                        | Day 3  |

**Required commands before merge**

```sh
# office-zedu-fe
pnpm check-format
pnpm check-lint
pnpm check-types
pnpm build

# office-zedu-be
go build ./...
go test ./tests/... -p 1 -timeout 600s
```

### 8.2 Release checklist

- [ ] The squad branch is green: format, lint, types and build (or the Go build and tests) pass on it.
- [ ] The squad lead has tested the slice against the dev deployment.
- [ ] The PR from the squad branch into `dev` names the slice and its revert unit.
- [ ] A reviewer from another team has approved that PR.
- [ ] `office_settings.enabled` is true only for the org the slice is meant to be visible to.
- [ ] Revert plan: the exact commits to revert are identified before the PR merges.
- [ ] Backend migration applied; additive only.
- [ ] Smoke test passed on `dev` after the merge.
- [ ] Rollback: revert the merge on `dev`; no DB rollback is needed for additive migrations, and disabling the pilot org takes the office away immediately without a deploy.
- [ ] On-call owner named for the release window.
- [ ] Demo notes posted.

### 8.3 Risks and mitigations

| Risk                                                          | Likelihood | Impact   | Mitigation                                                                                                                     |
| ------------------------------------------------------------- | ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Scope exceeds 3 days                                          | High       | High     | Must/Should/Stretch cut line; cut stretch first; unshipped slices wait for the next window                                     |
| Contributors + AI create conflicts                            | High       | High     | Task ownership, shared schema, small tasks, lead review, mocks                                                                 |
| Merge conflicts across ~40 people                             | High       | Medium   | Short-lived branches; the squad branch absorbs intra-squad conflicts; SQ0 arbitrates the frozen artefacts                      |
| A floor of 40 offices is too heavy to render                  | Medium     | High     | Only the current floor is built; the tower uses cutaway geometry without interiors                                             |
| Centrifugo or Elasticsearch missing in the target environment | Medium     | Critical | The prerequisites in §9.1.1 are checked before the sprint; presence and chat both depend on them                               |
| Centrifugo room authorization gaps                            | Medium     | Critical | Server-side access checks; subscribe authorization tests; org id in every channel name                                         |
| Zoe proxy auth mismatch                                       | Medium     | High     | Service-token path built early; stub fallback; port plan ready                                                                 |
| Furniture sync conflicts                                      | Medium     | Medium   | Version checks, 409 recovery, publish transaction                                                                              |
| Privacy leak in private offices/DMs                           | Low        | Critical | Server-enforced access; no client-only checks; context caps                                                                    |
| Asset licensing                                               | Low        | Medium   | CC0 packs only (Kenney/Quaternius/KayKit) plus the team-owned prototype assets with attribution; license log; no ripped assets |
| Protected-file changes block PR                               | Medium     | Medium   | Request approval for `internal/config/`, `.github/pr-review.config.json`, and tooling                                          |
| Cross-team review becomes a bottleneck                        | Medium     | High     | Reviewers named per PR when it opens at 18:00; a PR that misses its reviewer waits a day rather than shipping unreviewed       |
| PR-review bot rejects new office paths                        | High       | High     | Extend `.github/pr-review.config.json` allowlist with reviewer approval (`config-change-approved`)                             |

### 8.4 Cut lines (MoSCoW)

**Must**

- Zedu auth/session.
- Building shell + movement + collision.
- Channel rooms + DM private offices.
- Presence + status.
- Zoe proximity + streaming proxy.
- Admin building furniture + personal office furniture + persistence + real-time publish.
- Avatar tint + presets + persistence.
- The tower: floors, the general floor of channels, and the click-to-fly-in transition.

**Should**

- Speech bubbles and distance fade.
- Minimap and room directory.
- Focus pod DND.
- Meeting room Buzz action.
- Onboarding card.
- Person search that tints the floor and pulses the office.
- Per-floor `area` extras beyond a seeded default.

**Stretch**

- Zoe proactive tips.
- Room capacity animations.
- Avatar accessory variety.
- Layout templates per channel type.
- Voice input for Zoe.
- Buzz call started from a clicked office.

**Cut**

- Personal-office guest invites — not modeled; access is owner + active DM participants. Revisit post-sprint.
- Zoe rate limiting — deliberate acceptance for this sprint; context caps only.

### 8.5 Decisions and assumptions

1. DM ↔ personal office ownership: resolved — for a 1:1 DM, the counterpart on the requester's `dm_channels` row (`participant_id`); for a group DM, the earliest-added participant (`channel_participants.created_at`); non-owners are read-only guests.
2. Channel room furniture is admin-only in MVP; channel managers can edit later.
3. Personal-office guest invites are cut; access is owner + active DM participants.
4. Meeting rooms are occupancy-based, not calendar-booked.
5. Building capacity is 500 concurrent users; target load is 100.
6. Zoe text only; no voice.
7. Primitive + low-poly GLTF assets only.
8. Floor capacity is **40**, not a range; a floor is filled top-down and the top member floor may be partially filled.
9. Every channel in the org sits on the general floor, so a conversation is one portal door away from any floor.
10. The tower is 3D, in the same engine as the floor interior; there is no separate 2D renderer.
11. The office is switched on per org by `organisations.office_settings.enabled`, read at runtime. There is no env var and no build-time switch, so an org can be enabled or disabled without a deployment. Owners and administrators flip it in the organisation settings, under **The Office**.
12. The office rail item renders from `GET /api/v1/office/status`, so a disabled org never sees an entry point, and a deep link to `/[org]/office` for a disabled org redirects to the organisation landing `/[org]`.
13. `PUT /api/v1/office/settings` is writable by owners and administrators only, and it answers even when the office is off, so the switch can always be turned back on.

---

## 9. Appendices

### 9.1 Environment variables

```env
# office-zedu-fe
NEXT_PUBLIC_OFFICE_TILE_SIZE=1
NEXT_PUBLIC_OFFICE_PRESENCE_HZ=10
NEXT_PUBLIC_OFFICE_PROXIMITY_RADIUS=10
NEXT_PUBLIC_OFFICE_FLOOR_CAPACITY=40
NEXT_PUBLIC_ZOE_INTERACTION_RADIUS=2

# office-zedu-fe server only
ZEDU_API_URL=https://api.example.com
ZOE_SERVICE_URL=https://contributors.example.com
ZOE_SERVICE_TOKEN=server-only-secret

# contributor-helper server only
ZOE_ALLOW_SERVICE_AUTH=false
ZOE_SERVICE_TOKEN=server-only-secret

# office-zedu-be
CENTRIFUGO_OFFICE_CHANNEL_PREFIX=office
```

There is **no office env var and no build-time switch**. `organisations.office_settings.enabled` is the per-org switch, read at runtime through `GET /api/v1/office/status`, so enabling or disabling an org never needs a deployment. `NEXT_PUBLIC_OFFICE_FLOOR_CAPACITY` mirrors the server's floor capacity (40) for the client-side floor generator. `ZOE_ALLOW_SERVICE_AUTH` is set true only in the deployment that serves the office; the sample keeps it false. Secrets are server-only. Never expose `ZOE_SERVICE_TOKEN` via `NEXT_PUBLIC_*`.

### 9.1.1 Environment prerequisites

Two services must be live in the target environment before the office has realtime presence, and before the platform's own chat works at all:

- **Centrifugo** reachable by the browser at `NEXT_PUBLIC_CONNECT_URL`, publishing to `CENTRIFUGE_URL`, with `HMAC_SECRET` equal to the Centrifugo server secret. The subscription token endpoint signs with that secret; a mismatch silently stops every subscription.
- **Elasticsearch** reachable at `ELASTIC_URL` with a valid `ELASTIC_API_KEY`, with the `messages` and `threads` indices present. Messages are stored only in Elasticsearch, so an unreachable cluster breaks every send with a 500.

### 9.2 Dependencies

```json
{
  "dependencies": {
    "three": "^0.160.0"
  },
  "devDependencies": {
    "@types/three": "^0.160.0"
  }
}
```

No physics engine. Grid AABB collision and A\* are implemented in `engine/grid.ts`. Avoid adding `@react-three/fiber` unless SQ0 approves; a plain Three.js scene keeps the bundle smaller and the AI-generated code easier to review. The version above is illustrative — pin the current stable `three` and `@types/three` at Day 0, and keep them in lockstep.

### 9.3 Asset list

| Asset                                   | Source                                               | Notes                                                                                                         |
| --------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Avatar base                             | Quaternius / KayKit character packs (CC0)            | Rigged low-poly GLB, idle/walk/run; mix-and-match preset parts                                                |
| Zoe desk                                | Primitive/custom low-poly                            | System item, not editable                                                                                     |
| Desk, chair, table, sofa                | Kenney Furniture Kit (CC0)                           | Instanced                                                                                                     |
| Plant, lamp, rug, shelf                 | Kenney (CC0)                                         | Instanced                                                                                                     |
| Whiteboard, screen                      | Primitive                                            | Emissive screen optional                                                                                      |
| Focus booth                             | Primitive                                            | System item                                                                                                   |
| Meeting glass                           | `MeshPhysicalMaterial`                               | Transparent, no transmission on mobile                                                                        |
| Office shell, tower cutaway, floor plan | The team's own prototype (`zedu-tower`, by Nexprove) | Team-owned: reuse the geometry and interaction model with attribution, and reimplement in the Three.js engine |
| Office interior props                   | The team's own prototype (`zedu-tower`)              | Same terms: reuse with attribution                                                                            |

Record the license and source for every asset in `docs/office/ASSETS.md`, including the attribution line for the team-owned prototype assets.

### 9.4 Shared Schema examples

**Bootstrap response**

```json
{
  "building": { "id": "org_1", "orgId": "org_1", "name": "Zedu HQ" },
  "settings": { "enabled": true, "floorCapacity": 40 },
  "floors": [
    {
      "index": 0,
      "kind": "general",
      "name": "General",
      "capacity": 40,
      "occupancy": 0,
      "rooms": ["lobby", "reception", "channel:ch_1"]
    },
    {
      "index": 1,
      "kind": "member",
      "name": "Engineering",
      "capacity": 40,
      "occupancy": 38,
      "rooms": ["personal:usr_1", "area:1:visitors"]
    }
  ],
  "rooms": [
    {
      "id": "lobby",
      "kind": "lobby",
      "name": "Lobby",
      "refId": null,
      "floorIndex": 0,
      "capacity": 100,
      "isLocked": false,
      "bounds": { "x": 0, "y": 0, "w": 40, "h": 30 }
    },
    {
      "id": "channel:ch_1",
      "kind": "channel",
      "name": "#general",
      "refId": "ch_1",
      "floorIndex": 0,
      "capacity": 30,
      "isLocked": false,
      "bounds": { "x": 4, "y": 18, "w": 12, "h": 10 }
    },
    {
      "id": "area:1:visitors",
      "kind": "area",
      "name": "Visitors area",
      "refId": "1:visitors",
      "floorIndex": 1,
      "capacity": 6,
      "isLocked": false,
      "bounds": { "x": 2, "y": 4, "w": 6, "h": 6 }
    }
  ],
  "occupants": [
    {
      "userId": "usr_1",
      "displayName": "Ada Obi",
      "username": "ada",
      "avatar": {
        "tint": "#8b5cf6",
        "body": "b",
        "outfit": "smart",
        "accessory": "glasses"
      },
      "roomId": "personal:usr_1",
      "floorIndex": 1,
      "position": { "x": 3, "y": 5, "z": 0 },
      "rotation": 0,
      "status": "online",
      "activity": "At their desk",
      "updatedAt": 1792600000000
    }
  ],
  "layouts": {
    "channel:ch_1": {
      "roomId": "channel:ch_1",
      "floorIndex": 0,
      "version": 3,
      "grid": { "width": 12, "height": 10, "tile": 1 },
      "items": [
        {
          "id": "f_1",
          "catalogId": "table_meeting",
          "x": 4,
          "y": 4,
          "rotation": 0
        }
      ],
      "updatedAt": "2026-10-19T10:00:00Z"
    }
  },
  "channelRoomMap": { "ch_1": "channel:ch_1" },
  "dmOfficeMap": { "dm_1": "personal:user_2" }
}
```

`dmOfficeMap` is built for the requesting user. For a 1:1 DM, the same DM channel id maps to the counterpart's office for each caller, so A's map points to B's office and B's map points to A's office. For a group DM, it points to the earliest-added participant's office.

**Presence event**

```json
{
  "type": "update",
  "presence": {
    "userId": "usr_2",
    "displayName": "Ada",
    "username": "ada",
    "avatar": {
      "tint": "#7c3aed",
      "body": "b",
      "outfit": "smart",
      "accessory": "glasses"
    },
    "roomId": "personal:usr_2",
    "floorIndex": 1,
    "position": { "x": 4.2, "y": 0, "z": 7.8 },
    "rotation": 1.57,
    "status": "online",
    "activity": "In the lift to F1",
    "updatedAt": 1760875200000
  }
}
```

Published on `office:org:{orgId}:floor:{floorIndex}:room:{encodedRoomId}:presence`.

**Publish request**

```http
POST /api/v1/office/rooms/channel%3Ach_1/layout/publish
Authorization: Bearer <token>
If-Match: 3
Content-Type: application/json

{ "items": [ { "id": "f_1", "catalogId": "table_meeting", "x": 4, "y": 4, "rotation": 0 } ] }
```

### 9.5 Definition of Ready / Done

**Ready**

- Shared schema is locked.
- Task ownership is clear.
- Acceptance checks are written.
- Mock is available or dependency is merged.
- Revert unit is known.
- Estimate ≤ 6 hours.

**Done**

- Acceptance checks pass.
- Format/lint/types/build or Go build/tests pass.
- Reviewer approved.
- Independently revertable: reverting this PR alone restores prior behaviour.
- Screenshot/clip for UI.
- No unrelated changes.

### 9.6 Glossary

| Term              | Meaning                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------- |
| Building          | The org-wide 3D space: the tower.                                                                             |
| Tower             | The building seen from outside, in the same engine, with cutaway walls.                                       |
| Floor             | A storey of the tower. Floor 0 is the general floor; member floors are 1..n, filled top-down, 40 people each. |
| General floor     | The top floor. Holds every channel in the org, plus reception, lounge, meeting rooms and focus pods.          |
| Member floor      | A floor of private offices, one per member, with per-floor `area` extras.                                     |
| Portal door       | The door from a member floor to the general floor.                                                            |
| Lift              | The way between floors. The ride is played in the tower overlay.                                              |
| Area              | A per-floor extra room: visitors area, open pods, game area, facilities.                                      |
| Room              | An addressable interior: lobby, channel, personal, meeting, focus, lounge, reception, area.                   |
| Room-scoped scene | The client loads only the current floor plus the tower shell, not every room.                                 |
| Layout            | Grid-snapped furniture document for one room.                                                                 |
| Draft             | Unpublished admin layout.                                                                                     |
| Publish           | Validated, versioned layout visible to everyone.                                                              |
| Presence          | Ephemeral position/status data transported by Centrifugo, namespaced per organisation.                        |
| Zoe               | The in-world contributor assistant backed by the knowledge base.                                              |

---

**End of PRD**
