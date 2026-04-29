# Chapter 3 — Subscription Engineering

> *Written against SpacetimeDB 2.0/2.1. Canonical implementation: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) `services/ChatSpacetimeDBService.ts` and `shared/spacetimedb/createHooks.ts`.*

## The problem

A SpacetimeDB subscription is the only way clients learn about table changes in real time. It's also the easiest thing to get wrong. The naïve approach — "subscribe to everything, render whatever" — works fine on a 5-row demo and falls over hard on a 40-table chat schema with 10K messages, 50 servers, and 1000 concurrent clients.

The constraints SpacetimeDB imposes (some by design, some by current implementation):

- **WHERE clauses only support `=`.** No `!=`, `<`, `>`, `LIKE`, `IN (...)`. Trying to use any of those crashes the subscription. ([STDB issue tracker](https://github.com/clockworklabs/SpacetimeDB/issues) has open work to extend this.)
- **`Option<T>` columns can't be filtered server-side.** Subscriptions can't express `WHERE col IS NULL` or `WHERE col = 'x'` against an optional column.
- **One bad query in a `subscribe([...])` batch fails the whole batch.** Silent data missing rather than loud error.
- **Per-subscription overhead scales nonlinearly** at higher counts. Adding 4 queries × N items in a single `subscribe()` call costs more than 4 separate subscriptions.
- **Subscription updates fire callbacks for every changed row**, which fires React re-renders, which on hot tables (chat messages, presence) becomes a 150-250ms message-handler violation in the browser.

Each of these has a workaround. Together they amount to a small discipline that turns "real-time chat with thousands of rows" from a 30-second initial-load slog into a sub-second connect.

## Pattern 1: Scope subscriptions narrowly

Don't subscribe to a table — subscribe to a **slice of a table that the user is currently looking at**. The chat module uses four scopes:

```typescript
// commands/standalone/CHAT/services/ChatSpacetimeDBService.ts
// On connect (authenticated user)
const sub1 = wave1.subscribe([
  'SELECT * FROM chat_users',
  'SELECT * FROM rooms',
  'SELECT * FROM chat_servers',
  `SELECT * FROM server_members WHERE user_id = '${userId}'`,
  `SELECT * FROM room_members WHERE user_id = '${userId}'`,
  `SELECT * FROM starred_channels WHERE user_id = '${userId}'`,
]);

// On room select
return builder.subscribe([
  `SELECT * FROM messages WHERE room_id = '${roomId}'`,
  `SELECT * FROM room_members WHERE room_id = '${roomId}'`,
  `SELECT * FROM typing_indicators WHERE room_id = '${roomId}'`,
  `SELECT * FROM reaction_room_index WHERE room_id = '${roomId}'`,
  `SELECT * FROM message_edit_room_index WHERE room_id = '${roomId}'`,
  // ...
]);

// On server select
const queries = [
  `SELECT * FROM channel_categories WHERE server_id = '${serverId}'`,
  `SELECT * FROM server_roles WHERE server_id = '${serverId}'`,
  // ...
];
```

The four scopes — **core**, **room**, **server**, **public-guest** — each have a clear lifecycle:

| Scope | Subscribed when | Unsubscribed when |
|---|---|---|
| Core | User authenticates | User logs out |
| Room | User opens a channel | User leaves it (or switches) |
| Server | User opens a server | User switches servers |
| Public-guest | Unauthenticated visit | Authentication completes |

This is *not* "the simpler the schema, the simpler the subscription." Chat has 40 tables. The scoping discipline is what keeps the data per-client manageable. **A new user's first connect should only pull data they can see right now**, not "everything in every server they've ever joined."

### The `_` reserved subscription rule

If you're new to STDB you might think you can do `WHERE server_id IN ('a', 'b', 'c')` to subscribe to several servers at once. You can't. The subscription parser rejects `IN`. You write three subscriptions, or you let the user-facing UX dictate "one server active at a time" and switch.

The chat module went the second way: only one server is "open" at a time per client. Switching un-subscribes the old server scope and subscribes the new. This is a *user experience* decision (Discord works the same way) that lines up nicely with the constraint.

## Pattern 2: Wave-load deferred tables

When a user connects, you want them to see the most-important data first and the rest can wait a frame or two. The chat module does this with two subscription waves:

```typescript
// Wave 1: Core (everything the first paint needs)
const sub1 = wave1.subscribe([
  'SELECT * FROM chat_users',
  'SELECT * FROM rooms',
  'SELECT * FROM chat_servers',
  // ... own memberships
]);

// Wave 2: Deferred — load after first render completes
setTimeout(() => {
  conn.subscriptionBuilder()
    .subscribe([
      'SELECT * FROM user_profiles',
      `SELECT * FROM room_invitations WHERE invitee_id = '${userId}'`,
      `SELECT * FROM drafts WHERE user_id = '${userId}'`,
      `SELECT * FROM read_positions WHERE user_id = '${userId}'`,
      `SELECT * FROM scheduled_messages WHERE author_id = '${userId}'`,
      `SELECT * FROM bookmarks WHERE user_id = '${userId}'`,
      // ...
    ]);
}, 500);
```

A 500ms delay isn't tuned to perfection — it's "long enough to let React commit the first render and the user to see something." Wave 2 is data that's only needed when the user takes a specific action (open the drafts panel, view bookmarks). Loading it in wave 1 doubles the initial subscription apply time for no UX benefit.

This isn't a pattern STDB requires. It's a UX pattern that *exploits* how STDB subscriptions work: each subscription is independent, they can apply at different times, and the client can react to each `onApplied` callback separately.

## Pattern 3: Lazy modal subscriptions

Some data is rarely needed but heavy when it is. Edit history is a perfect example: most users never click "view edit history," but those who do want every revision back to the original. Subscribing to `message_edits` globally is wasteful; subscribing to it never means the modal can't render.

The fix: subscribe **on mount of the modal**, unsubscribe on close.

```typescript
// commands/standalone/CHAT/components/EditHistoryModal.tsx
useEffect(() => {
  const svc = getChatSpacetimeDBService();
  const conn = (svc as any).getConnection();
  if (!conn) return;

  let cancelled = false;
  const handle = conn.subscriptionBuilder()
    .onApplied(() => {
      if (cancelled) return;
      const rows = [];
      for (const e of conn.db.message_edits.iter()) {
        if (e.message_id !== messageId) continue;
        rows.push(e);
      }
      setFullEdits(rows);
    });
  const sub = handle
    .onError((_c, err) => console.warn('subscription error', err))
    .subscribe([`SELECT * FROM message_edits WHERE message_id = '${messageId}'`]);

  return () => {
    cancelled = true;
    try { sub.unsubscribeThen?.(() => undefined) ?? sub.unsubscribe?.(); } catch {}
  };
}, [messageId]);
```

The pattern has a subtlety worth naming: **fall back to a less-rich source while the lazy subscription is loading.** In the chat module, `messageEdits` derived from `message_edit_room_index` (already in scope from the room subscription) renders a placeholder list with `oldContent: ''`. When the lazy subscription apply lands, the modal swaps in the full edits with `oldContent` populated. The user never sees a blank modal.

This is "two-tier data": cheap data is always present, expensive data loads on demand, and the UI gracefully degrades between them. The same pattern applies to: pinned messages (cheap pin index → expensive full message rehydrate), member lists (cheap membership rows → expensive profile fetch), thread histories.

## Pattern 4: rAF-batched table updates

SpacetimeDB's insert/update/delete callbacks fire **per row**. A subscription update with 50 inserted rows fires 50 callbacks. If each callback triggers `setState` in React, you've fired 50 re-renders for one logical event.

The fix: coalesce callbacks within a frame using `requestAnimationFrame`.

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

const subOptions = {
  onInitial: (data) => { setRawData(data); setLoading(false); },
  onInsert: (row) => { refresh(); onInsert?.(row); },
  onUpdate: (oldRow, newRow) => { refresh(); onUpdate?.(oldRow, newRow); },
  onDelete: (row) => { refresh(); onDelete?.(row); },
};
```

The mechanism: any callback (insert, update, delete) schedules a single frame-aligned re-read of the table cache. Subsequent callbacks within the same frame are no-ops because `rafId` is already set. When the frame fires, ONE `setRawData` call delivers the whole batch.

This change alone took the chat module from 150-250ms message-handler violations on busy rooms to consistent sub-frame updates. The complexity cost is one helper function shared across all hooks; the benefit is ~10-20× re-render reduction on hot tables.

### Why re-read instead of incrementally update?

Subtle but important: the pattern calls `subRef.getData()` to re-read the *full* table cache, not "apply this row to local state." Two reasons:

1. SpacetimeDB's callback row references aren't guaranteed to be the same objects you got from `iter()`. Reference-equality filter/map can miss; re-reading via the SDK's cache is always correct.
2. Coalescing callback payloads correctly under reorderings (insert+delete of same row, update across delete) is tricky and easy to get wrong. Re-reading is dumber and more reliable.

## Pattern 5: Filter limitations and the `Option<T>` workaround

Two STDB constraints regularly bite you in the same place:

1. **Subscriptions can only filter by `=` in WHERE clauses.** No `!=`, `<>`, `<`, `>`, `IN`, `LIKE`. The parser rejects them outright.
2. **`Option<T>` columns can't be filtered server-side at all.** You can't write `WHERE col = 'x'` against an `Option<String>`; the subscription apply errors.

Both bite when you have a column that *sometimes* has a value and you want to scope the subscription to only the rows where it's set. Common case: a `room_id: Option<String>` column on a `reactions` table — DM reactions have no room, server reactions do, and clients only want server reactions for the current server.

You can't filter the source table. The workaround is a **shadow index table** that mirrors only the rows where the optional column is set:

```rust
#[spacetimedb::table(accessor = reaction_room_index, public)]
pub struct ReactionRoomIndex {
    #[primary_key]
    pub reaction_id: String,
    #[index(btree)]
    pub room_id: String,         // ← String, not Option<String>
    pub message_id: String,
    pub user_id: String,
    pub emoji: String,
    pub created_at: u64,
}
```

The reducer that writes the source `reactions` table dual-writes the index when `room_id` is `Some`, and skips the write when it's `None`. Clients that only need server-scoped reactions subscribe to `reaction_room_index WHERE room_id = '...'` — which works because the column is now non-optional.

This pattern is covered in depth in chapter 4 (it's also a schema-discipline pattern). The relevant subscription mechanic: **denormalize columns to make them filterable**, accept the dual-write cost in exchange for a bounded subscription. The chat module saved an order of magnitude on subscription bandwidth using this workaround for two source tables (`reactions` and `message_edits`).

## Tradeoffs and gotchas

**One bad query fails the whole batch.** If your `subscribe([q1, q2, q3])` array has a syntax error in `q2`, the whole subscription apply fails silently — `onApplied` doesn't fire and you wonder why no data is showing up. The dev-time fix is `onError` callbacks on every subscription. The production fix is to never construct queries with user-supplied values without sanitization (single-quote escapes, etc.).

**Per-subscription overhead is real and nonlinear.** We learned this when a Phase 4 deploy added 2 queries × N DM rooms to a single `subscribe()` call and server-switching latency went from 1s to 30-45s. The fix wasn't "remove the queries" — it was "don't add them to a high-fanout `subscribe()`." Active-room subscriptions can carry the cost; background subscriptions can't.

**`onApplied` is not "all data has arrived."** It means "this batch of subscription queries has applied." Data from *another* subscription (a different `subscribe()` call) may still be in flight. UI code should not assume the whole world is loaded after one `onApplied` — gate render on the subscription that owns the data.

**Subscription cleanup matters.** A modal that subscribes on mount must unsubscribe on unmount. Repeated subscribe-without-unsubscribe leaks subscriptions; we've seen sessions accumulate 50+ ghost subscriptions over a long session. The pattern in this chapter (`return () => sub.unsubscribe()` in `useEffect`) is the contract; enforce it via lint or convention.

**Don't use `_trackContent: true` (cache mode) reflexively.** The scan tool in NexusSense has three cache modes; the *content-aware* mode is a token-saving feature for AI agents reading text-heavy elements. For UI subscription state, the structural cache is sufficient. Same principle applies to STDB subscriptions: don't ask for more invalidation granularity than you need.

**Reducer-call frequency is independent of subscription density.** A user with 100 active subscriptions doesn't pay more per reducer call than a user with 1. Subscription cost is overhead at apply time and update-fanout time; reducer cost is unrelated.

## In the skill

The official [`spacetimedb-typescript` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-typescript) covers `subscriptionBuilder()`, the `onInsert`/`onUpdate`/`onDelete` callback contract, and the `withToken()` connection pattern. The [`spacetimedb-concepts` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-concepts) covers the subscription model and update propagation.

Anti-pattern flags in the skills: don't use `useTable` without `useMemo` on derived state (creates new array references and triggers cascading re-renders); don't use React StrictMode with WebSocket connections (double-invokes effects, causing connect-disconnect loops); avoid optimistic UI state shadows (let subscription updates be the single source of truth).

## In the code

| Concept | File | Purpose |
|---|---|---|
| Four-scope subscription strategy | `services/ChatSpacetimeDBService.ts:670+` | Core / room / server / public |
| Wave-load split | `services/ChatSpacetimeDBService.ts:683-712` | Wave 2 deferred tables |
| Lazy modal subscription | `components/EditHistoryModal.tsx:29-74` | On-mount subscribe pattern |
| rAF batching | `shared/spacetimedb/createHooks.ts:159-207` | Per-frame flush |
| Shadow index workaround | `module/src/tables/reaction_room_index.rs` | `Option<T>` denormalization |
| Subscription error handler | `services/ChatSpacetimeDBService.ts` (every `onError`) | Don't fail silently |

---

[← Chapter 2: Cross-Module Communication](./02-cross-module-communication.md) · [Cookbook index](../README.md) · [Chapter 4: Schema Discipline →](./04-schema-discipline.md)
