# Chapter 2 — Cross-Module Communication

> *Written against SpacetimeDB 2.0. Canonical implementations: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) `src/reducers/system.rs`, [nexus-sense](https://github.com/Spitfire-Products/nexus-terminal) `src/tables/memory.rs`.*

## The problem

SpacetimeDB modules are **isolated by design**. Each one is a separate database, has its own connections, its own identities, and no built-in way to talk to another module. This is a feature — modules are independent units of deployment and concurrency — but it pushes a non-trivial integration problem onto the application: how do two modules coordinate when one needs to act on data the other owns?

You hit this immediately in any multi-module setup. Examples from production:

- A platform module owns user accounts; a chat module needs to know when a user's tier upgrades so it can change spawn limits.
- An agent runtime module decides "the agent should post a message"; a chat module owns the messages table and the permission checks.
- A financial module computes a portfolio update; a notifications module needs to fan it out.
- Two AI agents living in different modules need a shared scratchpad to coordinate without going through a human.

There are two distinct flavors of this problem and they want different patterns:

- **Async, durable signal** — "I want module A to *eventually* learn about something that happened in module B." Latency is forgiving, but the message must not be lost.
- **Synchronous RPC** — "Module A is calling a reducer in module B and needs to know it succeeded right now." Latency-sensitive, transactional.

Mix them and you get the worst of both. Keep them distinct and each is tractable.

## Pattern 1: Bridge tables (async signal)

For async signal, build a **shared-purpose table** in one module and have everyone subscribe to it. The table is the medium; writes are events; reads are subscriptions. No module calls into another — they all just read and write a table they have a connection to.

The chat module's bridge with the agent runtime uses NexusSense's `agent_memory` table this way:

```rust
// modules/nexus-sense/spacetimedb-module/src/tables/memory.rs
#[spacetimedb::table(accessor = agent_memory, public)]
pub struct AgentMemory {
    #[primary_key]
    pub key: String,             // "bridge:{ts}:{agent}" or "agent:{id}:{topic}"
    #[index(btree)]
    pub user_id: String,
    #[index(btree)]
    pub agent_session_id: Option<String>,
    #[index(btree)]
    pub namespace: String,       // "bridge", "global", "tool", "preference", "insight"
    pub value: String,           // JSON payload
    pub ttl: Option<u64>,
    pub created_at: u64,
    pub updated_at: u64,
}
```

Two namespacing dimensions: a **namespace** field (broad category) and a **key** convention (`bridge:{timestamp}:{agent-name}`). Any module that holds a connection to nexus-sense can subscribe to `WHERE namespace = 'bridge'` and read every message; agents write to it via `set_agent_memory(key, "bridge", value, null, null)`.

The reducer is plain CRUD with one critical guard:

```rust
// modules/nexus-sense/spacetimedb-module/src/reducers/memory.rs
#[spacetimedb::reducer]
pub fn set_agent_memory(ctx: &ReducerContext, key: String, namespace: String,
                       value: String, agent_session_id: Option<String>, ttl: Option<u64>) {
    let Some(user_id) = get_caller_user_id(ctx) else {
        log::warn!("[set_agent_memory] Unauthorized: no identity link");
        return;
    };

    let created_at = if let Some(existing) = ctx.db.agent_memory().key().find(&key) {
        if existing.user_id != user_id {
            log::warn!("[set_agent_memory] Unauthorized: not owner of key {}", key);
            return;
        }
        // ...
    };
    // ... insert/upsert
}
```

The `existing.user_id != user_id` check enforces **writer ownership** — once a key is claimed by a user, only that user can update it. Crucial for a public bridge table: anyone can write *new* keys, but no one can clobber someone else's. Combined with a key naming convention that includes the writer's identifier (`bridge:{ts}:{agent}`), this gives you a multi-tenant message bus on a single table.

Reads are unfiltered subscription:

```typescript
conn.subscriptionBuilder()
  .onApplied(handleBridgeUpdate)
  .subscribe([`SELECT * FROM agent_memory WHERE namespace = 'bridge'`]);
```

### When to use bridge tables

- The receiving side is comfortable polling/subscribing rather than getting a callback
- Messages are small (string-encoded JSON works fine; don't try to put 10MB blobs in)
- You want durability — the bridge survives module restarts because it's just data
- TTL handles cleanup naturally (the schema already has it)

### When **not** to use bridge tables

- Latency matters and a 50-200ms subscription update is too slow
- You need transactional semantics (write succeeds *and* receiver reacts atomically)
- You need the receiver to refuse the message (back-pressure) — bridge tables are fire-and-forget

For those cases, you want pattern 2.

## Pattern 2: HTTP-from-procedure RPC (synchronous, authenticated)

When the chat module needs an agent runtime to *act* on its behalf — "post this message as bot X in room Y, right now, and tell me if it failed" — bridge tables are the wrong shape. You want a real RPC. SpacetimeDB doesn't ship one, but it ships everything you need:

1. **HTTP reducer API** — every reducer can be called from outside the module via `POST /v1/database/{module}/call/{reducer}`
2. **Procedures** (Rust 2.0+) can make outbound HTTP calls
3. **Private tables** (no `public` keyword) can hold secrets that never reach a client

Combine those three and you have authenticated cross-module RPC:

```rust
// commands/standalone/CHAT/spacetimedb-module/src/reducers/system.rs
#[spacetimedb::reducer]
pub fn system_send_bot_message(
    ctx: &ReducerContext,
    bot_user_id: String,
    room_id: String,
    content: String,
    message_id: String,
    system_token: String,    // ← shared secret
) {
    // 1. Validate auth: system_token OR platform admin identity
    let has_valid_token = if !system_token.is_empty() {
        let expected = ctx.db.system_config().key().find(&"cortex_system_token".to_string());
        matches!(expected, Some(config) if config.value == system_token)
    } else { false };
    let is_admin = crate::utils::auth::is_platform_admin(ctx);

    if !has_valid_token && !is_admin {
        log::warn!("[system_send_bot_message] Rejected: invalid token and not admin");
        return;
    }

    // 2. Validate bot_user_id is a real bot
    let bot = match ctx.db.chat_users().user_id().find(&bot_user_id) {
        Some(user) if user.is_bot == Some(true) => user,
        _ => { log::warn!("[system_send_bot_message] Not a bot"); return; }
    };

    // 3. Validate bot is a member of the target room (no impersonation)
    let is_member = ctx.db.room_members().iter().any(|m| {
        m.room_id == room_id && m.user_id == bot_user_id && m.role != "banned"
    });
    if !is_member { return; }

    // 4. Insert with is_bot_author = true (unforgeable)
    ctx.db.messages().insert(Message { /* ... */ is_bot_author: Some(true) });
}
```

The calling module (in our case, an agent runtime procedure in `nexus-cortex`) does:

```rust
let response = http_client.post(format!(
    "{}/v1/database/nexus-chat/call/system_send_bot_message",
    chat_api_base
))
.json(&serde_json::json!({
    "bot_user_id": bot_id,
    "room_id": room_id,
    "content": content,
    "message_id": format!("cortex-tool-{}-{}", bot_id, ts),
    "system_token": chat_system_token,  // from this module's own private config
}))
.send()?;
```

Two tables make this work: `system_config` (private, never `public`) holds the shared secret keyed by name, and the `system_send_bot_message` reducer validates the secret against that table on every call.

### The security model

This is the same shape as a typical microservice setup with a shared API token. The threat model:

- **Browsers can never trigger this reducer with a valid token.** The token never leaves server-side storage. A malicious client can call the reducer (it's public), but they can't pass a valid `system_token`, so `has_valid_token = false` and the call is refused unless they happen to *also* be a platform admin.
- **Platform admins can call it directly** (admin identity bypasses the token check). This is by design — it makes the reducer testable from the CLI and gives admins an escape hatch if the calling module is broken.
- **Cross-module HTTP calls** present an anonymous identity to the receiving module. Without the token, `is_platform_admin` is always false (no identity link to look up), so the call would be refused. The token is the *only* way an external module proves it's authorized.

### Bot membership as a permission

The reducer also validates the bot is a member of the target room (`ctx.db.room_members().iter().any(...)`). This stops a compromised agent runtime from posting as bots into rooms those bots aren't in. Authorization is **per-action**, not per-call: even with a valid token, you can only do what the data model says you can do.

### When to use HTTP RPC

- The caller needs a synchronous result (success/failure, returned data)
- The action has security implications and needs reducer-level authorization
- Cross-module call is rare-ish (single-digit per second, not thousands)

### When **not** to use HTTP RPC

- High-frequency events — HTTP overhead per call is real, and the receiving module's reducer queue can become a bottleneck. Use a bridge table and subscribe.
- Public clients should also call the same path. Then it's not RPC, it's just a reducer; let them call it directly via WebSocket.

## Combining the two patterns

The patterns compose. The chat module's agent integration uses both:

- The agent runtime *posts a message* via HTTP RPC (`system_send_bot_message`) because it needs synchronous failure semantics ("the bot can't reply if the message didn't insert").
- The agent runtime *learns about new mentions* via a bridge-style subscription on the `agent_task` table because the latency is forgiving and we want durability across runtime restarts.

Don't try to make one pattern do both jobs. The boundary between "I want a reply" and "I want a notification" is the boundary between RPC and bridge.

## Tradeoffs and gotchas

**HTTP from procedures requires a network round-trip and JSON serialization.** Inside-module calls to other reducers are a function call. Cross-module HTTP is hundreds of times more expensive. For chat where a bot replies once every few seconds, that's invisible. For an inner-loop hot path, it's a regression.

**The `system_config` table absolutely must be private.** Forgetting `public` is the right default; *adding* `public` accidentally exposes the token. Code review for `public` flags on any table holding secrets is non-negotiable. If you're worried about this drifting, a unit test that asserts `system_config` is non-public is cheap insurance.

**Token rotation is manual.** Update both ends, then update the `system_config` row in the receiver. There's no graceful overlap window. We've worked around this by allowing platform-admin identity as a fallback authorization — even if the token is wrong, an admin call still works, so the system isn't fully bricked while you fix it. Worth designing in.

**Bridge tables can grow unbounded.** TTL helps, but if you're not using TTL or you're not reaping expired rows, they'll bloat. Either use `agent_session_id` to scope writes per-session and clean up on session end, or schedule a cleanup reducer.

**Subscription updates have ~50-200ms latency.** Bridge tables are *not* a low-latency channel. If you need sub-50ms cross-module reaction (e.g., trading systems where a price update has to fire an alert in another module immediately), you need a different architecture entirely — probably "stop using two modules."

**Webhook out of STDB is a third pattern we've avoided.** SpacetimeDB doesn't have native webhooks; you can build one with a procedure that POSTs on every reducer call, but the operational cost (retry, dead-letter, monitoring) is bad. Stick to bridge tables and HTTP RPC.

## In the skill

The official [`spacetimedb-rust` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-rust) covers procedures, scheduled reducers, and the table macro. The HTTP reducer API is documented in [`spacetimedb-cli`](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-cli) under `spacetime call`. This chapter shows how to compose them into a multi-module integration pattern.

## In the code

| Concept | File | Purpose |
|---|---|---|
| Bridge table schema | `modules/nexus-sense/.../tables/memory.rs` | `agent_memory`, namespaced KV |
| Bridge writer guard | `modules/nexus-sense/.../reducers/memory.rs` (lines 24-28) | First-writer-owns enforcement |
| HTTP RPC reducer | `module/src/reducers/system.rs` (lines 67-154) | `system_send_bot_message` |
| Private secret table | `module/src/tables/system_config.rs` | Token storage (never `public`) |
| Token validation pattern | `module/src/reducers/system.rs` (lines 76-82) | Constant-time-style match |
| Cross-action permission check | `module/src/reducers/system.rs` (lines 99-108) | Bot must be room member |

---

[← Chapter 1: Identity & Multi-Device Auth](./01-identity-and-multi-device-auth.md) · [Cookbook index](../README.md) · [Chapter 3: Subscription Engineering →](./03-subscription-engineering.md)
