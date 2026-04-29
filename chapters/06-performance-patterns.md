# Chapter 6 — Performance Patterns

> *Written against SpacetimeDB 2.0/2.1. Canonical implementation: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) message and presence subscriptions, plus generic patterns for high-entity-count visualizations.*

## The problem

SpacetimeDB's real-time model has a property that's easy to miss until it bites you: **every subscription update fires a callback per row, and those callbacks land on the main thread**. Add up enough rows, enough subscriptions, and enough connected clients, and your application stops being "real-time" and starts being "perpetually 200ms behind."

Three failure modes we've hit at scale:

1. **Wide subscriptions.** A user subscribed to all 10,000 messages in a server pays for every insert/update/delete. They probably only need the visible viewport.
2. **Hot tables on cold callbacks.** A presence table updates every second per online user. Each update fires onUpdate. React's reconciler runs. The browser becomes a slideshow.
3. **Large entity counts under DOM rendering.** A real-time visualization of 25,000 entities updating positions 30× per second is impossible if every entity is a React component reading from a hook reading from a subscription.

The patterns in this chapter address each. Some are STDB-specific. Some are "this is just performance work that takes specific shape because STDB is in the loop."

## Pattern 1: Hot/cold table split

If a logical entity has both rapidly-changing fields (status, presence, position, last_message_at) and stable fields (display name, avatar, role), put them in different tables. Subscribe to the hot table at high frequency; subscribe to the cold table once.

The chat module is a small-scale example of this pattern. `chat_users` carries every user's identity-stable data (`user_id`, `display_name`, `avatar_data`, `tier`, `is_bot`, `bot_owner_user_id`) — which changes rarely — alongside hot fields (`online`, `status`, `last_seen_at`, `last_message_at`, `last_typing_at`) that change every few seconds for active users. A user with a 12-month-old display name might have its `last_seen_at` updated 1000+ times per session.

Splitting these would look like:

```rust
// Cold — subscribed once, rarely updates
#[spacetimedb::table(accessor = chat_user_profile, public)]
pub struct ChatUserProfile {
    #[primary_key]
    pub user_id: String,
    pub display_name: String,
    pub avatar_data: Option<String>,
    pub tier: Option<String>,
    pub platform_role: Option<String>,
    pub is_bot: Option<bool>,
    pub bot_owner_user_id: Option<String>,
    pub created_at: u64,
}

// Hot — subscribed at the same scope, updates per heartbeat / message / status change
#[spacetimedb::table(accessor = chat_user_presence, public)]
pub struct ChatUserPresence {
    #[primary_key]
    pub user_id: String,
    pub online: bool,
    pub status: String,
    pub last_seen_at: u64,
    pub last_message_at: u64,
    pub last_typing_at: u64,
}
```

The chat module ships these in one combined `chat_users` table because the user count is bounded (presence updates aren't a bandwidth problem at chat scale) and the operational simplicity of "one user, one row" is worth the extra wire bytes per heartbeat. We've kept the pattern as a deliberate non-application — it's right for a chat-scale workload to *not* split.

Where the pattern absolutely earns its weight is real-time visualization workloads: think a logistics tracker rendering 5,000 vehicles updating positions every second, or a game with 10,000 networked entities, or a sensor dashboard where every device reports a heartbeat per minute. In those cases the naïve schema (one wide row per entity, full-row update on every change) wastes bandwidth proportional to the cold-field width × the hot-field update rate. Splitting cuts that proportionally.

The cold table is subscribed once at session start with no filter; the hot table is subscribed on a viewport-based filter or unbounded if entity counts are bounded. Joins happen client-side at render time:

```typescript
const profiles = useTable<ChatUserProfile>('chat_user_profile');
const presence = useTable<ChatUserPresence>('chat_user_presence');

const profileMap = useMemo(
  () => new Map(profiles.map(p => [p.user_id, p])),
  [profiles],
);

const merged = useMemo(
  () => presence.map(p => ({ ...p, ...profileMap.get(p.user_id) })),
  [presence, profileMap],
);
```

### When to split, when not to

Split when:
- Some columns update orders of magnitude more often than others
- The high-frequency columns are small (numbers, short strings)
- The low-frequency columns are large or many

Don't split when:
- Update rates are similar across columns
- The split forces a join that runs on every render
- The application logic genuinely operates on whole entities together (chat messages don't split this way)

## Pattern 2: GPU instanced rendering against subscription buffers

Once you have a hot table with 1K+ entities updating in real time, the bottleneck shifts from network to render. React, even with memoization, can't reconcile thousands of entities at 30Hz. Neither can SVG. Neither can canvas if you draw each entity as a separate path.

The pattern that scales is **WebGL2 instanced rendering, with subscription callbacks writing directly into a `Float32Array` that the GPU reads.** Sketch:

```typescript
const FLOATS_PER_ENTITY = 9;
const buffer = new Float32Array(MAX_ENTITIES * FLOATS_PER_ENTITY);
const dirtyRange = { lo: Infinity, hi: -Infinity };

// Map entity_id -> slot index in the buffer
const slotMap = new Map<string, number>();

function onEntityUpdate(row: { entity_id: string, x: number, y: number, /* ... */ }) {
  const slot = slotMap.get(row.entity_id);
  if (slot === undefined) return;
  const offset = slot * FLOATS_PER_ENTITY;
  buffer[offset]     = row.x;
  buffer[offset + 1] = row.y;
  // ... etc
  dirtyRange.lo = Math.min(dirtyRange.lo, offset);
  dirtyRange.hi = Math.max(dirtyRange.hi, offset + FLOATS_PER_ENTITY);
}

// Per-frame: upload only the dirty range to GPU, then draw all instances in ONE call
function renderFrame() {
  if (dirtyRange.hi > dirtyRange.lo) {
    gl.bufferSubData(gl.ARRAY_BUFFER, dirtyRange.lo * 4,
                     buffer.subarray(dirtyRange.lo, dirtyRange.hi));
    dirtyRange.lo = Infinity;
    dirtyRange.hi = -Infinity;
  }
  gl.drawArraysInstanced(gl.POINTS, 0, 1, activeEntityCount);
}
```

Key properties:

- **Subscription callbacks write directly into the typed array.** No allocation per update. No React state. The array IS the source of truth for the GPU.
- **Dirty-range tracking** so we only upload changed slots, not the whole buffer every frame.
- **One GPU draw call** for all entities via `drawArraysInstanced`. The GPU reads from the array and renders thousands of points/sprites/meshes in one operation.
- **React is bypassed entirely.** The visualization is a `<canvas>` with a game-loop render. React mounts the canvas once and never re-renders it.

The pattern's ceiling is roughly "however much your GPU can render in one frame, however much your network can deliver per second." On a modest laptop GPU, that's tens of thousands of entities at 60fps; the bottleneck shifts back to subscription bandwidth, not render performance.

### When to reach for GPU instanced rendering

- Entity count > 1000
- Update rate > 10Hz per entity
- Entities have a regular shape (point, sprite, mesh) that can be instanced
- Position-based 2D or 3D layout (not flowing text or UI components)

Anything outside that envelope, regular DOM/SVG/canvas is probably fine and much simpler.

## Pattern 3: rAF-batched subscription updates

(Cross-reference to chapter 3.) Even when you're not GPU-rendering, the rAF batching pattern is the single most-impactful change you can make to a chat-style or activity-feed-style UI.

The chat module's `useTable` hook coalesces all `onInsert`/`onUpdate`/`onDelete` callbacks within a frame into one `setState`:

```typescript
// shared/spacetimedb/createHooks.ts (excerpt)
let rafId: number | null = null;
const flush = () => {
  rafId = null;
  if (subRef) setRawData(subRef.getData());
};
const refresh = () => {
  if (rafId !== null) return;
  rafId = requestAnimationFrame(flush);
};
```

Without this, a subscription update with 50 changed rows fires 50 React re-renders. With it, 50 rows = 1 re-render. The pattern took the chat module from 150-250ms message-handler violations on busy rooms to consistent sub-frame updates.

For React-only apps this is the floor of "make subscriptions usable at scale." For non-React (vanilla, Svelte, Solid), the same idea applies — coalesce subscription callbacks into per-frame batches before triggering reactive updates.

## Pattern 4: Subscription scope discipline as a performance lever

(Cross-reference to chapter 3.) Performance work on STDB usually starts at the subscription, not at the renderer. Most apps that feel slow have one of these:

- A subscription that returns 10K rows where 100 would do
- A subscription scope that didn't unsubscribe when the user navigated away
- A subscription that runs while the user can't see the data
- A subscription that runs N×M when N or M = 1 would suffice

The discipline:

- **Scope subscriptions to what the user can see right now.** Off-screen rooms don't need messages. Closed servers don't need member lists.
- **Unsubscribe on view change.** Don't carry old subscriptions forward "in case the user comes back."
- **Snapshot on demand for one-shot queries.** If you need to read a list once, use the HTTP SQL API or a one-shot subscription, not a long-lived stream.
- **Filter server-side, not client-side.** A subscription that returns 10K rows so the client can `.filter()` to 100 is wasted bandwidth. Use a `WHERE`.

Most "STDB is slow" complaints turn out to be subscription design problems. The performance ceiling of STDB itself is high. The performance ceiling of "subscribe to everything and filter in React" is low.

## Pattern 5: Avoid re-subscribing to large tables on minor state changes

A subtle React-specific footgun: `useTable` subscriptions react to dependency-array changes. If a parent component re-renders and creates a new options object, child subscriptions can churn:

```typescript
// BAD: options object identity changes every render → resubscribe every render
function MessageList({ roomId }) {
  return <MessageRows
    options={{ filter: m => m.room_id === roomId }} />;
}

// GOOD: filter is stable across renders
function MessageList({ roomId }) {
  const filter = useCallback((m) => m.room_id === roomId, [roomId]);
  return <MessageRows options={{ filter }} />;
}

// EVEN BETTER: scope at subscription, not at filter
function MessageList({ roomId }) {
  // server-side filter — only relevant rows reach the client
  return <MessageRowsForRoom roomId={roomId} />;
}
```

The third form is best. A server-side `WHERE room_id = '...'` subscription is more efficient than a wide subscription with client-side filtering, *and* it's stable across renders (the room_id is the only input).

For deeply-nested components, the rule of thumb is **subscribe at the highest stable boundary**. The component that owns "current room" subscribes to messages for that room. Children read via context or props.

## Tradeoffs and gotchas

**Hot/cold splits double the JOIN work.** Every render that needs both hot and cold data has to do a `Map.get()` per row. At high entity counts and high frame rates, that's millions of lookups per second. `Map` is fast, but profile it before splitting. Tables with under ~1000 rows usually don't need splitting at all — the wire savings are too small to justify the per-render join.

**GPU instanced rendering is a separate skill.** WebGL shader programming, buffer management, and texture atlasing have their own pitfalls. Don't reach for this pattern until simpler approaches fail. A workable instanced renderer is a few hundred lines including shader source, but those lines tend to take real iteration time — budget accordingly.

**Dirty-range tracking can lose updates if the slot map changes.** When entities enter/leave the active set, the slot map shifts. The dirty range is invalidated and you have to upload the whole buffer. Mitigation: keep entities in stable slots even after they go inactive (use a "ghost" status flag), reusing slots only when memory pressure forces it.

**rAF batching delays the *first* paint of a subscription update by up to 16ms.** Usually invisible. Occasionally it shows up as "the message I sent took 2 frames to appear." If sub-16ms latency matters to you, don't batch — but you probably don't need sub-16ms latency for chat.

**STDB doesn't expose `LIMIT` or `OFFSET` on subscriptions.** Pagination patterns from REST APIs don't translate. The closest equivalent is "subscribe to a narrower scope" — e.g., only the last 50 messages by timestamp range. STDB has open work on this; track the issue tracker.

**Subscription density matters more than reducer rate.** A user subscribed to 100 noisy tables and never calling reducers will pay more than a user subscribed to 10 quiet tables and calling reducers constantly. Performance budgeting starts at "what tables are we subscribed to" and only after that worries about reducer throughput.

**Profile under realistic load.** A demo with 5 entities tells you nothing about how the app behaves with 5000. Generate test data that matches your worst-case production volume; subscribe a single client; watch the browser's performance trace. The bottleneck is almost always not where you guessed.

## In the skill

The official [`spacetimedb-typescript` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-typescript) covers `useTable` and the `useMemo` requirement on derived state. The [`spacetimedb-concepts` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-concepts) covers the subscription update model and incremental view evaluation.

This chapter is downstream of those primitives — given you understand them, here's what they look like at the limits of what they can do.

## In the code

| Concept | File | Purpose |
|---|---|---|
| Hot/cold combined-table example | `module/src/tables/users.rs` (chat_users) | Demonstrates fields with mixed update rates |
| rAF batching | `shared/spacetimedb/createHooks.ts:159-207` | Per-frame React update coalescing |
| Server-side scope filter | `services/ChatSpacetimeDBService.ts:740-758` | Per-room message subscription |
| Wave-load deferred subscription | `services/ChatSpacetimeDBService.ts:683-712` | First-paint priority |
| Stable subscription identity | Hook usage patterns across the module | Avoid resubscribe on every render |
| GPU instanced rendering | (pattern only — no canonical impl in chat repo) | High-entity-count visualization |

---

[← Chapter 5: Durability & Consistency](./05-durability-and-consistency.md) · [Cookbook index](../README.md) · [Chapter 7: Agent & Bot Architectures →](./07-agent-and-bot-architectures.md)
