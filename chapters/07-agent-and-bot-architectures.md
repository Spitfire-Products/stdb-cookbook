# Chapter 7 — Agent & Bot Architectures

> *Written against SpacetimeDB 2.0/2.1. Canonical implementations: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) `services/AgentChatService.ts`, `module/src/reducers/agents.rs`, `module/src/tables/agent_credentials.rs`.*

## The problem

The moment your real-time app needs AI agents, bots, NPCs, or any kind of automated participant, you hit a question SpacetimeDB doesn't answer for you: **what does it mean for a bot to "be a user"?**

Naïve approaches fail in characteristic ways:

- **"The bot is a server-side process that calls reducers as a special user_id"** — works for simple cases, breaks the moment you need real-time presence (the bot has no STDB connection, can't be subscribed to typing indicators or member lists), and creates a massive trust boundary (the server can post as anyone).
- **"The bot piggybacks on its owner's connection"** — the bot's actions look like its owner's. Permissions are wrong. Subscriptions are wrong. Multi-bot systems share a connection's identity-link slot.
- **"The bot connects as the human owner with their token"** — security disaster. Bot can do anything the user can.

The pattern that actually works treats bots as **first-class STDB clients with their own identity, their own subscriptions, and their own connection lifecycle** — but with a controlled provisioning path that prevents anyone from making a bot impersonate someone else.

## Pattern 1: Bots are users with their own STDB connections

Architecturally, a bot is **a `chat_users` row with `is_bot = true` and its own STDB connection**. It connects to the same module the human users connect to, registers its own identity link, and calls reducers on its own behalf.

```rust
// commands/standalone/CHAT/spacetimedb-module/src/reducers/agents.rs
#[spacetimedb::reducer]
pub fn provision_agent(
    ctx: &ReducerContext,
    agent_user_id: String,
    display_name: String,
    owner_user_id: String,
    is_sub_agent: bool,
) {
    let Some(caller_user_id) = get_caller_user_id(ctx) else {
        log::warn!("[provision_agent] Unauthorized: no identity link");
        return;
    };

    // Spawn-permission check (admin or owner with quota)
    if is_sub_agent {
        // Sub-agents can't spawn sub-agents
        if /* depth check */ {
            log::warn!("[provision_agent] Rejected: max depth = 1");
            return;
        }
    } else if !is_platform_admin(ctx) {
        log::warn!("[provision_agent] Rejected: not platform admin");
        return;
    }

    // Spawn-quota check by tier
    if active_count >= max_agents {
        log::warn!("[provision_agent] Spawn limit {}/{} reached for tier {}",
                   active_count, max_agents, owner_tier);
        return;
    }

    // Insert chat_users row with is_bot = true
    ctx.db.chat_users().insert(ChatUser {
        user_id: agent_user_id.clone(),
        stdb_identity: String::new(), // ← empty — set by register_agent_identity
        display_name,
        is_bot: Some(true),
        bot_owner_user_id: Some(owner_user_id),
        // ... other fields
    });

    // Generate and store an owner_secret — the credential the bot's
    // future connection uses to claim this user_id
    let owner_secret = generate_random_secret();
    ctx.db.agent_credentials().insert(AgentCredential {
        agent_user_id: agent_user_id.clone(),
        owner_secret: owner_secret.clone(),
        owner_user_id,
        created_at: now,
        last_active_at: now,
    });

    // ... auto-join rooms, etc.
}
```

Two things stand out:

1. **`stdb_identity: String::new()`** — the user row is provisioned without an identity. The bot doesn't yet exist as a connected client; it's just a record waiting for a connection to claim it.
2. **`agent_credentials` is a private table** — the `owner_secret` lives there and is never sent to clients. It's the credential a future bot connection uses to prove "I am this bot."

## Pattern 2: The provision → register → connect dance

A bot has two distinct moments: **provision** (record created, awaiting connection) and **register** (a connection claims the record). Splitting them lets the human user trigger provisioning without holding the bot's connection.

```typescript
// 1. Human user calls provisionAgent (via their own STDB connection)
await humanService.reducers.provisionAgent({
  agentUserId: newBotId,
  displayName: 'helper-bot',
  ownerUserId: humanUserId,
  isSubAgent: false,
});

// 2. Read back the owner_secret (only the human can read agent_credentials
//    for bots they own — RLS or reducer-mediated access).
const credential = await fetchOwnerSecret(newBotId);

// 3. Open a NEW STDB connection — fresh identity — and pass owner_secret
const botService = new AgentChatService({
  agentUserId: newBotId,
  ownerSecret: credential.ownerSecret,
});

await botService.connect();
// Inside connect():
//   - Generate fresh STDB identity (via the SDK)
//   - Call registerAgentIdentity({ agentUserId, ownerSecret })
//   - That reducer validates the secret, then binds this connection's
//     identity to agentUserId in user_identity_links.
```

The third step's reducer is the trust boundary:

```rust
// commands/standalone/CHAT/spacetimedb-module/src/reducers/agents.rs
#[spacetimedb::reducer]
pub fn register_agent_identity(ctx: &ReducerContext, agent_user_id: String, owner_secret: String) {
    // Verify agent exists and is a bot
    let Some(user) = ctx.db.chat_users().user_id().find(&agent_user_id) else { return; };
    if user.is_bot != Some(true) { return; }

    // Verify owner_secret matches
    let Some(cred) = ctx.db.agent_credentials().agent_user_id().find(&agent_user_id) else { return; };
    if cred.owner_secret != owner_secret { return; }

    // Link this connection's identity to the agent_user_id
    let stdb_identity = sender_hex(ctx);
    register_identity_link(ctx, &stdb_identity, &agent_user_id);

    // Mark online
    ctx.db.chat_users().user_id().delete(&agent_user_id);
    ctx.db.chat_users().insert(ChatUser {
        stdb_identity: stdb_identity.clone(),
        online: true,
        last_seen_at: now,
        ..user
    });
}
```

This composes with chapter 1's identity-link first-link-wins rule. The bot's connection has a fresh STDB identity. It calls `register_agent_identity` with the secret, which calls `register_identity_link` — same helper humans use — to bind identity ↔ agent_user_id. From that moment on, the bot's connection is indistinguishable from any other STDB client (because that's what it is). Reducers it calls go through `get_caller_user_id`, get back the agent_user_id, and proceed.

### Why a separate connection?

Three reasons:

- **Subscription scope is per-connection.** The bot subscribes to channels it's a member of, separately from its owner. A bot in 50 rooms and an owner in 5 rooms are subscribed independently. Combining connections would force one side to subscribe to the other's data.
- **Identity-link slots.** Each connection gets one identity link. A shared connection can't be both the human and the bot.
- **Failure isolation.** If the bot's connection drops (token expired, server kicked it, network blip), the human is unaffected. Conversely, the human logging out doesn't disconnect their bots.

## Pattern 3: Heartbeat for presence

Online presence in STDB is "is the connection currently delivering subscription updates" — but for application-level "user is active" semantics, you usually want richer signal. The chat module uses a 30-second heartbeat:

```typescript
// commands/standalone/CHAT/services/AgentChatService.ts
private startHeartbeat(): void {
  this.stopHeartbeat();
  // Every 30 seconds, set status to online
  this.heartbeatInterval = setInterval(() => {
    if (this.isConnected()) {
      try {
        this.connection?.reducers.setStatus({ status: 'online' });
      } catch { /* ignore — disconnected between check and call */ }
    }
  }, 30_000);
}
```

The pattern: every 30 seconds, the bot's connection calls `setStatus('online')` on its own behalf. The reducer updates the bot's `chat_users.last_seen_at`, which subscriptions see, which keeps the bot rendered as "online" in member lists.

If the bot crashes or its connection drops, no more setStatus calls land. After ~60 seconds (two heartbeat windows), other clients can treat `last_seen_at + 60s < now` as "offline" without needing STDB to tell them.

This is **application-level liveness**. STDB's native connection-state events (onDisconnect) are more authoritative but only fire for clients you've directly observed. The heartbeat-and-staleness pattern is observable from any subscriber.

## Pattern 4: Exponential-backoff reconnection

Bot connections need to survive transient failures. The pattern:

```typescript
// commands/standalone/CHAT/services/AgentChatService.ts
private scheduleReconnect(): void {
  if (this.disposed || this.reconnectAttempts >= this.maxReconnectAttempts) {
    return;
  }

  const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30_000);
  this.reconnectAttempts++;
  devLog(`[${this.logTag}] Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);

  this.reconnectTimer = setTimeout(async () => {
    try { await this.connect(); }
    catch { /* will retry via onDisconnect */ }
  }, delay);
}
```

Properties worth naming:

- **Cap on attempts.** After `maxReconnectAttempts` (default 10), stop trying. A bot stuck in a reconnect loop because of a permanent problem (revoked owner_secret, banned user, broken module) shouldn't burn forever — it should fail loudly and let the supervising layer (AgentChatManager) decide to dispose it.
- **Cap on delay.** `Math.min(..., 30_000)` keeps the maximum delay at 30 seconds. Without the cap, after 10 attempts you'd wait 17 minutes between retries, which makes debugging miserable.
- **Reset attempts on successful connect.** When the connection comes back, `reconnectAttempts = 0` so the next disconnect starts the clock fresh.

For human user connections, you'd add UI feedback ("Reconnecting...") and probably reauthenticate. For bots, silent reconnect is fine — the heartbeat tells everyone the bot is alive again.

## Pattern 5: DDIL — queue reducer calls during disconnection

DDIL = "delivery during intermittent connectivity." When a bot's connection is briefly down, calls it would have made shouldn't be lost — they should be queued and replayed on reconnect.

```typescript
// commands/standalone/CHAT/services/AgentChatService.ts
private pendingCalls: Array<() => void> = [];

private replayPendingCalls(): void {
  const calls = [...this.pendingCalls];
  this.pendingCalls = [];
  for (const call of calls) {
    try { call(); }
    catch (err) { console.error(`[${this.logTag}] Failed to replay`, err); }
  }
}

queueOrCall(fn: () => void): void {
  if (this.isConnected()) {
    fn();
  } else {
    this.pendingCalls.push(fn);
  }
}
```

Application code calls `service.queueOrCall(() => service.reducers.sendMessage(...))` instead of calling reducers directly. If connected, the call fires immediately. If not, it's queued. On reconnect, the queue drains.

### What DDIL doesn't cover

- **Persistence across process restarts.** The queue is in memory. If the bot's process dies, the queue is lost. For mission-critical "this message must eventually be delivered" semantics, you'd need a persistent queue (writing to a STDB table works, but adds complexity).
- **Idempotency.** A queued reducer call replayed after reconnect might race with a server that already received and processed the original. Use chapter 5's client-supplied IDs to make replays safe.
- **Ordering.** The queue replays in FIFO order. If the call sequence has dependencies (call B requires call A's result), DDIL alone doesn't give you that — you need to either chain via callbacks or design reducers to be order-independent.

For chat-style "post messages, react, set status" workloads, DDIL is sufficient. For workflows with hard ordering or persistence requirements, design more carefully.

## Pattern 6: Per-tier spawn quotas

Once bots are first-class users, you have to govern how many a single human can create. Quotas are admin-managed and tier-based:

```rust
// In provision_agent
let owner_tier = get_user_tier(&owner_user_id);
let limit = ctx.db.agent_spawn_limits().tier().find(&owner_tier);
let max_agents = limit.map(|l| l.max_concurrent_agents).unwrap_or(0);

let active_count: u64 = ctx.db.chat_users().iter()
    .filter(|u| u.is_bot == Some(true)
        && u.bot_owner_user_id.as_deref() == Some(&owner_user_id))
    .count() as u64;

if active_count >= max_agents {
    log::warn!("[provision_agent] Spawn limit {}/{} reached for tier {}",
               active_count, max_agents, owner_tier);
    return;
}
```

The `agent_spawn_limits` table is admin-managed:

```rust
#[spacetimedb::table(accessor = agent_spawn_limits, public)]
pub struct AgentSpawnLimit {
    #[primary_key]
    pub tier: String,
    pub max_concurrent_agents: u32,
    pub bot_rate_limit_ms: Option<u64>,
    pub max_servers: Option<u32>,
}
```

Free tier: 0 bots. Pro: 5. Enterprise: 25. (Actual numbers per your business model.) The limits are **public** so clients can render quota UI without a separate API call. Admins update the limits via a reducer; the change propagates to all subscribed clients via the normal subscription model.

## Tradeoffs and gotchas

**One STDB connection per bot is bandwidth.** A user with 25 active bots has 25 active subscriptions. STDB handles this fine; the cost is real but bounded. Don't try to multiplex bots over one connection — you lose identity isolation, subscription scope, and failure isolation simultaneously.

**Owner_secret rotation is hard.** If a secret is leaked, you need to rotate it without disrupting the bot. The pattern: deprovision the bot, provision a new one with the same display_name, swap localStorage tokens. This causes brief downtime; for high-availability bots, design for it.

**Sub-agents (depth = 1).** The chat module allows agents to spawn sub-agents but caps at depth 1 to prevent runaway recursion. This is hard-coded; if your application allows deeper hierarchies, you need a separate rule. Recursion-aware quotas (counts include sub-agents) are usually right.

**Bot deletion cascades a lot.** Deprovisioning removes the chat_users row, the agent_credentials row, the user_identity_links rows, all server_members rows, all room_members rows. We've found writing this as one big reducer (`deprovision_agent`) is cleaner than sequencing calls — atomicity gives you "either fully deleted or still fully there." Half-deleted bots are bad.

**Heartbeat collides with `setStatus` from human users.** If your app has "set status to away/busy/dnd" UI, the heartbeat overwrites it. We solved this by having the heartbeat call only when the current status is `'online'` — explicit non-online statuses are sticky. Worth considering at design time.

**Bot connections don't have UI for reconnect feedback.** Humans see a "reconnecting..." indicator; bots don't. Logs are your only signal. For production deployments, plumb bot-connection state into a metrics dashboard so you can see when bots are repeatedly dropping.

**The `light mode` flag matters.** `withLightMode(true)` reduces SDK overhead for clients that don't need full table state. Bots that only call reducers and don't need to render UI from subscriptions benefit from this. Worth measuring.

## In the skill

The official [`spacetimedb-typescript` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-typescript) covers `DbConnection.builder()`, the connection lifecycle, and `withToken()` for persistence. The [`spacetimedb-rust` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-rust) covers reducer auth and table macros.

This chapter ties them into a multi-actor pattern. The agent-as-first-class-client approach isn't an STDB feature — it's a discipline you can build on top of STDB primitives, and one of the things that makes a real-time application feel coherent across humans and AI participants.

## In the code

| Concept | File | Purpose |
|---|---|---|
| Provision reducer | `module/src/reducers/agents.rs:84-217` | Create bot user + credentials |
| Register reducer | `module/src/reducers/agents.rs:219-270` | Connection claims bot identity |
| Private credentials table | `module/src/tables/agent_credentials.rs` | Owner-secret storage |
| Spawn limits table | `module/src/tables/agent_spawn_limits.rs` | Per-tier quotas |
| Bot connection class | `services/AgentChatService.ts` | Per-bot STDB connection |
| Heartbeat | `services/AgentChatService.ts:143-160` | 30s setStatus loop |
| Reconnection | `services/AgentChatService.ts:120-137` | Exponential backoff |
| DDIL queue | `services/AgentChatService.ts:163-183` | Replay on reconnect |
| Lifecycle orchestrator | `services/AgentChatManager.ts` | provision → connect → subscribe → deprovision |

---

[← Chapter 6: Performance Patterns](./06-performance-patterns.md) · [Cookbook index](../README.md)
