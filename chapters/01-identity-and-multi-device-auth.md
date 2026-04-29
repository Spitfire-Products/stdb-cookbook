# Chapter 1 — Identity & Multi-Device Auth

> *Written against SpacetimeDB 2.0. Canonical implementation: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) module, `src/utils/auth.rs` and `src/reducers/auth.rs`.*

## The problem

SpacetimeDB ships you exactly one identity primitive: `ctx.sender()` — the cryptographic identity of the WebSocket connection that called the reducer. That's it. Some consequences this leaves to userland:

1. **You don't have a user model.** STDB knows about identities, not users. A user with two devices has two identities, and STDB has no opinion on whether they're the same person.
2. **You don't have a login flow.** Connections are identity-stable across reconnects (token-based) but a *user* logging in on a fresh browser produces a brand new identity that has no history.
3. **You don't have role/tier/profile.** None of that information is part of the connection — it has to be associated with the identity by your application.
4. **You don't have account recovery.** Lose the localStorage token and you have no way to prove the new identity belongs to the same user. The application has to bridge that.

Most production STDB apps need all four. Skipping the design here means baking assumptions into every reducer that are painful to undo later.

## The pattern: identity-link table + first-link-wins reducer

Build a single `user_identity_links` table that maps STDB identities to your application's user IDs, and a `register_identity(user_id)` reducer that's the *only* path for a connection to claim a user_id. Every other reducer that needs to know "who is this caller as a user" goes through a helper that resolves `ctx.sender()` → identity link row → user_id.

This costs you one table, one reducer, two helper functions, and one RLS filter. It buys you:

- Multi-device support out of the box (same user_id, many identities)
- A clean security boundary (identity-link reducer is the trust boundary; everything else can assume "if you have a user_id, you've been authenticated upstream")
- A natural place to put auto-prune, last-seen tracking, and audit logging
- An easy migration path when you eventually add federated auth, social login, or session revocation — you only touch one reducer

### The table

```rust
// commands/standalone/CHAT/spacetimedb-module/src/tables/auth.rs
#[spacetimedb::table(accessor = user_identity_links, public)]
pub struct UserIdentityLink {
    #[primary_key]
    #[index(btree)]
    pub stdb_identity: String,    // hex of ctx.sender()
    #[index(btree)]
    pub user_id: String,           // your app's user identifier
    pub created_at: u64,
    pub last_seen_at: u64,
}
```

Two btree indexes are deliberate: one for `stdb_identity` (the lookup direction every reducer takes) and one for `user_id` (the lookup direction the auto-prune and admin queries take). PK on `stdb_identity` enforces "one link per identity" — multiple users can't claim the same connection.

Marked `public` because clients need to read their own row (filtered by RLS, see below). If you don't need clients to know their link exists, you can drop `public` and the RLS filter both.

### The RLS filter

```rust
// src/lib.rs
#[client_visibility_filter]
const IDENTITY_LINK_FILTER: Filter = Filter::Sql(
    "SELECT * FROM user_identity_links WHERE user_identity_links.stdb_identity = :sender"
);
```

Every client sees only their own link. This is the *only* table with a `client_visibility_filter` in the chat module — everything else is gated by reducer auth guards plus client-side subscription scope. RLS here is non-negotiable: if you let one client read another client's identity link, they can map identities to user_ids and impersonate by spoofing the `user_id` argument to other reducers.

Requires `features = ["unstable"]` in your `Cargo.toml`. As of STDB 2.0 the `#[client_visibility_filter]` macro is gated behind that feature flag.

### The register reducer

```rust
// src/reducers/auth.rs
#[spacetimedb::reducer]
pub fn register_identity(ctx: &ReducerContext, user_id: String) {
    if user_id.trim().is_empty() {
        log::warn!("[register_identity] Rejected: empty user_id");
        return;
    }
    let stdb_identity = sender_hex(ctx);
    register_identity_link(ctx, &stdb_identity, &user_id);
}
```

Public reducer — anyone connected can call it. Trust comes from the *user_id you pass*, not from this reducer. The application is responsible for upstream authentication (Firebase, Auth0, Supabase, your own platform module) before calling this. The reducer's job is to bind the connection's STDB identity to whatever user_id the application says they are.

This is the exact place where many implementations get the security wrong. The temptation is to add "verification" inside the reducer — but the reducer can't verify anything STDB itself doesn't know. The only check it *can* do is **first-link-wins**:

### First-link-wins (the core security rule)

```rust
// src/utils/auth.rs
pub fn register_identity_link(ctx: &ReducerContext, stdb_identity: &str, user_id: &str) {
    let now = crate::timestamp_ms(ctx);

    if let Some(existing) = ctx.db.user_identity_links()
        .stdb_identity().find(&stdb_identity.to_string())
    {
        // SECURITY: Reject re-linking to a different user_id.
        // Prevents identity hijacking where an attacker calls
        // register_identity with a victim's user_id to impersonate them.
        if existing.user_id != user_id {
            log::warn!(
                "[register_identity_link] REJECTED: identity {}... already linked to {}, cannot re-link to {}",
                &stdb_identity[..16.min(stdb_identity.len())],
                existing.user_id, user_id
            );
            return;
        }
        // Same user_id: heartbeat the last_seen_at
        ctx.db.user_identity_links().stdb_identity().delete(&stdb_identity.to_string());
        ctx.db.user_identity_links().insert(UserIdentityLink {
            stdb_identity: stdb_identity.to_string(),
            user_id: user_id.to_string(),
            created_at: existing.created_at,
            last_seen_at: now,
        });
    } else {
        ctx.db.user_identity_links().insert(UserIdentityLink {
            stdb_identity: stdb_identity.to_string(),
            user_id: user_id.to_string(),
            created_at: now,
            last_seen_at: now,
        });
    }

    // Auto-prune: keep only 10 most recent links per user
    let mut user_links: Vec<UserIdentityLink> = ctx.db.user_identity_links()
        .iter()
        .filter(|l| l.user_id == user_id)
        .collect();

    if user_links.len() > 10 {
        user_links.sort_by(|a, b| b.last_seen_at.cmp(&a.last_seen_at));
        for stale in &user_links[10..] {
            ctx.db.user_identity_links().stdb_identity().delete(&stale.stdb_identity);
        }
    }
}
```

The critical line: `if existing.user_id != user_id { return; }`. **Once an STDB identity has been bound to a user_id, no reducer call from that connection can re-bind it to a different user_id.** This closes the obvious attack: an attacker on a fresh device calls `register_identity` with the victim's user_id, hoping to inherit their data. Without first-link-wins, that succeeds. With it, the fresh device must produce its *own* user_id (and the upstream auth must have actually authenticated them as that user), at which point they get *their* data, not the victim's.

The corollary is that compromised connections are unrecoverable at this layer. If an attacker steals a victim's STDB token, they can call any reducer the victim could. The identity-link table doesn't help with that — token security is a separate concern (HTTPS, secure storage, rotation policies). What it *does* prevent is a much easier attack: identity *spoofing* without ever touching the victim's tokens.

### The lookup helper

Every reducer that needs to know the caller's user_id calls one helper:

```rust
// src/utils/auth.rs
pub fn get_caller_user_id(ctx: &ReducerContext) -> Option<String> {
    let caller_hex = ctx.sender().to_hex().to_string();
    ctx.db.user_identity_links().stdb_identity().find(&caller_hex)
        .map(|link| link.user_id.clone())
}
```

`None` means the connection hasn't called `register_identity` yet. Reducers use it like:

```rust
let Some(user_id) = get_caller_user_id(ctx) else {
    log::warn!("[send_message] Unauthorized");
    return;
};
```

This is the single trust boundary. If you have a user_id back from this helper, the rest of the reducer can assume the caller is authenticated as that user. If you don't, refuse the call.

## Auto-prune: capping links per user

Without auto-prune, a user who logs in on every device they touch (work laptop, home laptop, phone, partner's iPad, an old session that never properly logged out) accumulates identity links forever. Each link is small but: each costs a row, each costs RLS filter evaluation per reducer call from any connection, and each is a credential surface area.

Cap at a sensible number. In the chat module: 10. The cap is a heuristic — too low and legitimate users with many devices get logged out unexpectedly; too high and the table bloats. 10 is a comfortable middle that covers "phone + laptop + tablet + home browser + work browser" with headroom.

When the cap is exceeded, the *least recently seen* link is dropped. The user's most active devices stay; a forgotten browser session expires. The user just has to log in again on that device — same user_id, new identity, new link. No data loss.

## Client side: AuthBridge

Multi-module setups (Nexus has 7 STDB modules) want every module to learn the user's identity at the same time. Building that into each module's connection logic is duplication. The pattern we use is a single `AuthBridge` singleton that:

1. Owns the current user state (user_id, tier, role, displayName)
2. Knows about every connected STDB module via `register(moduleId, capabilities)`
3. On login, walks every registered module and calls its `registerIdentity` capability
4. On profile changes (tier upgrade, display name edit), pushes those changes to every module that supports the corresponding sync reducer

```typescript
// shared/spacetimedb/AuthBridge.ts (excerpt)
register(id: string, opts: { getConnectionState, capabilities }) {
  this.modules.set(id, { id, ...opts });
  if (this.currentUserId) this.syncModule(id);
}

onLogin(userId: string, tier?: string, role?: string, displayName?: string) {
  this.currentUserId = userId;
  this.currentTier = tier ?? null;
  this.currentRole = role ?? null;
  this.currentDisplayName = displayName ?? null;
  this.syncAllModules();
}

private syncModule(id: string) {
  const mod = this.modules.get(id);
  if (mod.getConnectionState() !== 'connected') return;
  if (this.currentUserId) {
    this.syncCapability(id, 'identity', () =>
      mod.capabilities.registerIdentity(this.currentUserId!));
  }
  // ... tier, role, displayName
}
```

Two things this bridge enforces that aren't obvious:

- **Sync is gated on `getConnectionState() === 'connected'`.** Calling `registerIdentity` reducer on a not-yet-connected module fails silently. The bridge buffers the sync until the module reports connected, at which point its first action is identity registration.
- **Each capability is independent.** A module that supports `registerIdentity` but not `syncTier` (older modules in a multi-version deploy) just gets the supported subset. Capabilities are a per-module declaration, not a contract every module has to satisfy.

The bridge lives entirely client-side. Nothing on the module side knows about it; modules just expose the reducers. That decoupling is the whole point — the bridge is application code, not STDB code.

## Tradeoffs and gotchas

**Display name racing.** Auth providers (Firebase, etc.) often have their own display name field that doesn't match what the user picked in your app. Don't pass the auth provider's display name to `onLogin` — it'll overwrite the user's chosen name on every reconnect. We learned this the hard way; the AuthBridge takes `displayName` only when it's been set explicitly by the user, never auto-populated from auth.

**Token recovery is out of scope.** If a user clears localStorage, their STDB identity is gone. The next login produces a new identity, which calls `register_identity` with their existing user_id and gets a new link added. From the user's perspective: they "lost" nothing because data is keyed by user_id. From STDB's perspective: a new identity link was created. From a security perspective: this is fine because the upstream auth provider verified them as that user_id before the bridge called `registerIdentity`.

**Identity-link bloat is a real cost.** We've seen single users accumulate 30+ links over a year of casual use across devices. Auto-prune at 10 is conservative; bump it if your use case has legitimately more devices, but watch for cases where a user appears to have hundreds of "devices" — that's usually a bug in your auth flow, not a feature, and the cap surfaces it.

**RLS overhead scales with subscriber count.** The `user_identity_links` filter runs per reducer call from every subscribed connection. At very high user counts this becomes measurable. If you hit that scale, consider whether the link table needs to be `public` at all — if clients never need to read their own link directly (the helper is server-side only), you can drop `public` and the filter together.

**This pattern doesn't help with bots.** Agent connections that pretend to be users need their own scheme. We use a sibling pattern (`provision_agent` reducer that creates a synthetic user_id with `is_bot=true` and binds the agent's STDB identity to it via the same identity-link mechanism). Covered in chapter 7.

## In the skill

The official [`spacetimedb-rust` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-rust) covers `ctx.sender()` semantics, the `#[client_visibility_filter]` macro, and the `features = ["unstable"]` requirement. This chapter shows how those primitives compose into a production auth pattern.

The official [`spacetimedb-typescript` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-typescript) covers `.withToken()` for connection persistence and the table-update lifecycle. The AuthBridge in this chapter is a higher-level wrapper that coordinates multiple connections.

## In the code

| Concept | File | Purpose |
|---|---|---|
| Identity link table | `module/src/tables/auth.rs` | Schema |
| RLS filter | `module/src/lib.rs` (lines 33-36) | Per-client visibility |
| `register_identity` reducer | `module/src/reducers/auth.rs` | Public entry point |
| `register_identity_link` helper | `module/src/utils/auth.rs` (lines 39-86) | First-link-wins + auto-prune |
| `get_caller_user_id` helper | `module/src/utils/auth.rs` (lines 18-22) | Trust boundary lookup |
| `is_platform_admin` helper | `module/src/utils/auth.rs` (lines 31-35) | Role gate |
| AuthBridge (client) | `shared/spacetimedb/AuthBridge.ts` | Cross-module sync coordinator |

---

[← Cookbook index](../README.md) · [Chapter 2: Cross-Module Communication →](./02-cross-module-communication.md) *(planned)*
