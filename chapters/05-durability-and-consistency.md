# Chapter 5 — Durability & Consistency

> *Written against SpacetimeDB 2.0/2.1 (Confirmed Reads requires SDK ≥ 1.4.0). Canonical implementation: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) reaction/edit dual-write reducers and the financial module's connection config.*

## The problem

Every database has a moment between "the application thinks the write succeeded" and "the data is durable on disk." Crash in that window and the write is lost. SpacetimeDB has the same window, with two specific gotchas you'll hit before you hit it elsewhere:

1. **Default subscription updates fire after in-memory commit, not after `fsync`.** The reducer call returns success, the subscription update flows to clients, and clients re-render their UI showing the new state. If the module crashes before `fsync`, the data is gone — but the clients have already shown it. Stale UI, lost data, unhappy users.
2. **Shadow indexes (chapter 4) introduce a "two writes, two truths" risk.** If your source-table write succeeds and the shadow-index write fails, your subscription readers see one truth and your authoritative writers see another. Without atomicity, this is a maintenance nightmare.

Both have first-class fixes in STDB. Most apps default to the wrong one because the defaults optimize for latency. Production financial-grade workloads need the durable defaults; chat-grade workloads can stay fast.

## Pattern 1: Confirmed Reads for crash-safe subscriptions

`Confirmed Reads` is a connection-level flag (SDK ≥ 1.4.0) that delays subscription updates until the underlying transaction is `fsync`'d to disk (or replica-acknowledged in a cluster).

```typescript
// modules/financial/services/FinancialSpacetimeDBService.ts
const conn = DbConnection.builder()
  .withUri('wss://maincloud.spacetimedb.com')
  .withModuleName('nexus-financial')
  .withConfirmedReads(true)    // ← wait for fsync before delivering updates
  .build();
```

The tradeoff is binary:

- **`true`** — subscription updates land on clients only after the server has durably committed. Latency increases (typically 5-50ms additional for a one-shot transaction; more under load). The client is never shown data that could be lost in a crash.
- **`false` / unset (default)** — subscription updates land after in-memory commit. Faster, but a server crash within the commit-to-fsync window can lose data clients have already seen.

The right answer depends on what your data is worth and how often you crash. Some heuristics:

- **Financial transactions, billing, audit logs, anything legally/contractually durable** → `withConfirmedReads(true)`. Always. The latency is invisible to humans; the durability bug under load is real.
- **Real-time messaging, presence, typing indicators, cursor positions** → leave it default. The data is ephemeral or easily re-sent. Speed matters more than durability.
- **User-edited content (drafts, profile changes, settings)** → mixed. The connection is per-application, so you can have one module on `withConfirmedReads(true)` and another on default. Pick per-module.

The chat module uses defaults (it's chat). The financial module uses confirmed reads (it's money). Both are correct for their domain.

### Why per-connection rather than per-write

Some databases let you specify durability per-write (`SYNC` vs `ASYNC` writes in MySQL, etc.). STDB makes the choice at connection setup. The reasoning, if you accept it: you don't actually want different durability for different operations on the same connection — that creates ordering anomalies where a high-durability write can be observed *after* a low-durability write that was logically earlier. Connection-level keeps the ordering invariant clean.

The cost: if you have one app with both ephemeral and durable concerns, you maintain two connections. We do this in nexus-terminal — one connection to nexus-financial (confirmed reads) and a separate one to nexus-chat (default).

## Pattern 2: Atomic dual-write within a single reducer

When you maintain a shadow index (chapter 4) or a denormalized mirror, every write to the source must also write the index. The atomicity guarantee comes free if you do both writes inside one reducer:

```rust
// commands/standalone/CHAT/spacetimedb-module/src/reducers/reactions.rs (excerpt)
#[spacetimedb::reducer]
pub fn toggle_reaction(ctx: &ReducerContext, message_id: String, emoji: String) {
    let user_id = match get_caller_user_id(ctx) {
        Some(u) => u,
        None => return,
    };

    let reaction_id = format!("{}:{}:{}", user_id, message_id, emoji);
    let now = timestamp_ms(ctx);

    // Look up the message's room_id (Option<String>)
    let Some(message) = ctx.db.messages().id().find(&message_id) else { return; };
    let room_id = message.room_id;

    // Check for existing reaction (toggle off)
    if let Some(existing) = ctx.db.reactions().id().find(&reaction_id) {
        // Source: delete
        ctx.db.reactions().id().delete(&existing.id);
        // Shadow: also delete (only if it had a room_id)
        if existing.room_id.is_some() {
            ctx.db.reaction_room_index().reaction_id().delete(&reaction_id);
        }
        return;
    }

    // Source: insert
    ctx.db.reactions().insert(Reaction {
        id: reaction_id.clone(),
        room_id: room_id.clone(),
        message_id: message_id.clone(),
        user_id: user_id.clone(),
        emoji: emoji.clone(),
        created_at: now,
    });

    // Shadow: insert (only when room_id is Some)
    if let Some(rid) = room_id {
        ctx.db.reaction_room_index().insert(ReactionRoomIndex {
            reaction_id, room_id: rid, message_id, user_id, emoji, created_at: now,
        });
    }
}
```

STDB reducers are atomic within their scope: the whole reducer body runs as one transaction, and either all writes commit together or none do. **You don't need explicit transaction syntax.** If the reducer panics or returns mid-flight, all writes roll back automatically.

This is the easiest part of STDB to take for granted. Most real databases require you to remember `BEGIN`/`COMMIT`. STDB doesn't — but the *implication* is that your dual-writes have to be in the same reducer call. Splitting them across two reducers (e.g., "first call inserts source, then a follow-up call inserts the shadow") loses atomicity.

### The cascade-delete sibling rule

Every dual-write reducer has a corresponding cascade-delete obligation. When the source table's row is deleted, the shadow index row must also be deleted in the same reducer:

```rust
// In the message-delete reducer
pub fn delete_message(ctx: &ReducerContext, message_id: String) {
    // ... auth checks ...

    // Cascade: delete reactions on this message
    let reaction_ids: Vec<String> = ctx.db.reactions().iter()
        .filter(|r| r.message_id == message_id)
        .map(|r| r.id.clone())
        .collect();

    for rid in &reaction_ids {
        ctx.db.reactions().id().delete(rid);
        // Also delete from shadow — atomic with the source delete
        ctx.db.reaction_room_index().reaction_id().delete(rid);
    }

    // ... delete the message itself, edits, attachments, etc.
}
```

The shadow-index inconsistency that bites worst is **orphaned shadow rows** — source row gone, shadow row still there. Subscriptions to the shadow show a row that doesn't have an authoritative source, downstream UI renders something nonsensical, support tickets open. The fix is to put the shadow-delete next to every source-delete in the same reducer. There is no audit reducer that catches this later — the discipline is at write-time, not check-time.

## Pattern 3: Reducer-level idempotency

STDB delivers exactly-once semantics within a single reducer call: the reducer body either commits fully or rolls back fully. But STDB does **not** deduplicate retries from the client side. If a client gets a network blip after sending a `send_message` reducer but before getting the ack, the SDK may retry the call. Without idempotency, the user sees "their message twice."

The pattern: **client-supplied IDs.** The client generates a UUID before the call and passes it as the message_id; the reducer uses it as the primary key. A retry produces the same primary key, the second insert fails (or is a no-op), and only one message exists.

```rust
#[spacetimedb::reducer]
pub fn send_message(ctx: &ReducerContext, room_id: String, content: String, message_id: String) {
    // Use the client-supplied message_id as PK
    if ctx.db.messages().id().find(&message_id).is_some() {
        // Idempotent retry — already inserted, return success silently
        return;
    }
    ctx.db.messages().insert(Message {
        id: message_id, /* ... */
    });
}
```

Two pieces have to align for this to work:

1. **Client generates the ID before the call.** Don't let the server generate it, because then a retry can't be detected.
2. **Reducer treats existing-ID as success, not error.** Logging a warning on retry is fine; refusing the retry is bad — the original commit may have actually failed and the retry is the *real* delivery.

Combined with STDB's atomic-reducer guarantee, this gives you exactly-once at the application layer with sane semantics. Without client IDs, you have at-most-once or at-least-once but not exactly-once.

## Pattern 4: Structured error logging instead of returning `Result`

Reducers in STDB don't have a transactional rollback when they "fail" with a returned error — they just return early. The convention we've found that works best:

```rust
pub fn send_message(ctx: &ReducerContext, /* ... */) {
    let Some(user_id) = get_caller_user_id(ctx) else {
        log::warn!("[send_message] Unauthorized");
        return;
    };
    let Some(room) = ctx.db.rooms().id().find(&room_id) else {
        log::warn!("[send_message] Room {} not found", room_id);
        return;
    };
    // ... actual write
}
```

Every refusal logs at `warn` with the reducer name in brackets. Log aggregation in production (`spacetime logs nexus-chat -f`) makes refusals greppable. Without the prefix, finding "why did this reducer refuse to commit" requires correlating logs by line number.

A failure that the client should know about is harder. STDB's reducer signature is `fn(...) -> ()` — there's no error return. The client knows the reducer returned, but it can't distinguish "wrote successfully" from "logged a warning and returned early." Workarounds:

- **Subscription-driven feedback.** The client watches its own writes via subscription. If the message it expects doesn't appear, treat it as a failure after a timeout.
- **Status table.** Reducer writes to a per-user status table on failure; client subscribes and reads.
- **Don't fail silently.** Refuse the call client-side first if you can — most validation can run in TypeScript before the reducer is invoked.

This is one of the genuinely awkward parts of STDB and we don't have a great answer. The patterns above are workarounds; a future "reducer Result" extension would be cleaner.

## Tradeoffs and gotchas

**Confirmed Reads has nonzero cost under load.** A single-shot transaction is fine. A burst of 1000 transactions/second with confirmed reads will start to see the `fsync` queue build up. Watch latency before turning it on for high-throughput tables.

**Dual-write atomicity only holds within one reducer.** If you split your write across reducer calls (e.g., "insert source in reducer A, schedule reducer B to insert shadow"), atomicity is lost. There's no STDB equivalent of "begin transaction across reducer calls." If you find yourself wanting one, your reducer is too granular — combine them.

**STDB does not protect you from logic errors.** "Atomic" means "all-or-nothing." It doesn't mean "correct." A reducer that decrements user A's balance and increments user B's balance is atomic; a reducer that decrements user A's balance and forgets to increment user B's balance is also atomic (and broken). Reducers are still application code; they still need code review.

**Confirmed Reads is per-connection, not per-subscription.** You can't have one subscription on a single connection use confirmed reads and another use defaults. If you need both behaviors against the same module, run two connections (one with `withConfirmedReads(true)`, one without).

**The dual-write pattern doubles your write amplification.** Two table inserts per logical write. For low-write tables (reactions, message edits) this is fine; for high-write tables (messages, presence) it's worth measuring before adopting. Sometimes you just want a simpler subscription scope (chapter 3) and not a shadow index.

**`spacetime logs` defaults to non-following.** Use `-f` to tail. The lookup pattern `[reducer_name] Rejected: reason` is the convention; without the prefix, log filtering during incidents is brutal.

**No built-in dead-letter queue.** A reducer that always fails (panics, asserts, or refuses) has no automatic retry or DLQ in STDB. Failed reducer calls just don't happen. If you need DLQ semantics, you build it yourself with a "pending" table and a scheduled cleanup.

## In the skill

The official [`spacetimedb-typescript` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-typescript) covers the connection builder and `withConfirmedReads(true)`. The [`spacetimedb-rust` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-rust) covers reducer atomicity and the `ctx.db.<table>()` insert/delete contract. The [`spacetimedb-concepts` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-concepts) covers the durability model.

This chapter is *operational* on top of those primitives — when to use confirmed reads, how to keep dual-writes atomic, how to design idempotency. The skills tell you the API; this chapter tells you the policy.

## In the code

| Concept | File | Purpose |
|---|---|---|
| Confirmed Reads enabled | `modules/financial/services/FinancialSpacetimeDBService.ts` | Durability for financial data |
| Confirmed Reads default | `commands/standalone/CHAT/services/ChatSpacetimeDBService.ts` | Speed for chat |
| Atomic dual-write | `module/src/reducers/reactions.rs:62-99` | Source + shadow in one reducer |
| Cascade-delete dual-write | `module/src/reducers/messages.rs:393-415` | Delete both tables atomically |
| Client-supplied message_id | `module/src/reducers/messages.rs` (insert path) | Idempotency via PK |
| Structured refusal logging | Every reducer | Greppable rejection traces |

---

[← Chapter 4: Schema Discipline](./04-schema-discipline.md) · [Cookbook index](../README.md) · [Chapter 6: Performance Patterns →](./06-performance-patterns.md)
