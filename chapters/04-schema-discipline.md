# Chapter 4 — Schema Discipline & The `--clear` Discipline

> *Written against SpacetimeDB 2.0/2.1, with a 2.6 forced-migration case study (Pattern 5). Canonical implementations: [nexus-chat](https://github.com/Spitfire-Products/nexus-chat) `module/src/tables/*.rs` and `module/src/reducers/shadow_index_backfill.rs`.*

## The problem

SpacetimeDB has automatic schema migrations. They're great when they work and terrifying when they don't. The set of changes STDB can apply non-destructively is narrower than feels intuitive, and the failure mode for everything else is `--clear` — wipe all data, redeploy from scratch. In production, that's a P0.

The migration compatibility matrix:

| Change | Data preserved? |
|---|---|
| Add new table | Yes |
| Add new reducer | Yes |
| Add/remove indexes | Yes |
| Add `Option<T>` column **at end of struct** | Yes |
| Add a non-`Option<T>` column | **No — requires `--clear`** |
| Remove a column | **No** |
| Reorder columns in a struct | **No** |
| Change a column's type | **No** |
| Change a column's nullability | **No** |
| Change PK or unique-key shape | **No** |

The compatibility rules apply to both `spacetime publish` and `spacetime dev`. In `dev` mode, `--delete-data` defaults to `on-conflict`, which means **dev mode silently wipes your database on incompatible schema changes**. Production-ready discipline is `spacetime dev --delete-data=never` so dev fails loudly the same way prod does.

## Pattern 1: New columns are always `Option<T>` at the end of the struct

This is rule #1 and it's easy to violate without realizing. Two examples that look identical to the reader and behave very differently to STDB:

```rust
// COMPATIBLE — added at end, Option<T>
pub struct Message {
    pub id: String,
    pub room_id: String,
    pub content: String,
    pub created_at: u64,
    pub edited_at: Option<u64>,
    // ... existing columns ...
    pub flags: u64,
    pub is_bot_author: Option<bool>,    // ← new column, end, Option, OK
}

// REQUIRES --clear — added in middle
pub struct Message {
    pub id: String,
    pub room_id: String,
    pub priority: Option<u8>,           // ← inserted before existing columns
    pub content: String,
    pub created_at: u64,
    // ...
}

// REQUIRES --clear — non-optional
pub struct Message {
    pub id: String,
    // ...
    pub flags: u64,
    pub priority: u8,                   // ← non-optional, even at end, breaks
}
```

The "always at end" rule exists because STDB pattern-matches struct shapes during migration. Reordering changes the shape; appending preserves it. The "always `Option<T>`" rule exists because STDB needs a value for every existing row, and `Option<T>` provides that value (`None`) without you having to write a backfill.

### When you actually need a default that isn't None

Sometimes `None` is wrong as a default — e.g., you're adding a `created_at` column to old rows that should get a real timestamp, not null. The pattern:

1. Add the column as `Option<u64>` first, deploy.
2. Write and run a one-time backfill reducer that fills `Some(timestamp)` for every existing row.
3. Optionally, in a *later* migration, you can add a non-null mirror column and deprecate the optional one.

You can't do step 3 in step 1. STDB will wipe the data. We have not found a shortcut to this. (Open issue worth tracking: a future "alter column" migration may relax this.)

### Things you can never do without `--clear`

- Change a column's type (`u32 → u64`, `String → Vec<u8>`)
- Reorder columns in a struct
- Remove a column
- Change a column's nullability either direction (`Option<T> → T` or `T → Option<T>`)
- Change a primary key

The honest answer for most of these: **add a new column with the new shape, dual-write to both for a release, then deprecate the old in a later release**. Don't try to do it in one shot. The pattern is exactly what a SQL migration with backfills looks like; STDB just enforces it through type-system errors instead of letting you write a destructive `ALTER TABLE`.

## Pattern 2: Never `--clear` without explicit user permission

This is operational discipline more than code. In our deploy script:

```bash
# Default: preserves data
npm run stdb:deploy

# DESTRUCTIVE — only with explicit user permission
npm run stdb:deploy:clear
```

The deploy script's default never passes `--clear`. The destructive variant is a separate npm script with a different name. The reasoning isn't paranoia — it's that the failure mode of accidental `--clear` is **silent**. STDB will happily wipe production data and redeploy in 5 seconds. There's no warning, no dry-run, no recovery — your only option after the fact is restore from a backup you'd better have taken.

If a deploy fails because of an incompatible schema change, the right response is **stop, investigate, propose a migration**. Not "rerun with `--clear` to get past it." We've drilled this rule hard because the cost of getting it wrong once is several hours of recovery.

The corollary for AI-assisted work: any agent that can run `spacetime publish --clear` *needs explicit user authorization for the specific deploy*, not blanket permission. Memory: ["NEVER use `--clear` without explicit user permission"].

## Pattern 3: Shadow indexes for `Option<T>` filter limits

Subscriptions can't filter `Option<T>` columns at all. The fix is a denormalized table that mirrors only the rows where the optional value is set, with the column re-typed as non-optional:

```rust
// commands/standalone/CHAT/spacetimedb-module/src/tables/reaction_room_index.rs
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

The source table (`reactions`) keeps `room_id: Option<String>` because DM reactions exist and don't have a server room. The shadow index only contains rows where `room_id` is `Some` — server-scoped reactions only. Clients that only care about server reactions subscribe to the index, not the source.

### The dual-write contract

Every reducer that writes the source table must also write the shadow index:

```rust
// In the reaction reducer
pub fn toggle_reaction(ctx: &ReducerContext, message_id: String, emoji: String) {
    // ... derive room_id from message_id ...
    let reaction_id = format!("{}:{}:{}", user_id, message_id, emoji);

    // Source-table write
    ctx.db.reactions().insert(Reaction {
        id: reaction_id.clone(),
        room_id: room_id.clone(),  // Option<String>
        message_id: message_id.clone(),
        user_id: user_id.clone(),
        emoji: emoji.clone(),
        created_at: now,
    });

    // Shadow index dual-write — only when room_id is Some
    if let Some(rid) = room_id {
        ctx.db.reaction_room_index().insert(ReactionRoomIndex {
            reaction_id,
            room_id: rid,
            message_id,
            user_id,
            emoji,
            created_at: now,
        });
    }
}
```

STDB reducers are atomic — both inserts commit together or both roll back. So the shadow index is *always* in sync with the source as long as the dual-write is in every relevant reducer. Cascade-deletes need the same dual-write treatment (see chapter 5).

### Backfilling existing rows

When you add the shadow index to an existing table, you need to populate it from rows that already exist. Write an admin-only backfill reducer:

```rust
#[spacetimedb::reducer]
pub fn backfill_reaction_room_index(ctx: &ReducerContext) {
    if !is_platform_admin(ctx) { return; }

    let mut indexed = 0u64;
    let mut already_indexed = 0u64;
    let mut skipped_no_room = 0u64;

    let rows: Vec<Reaction> = ctx.db.reactions().iter().collect();
    for r in rows {
        if ctx.db.reaction_room_index().reaction_id().find(&r.id).is_some() {
            already_indexed += 1;
            continue;
        }
        let Some(room_id) = r.room_id else {
            skipped_no_room += 1;
            continue;
        };
        ctx.db.reaction_room_index().insert(ReactionRoomIndex {
            reaction_id: r.id, room_id, message_id: r.message_id,
            user_id: r.user_id, emoji: r.emoji, created_at: r.created_at,
        });
        indexed += 1;
    }

    log::info!("[backfill] indexed={} skipped_no_room={} already_indexed={}",
               indexed, skipped_no_room, already_indexed);
}
```

Critical properties: **idempotent** (re-running is safe — already-indexed rows are skipped) and **scoped** (only writes the index, never mutates the source). Run once after the first deploy that adds the shadow index. The cost is one CLI call: `spacetime call your-module backfill_reaction_room_index`.

### When NOT to use shadow indexes

Shadow indexes are denormalization. They double write cost on every reducer that writes the source. Only worth it when:

- The source has an `Option<T>` column you genuinely need to filter on
- The filter is in a high-traffic subscription
- The shadow index size is bounded (you can't shadow-index a billion-row table)

If a column is *almost always* `Some`, consider just making it required (with a migration) instead. Shadow indexes are the workaround when you genuinely need optionality but also need filterability.

## Pattern 4: Mirror tables for cross-module data exposure

This is shadow-index logic generalized: when one module needs to expose data to clients but the source table has visibility constraints (e.g., RLS), build a public mirror.

The chat module exposes `runtime_config_mirror` — a public, read-only mirror of internal runtime configuration. Internal modules write the canonical config to a private table; a reducer mirror-writes to the public table on every change. Clients subscribe only to the public mirror. The pattern is the same write-amplification cost; the benefit is a clean separation between internal control plane and public data plane.

Generalized rule: **when a write goes into a table that clients can't subscribe to (private, RLS-restricted, or cross-module), mirror to a table they can.** Don't try to make the source table public — you lose your security boundary. Don't make clients hit an HTTP endpoint to read it — you lose real-time updates. Mirror.

## Pattern 5: The backup → `--clear` → restore harness (when a column type MUST change)

Pattern 1 says "never change a column's type." But sometimes you don't get a choice — an *upstream* change forces a breaking migration on a table full of production data. When that happens, the only safe path is a **deliberate, instrumented `--clear` with a full backup/restore cycle**. This is the disaster-recovery harness you hope you never need and are very glad you built.

### The forcing function (a real one)

SpacetimeDB maincloud is a managed cloud — Clockwork controls when it upgrades. In mid-2026 it auto-upgraded to 2.6, which shipped [PR #5287 "Parameterized query plans"](https://github.com/clockworklabs/SpacetimeDB/pull/5287). That retyped the RLS `:sender` from a substituted literal into a **typed `Identity` parameter**. Overnight, every `#[client_visibility_filter]` comparing a **hex-`String`** column to `:sender` was rejected at publish:

```
Unexpected type: (expected) (__identity__: U256) != String (inferred)
```

The fix was to migrate the RLS column `String → Identity` — a **column-type change**, which Pattern 1 tells you needs `--clear`. Across 7 modules, including data-bearing ones (one with 1.1M rows, one with ~18K rows of non-regenerable agent state). No amount of schema discipline prevents this; the discipline is in *how you survive it*.

### The three pieces

**1. Backup — read-only, owner-token, captures private + RLS tables.** `spacetime sql` has no `--json` in the CLI, so use the HTTP SQL API (`POST /v1/database/<module>/sql`) authenticated with the **owner token** from `~/.config/spacetime/cli.toml`. The owner bypasses RLS, so a single pass captures private and RLS-scoped tables too. Write each table to JSON plus a manifest with row counts and a sha256 per table.

**2. Verbatim import reducers — `import_<table>` + `import_<table>_batch`.** Generate, per table, a reducer that inserts a backed-up row exactly as captured. A macro keeps it terse:

```rust
macro_rules! import_table {
    ($single:ident, $batch:ident, $accessor:ident, $ty:ty) => {
        #[reducer]
        pub fn $single(ctx: &ReducerContext, row: $ty) -> Result<(), String> {
            require_recovery_owner(ctx)?;
            ctx.db.$accessor().insert(row);
            Ok(())
        }
        #[reducer]
        pub fn $batch(ctx: &ReducerContext, rows: Vec<$ty>) -> Result<(), String> {
            require_recovery_owner(ctx)?;
            for row in rows { ctx.db.$accessor().insert(row); }
            Ok(())
        }
    };
}
import_table!(import_users, import_users_batch, users, User);
// ... one line per table (skip scheduled/job tables — they re-seed) ...
```

The row-struct-as-reducer-arg is the key trick: the reducer takes the full table row type, so the restore driver sends back exactly what the backup captured. **Codegen these from the table list** — for an 80-table module you do not hand-write them. (Gotcha: a glob `use crate::tables::*` does NOT bring the accessor methods into scope — you get "no method named X" for every table. Use explicit per-module `use crate::tables::<mod>::{<accessor>, <Struct>};`.)

**3. Restore driver — batched, bounded-concurrency `/call` replay + verify.** Read the backup JSON, POST each table's rows to `/call/import_<table>_batch`, then re-export and diff counts against the manifest.

### The flow

```
fresh backup  →  --clear publish (migrated schema)  →  claim recovery owner
              →  2-pass restore  →  verify counts  →  runtime-prove the migration
```

### Hard-won details

- **Two-pass restore.** Data tables restore batched (`--batch 1000`). **Identity-keyed tables restore per-row** (`--batch 1`). Why: the post-clear bootstrap (and any live reconnect) writes an identity row that **collides** with a backed-up row; a unique/PK-collision `insert` **panics**, and a panic in a batch loses the *entire* batch. Per-row, you lose exactly the one already-present row (count still reconciles).
- **Identity `/call` encoding.** A migrated `Identity` column is sent as `["0x<hex>"]` (single-element array, `0x`-prefixed). `Option<T>` args encode as `{"some": v}` (object) or `[0, v]`/`[1, []]` (tag) — both are accepted over HTTP `/call`.
- **Batch body ≈ 1 MB limit.** Even small rows × 1000 can blow the `/call` body ceiling. Compute a safe batch per table (`rows × avg_bytes < ~800 KB`); a table averaging 1.6 KB/row needs `--batch 500`, not 1000.
- **Oversized single rows (> ~1 MB) `413` and cannot be restored via HTTP `/call`** — e.g. a chat attachment storing a 3.5 MB base64 image inline. Fall back to the `spacetime call` CLI (different limit) or skip + self-heal. (Also: that's a design smell — large blobs belong in object storage, not an inline column.)
- **Throughput.** Per-row restore is commit-bound (~180 rows/s — one fsync per row). Batched (1000/txn) sustains ~185–210K rows/s once the host's token-validator cache is warm. Frame this honestly as *rows/sec via batching* — not "beat the TPS benchmark" (rows ≠ transactions; the 160K-tx/s figure is WebSocket-pipelined capacity, a different axis).
- **Skip scheduled/cron tables and regenerable caches.** Scheduled tables re-seed from their schedulers (or a `seed_*`/`arm_*` reducer you call post-restore). Admin/analytics caches regenerate on the next refresh — skip them and re-run the refresh reducer instead of restoring stale snapshots.

### Authorizing the restore is the genuinely hard part

Post-`--clear` the database is **empty** — no users, no roster, no identity links. So the recovery import reducers *cannot* check any stored role/membership to authorize the operator. And the obvious "is this an internal call" check, `ctx.sender() == ctx.identity()`, **does not match the owner's CLI** — `ctx.identity()` is the *module's* identity, not the database owner's, so it's only true for internal/scheduled calls. (We shipped this wrong twice before pinning it down.)

The only signal available post-clear is the caller's intrinsic **identity**. The pattern that works without hardcoding a specific identity is **claim-based trust-on-first-use**:

```rust
#[spacetimedb::table(accessor = recovery_owner)]   // private singleton
pub struct RecoveryOwner { #[primary_key] pub id: u8, pub owner: Identity }

#[reducer]
pub fn claim_recovery_owner(ctx: &ReducerContext) -> Result<(), String> {
    match ctx.db.recovery_owner().id().find(&0u8) {
        Some(ro) if ro.owner == ctx.sender() => Ok(()),               // idempotent
        Some(_) => Err("already claimed by another identity".into()),
        None => { ctx.db.recovery_owner().insert(RecoveryOwner { id: 0, owner: ctx.sender() }); Ok(()) }
    }
}

fn require_recovery_owner(ctx: &ReducerContext) -> Result<(), String> {
    if ctx.sender() == ctx.identity() { return Ok(()); }             // internal/scheduled
    match ctx.db.recovery_owner().id().find(&0u8) {
        Some(ro) if ro.owner == ctx.sender() => Ok(()),
        _ => Err("not the recovery owner — call claim_recovery_owner first".into()),
    }
}
```

The restore's first step is `claim_recovery_owner` — whoever runs the restore (the operator) becomes the authorized owner; every `import_*` then requires `sender == recovery_owner`. The TOFU window is only the post-clear moment, operator-controlled. A `release_recovery_owner` (current-owner-only) supports re-keying.

> **The vuln we shipped first.** Our initial gate was "caller is a *registered* identity" — which, the moment any normal user logs in, lets *them* call `import_users`, `import_secrets`, `import_*_keys`, `import_org_memberships`… i.e. inject arbitrary rows into any table. Recovery import reducers are **owner-only by nature**; design the gate for that from line one, not as a follow-up. If you generate powerful reducers, generate the authorization with them.

## Tradeoffs and gotchas

**Migration compatibility is checked at publish time, not author time.** You won't know your column add is incompatible until `spacetime publish` rejects it (or worse, a `dev`-mode `--delete-data` wipe runs). Always have a `cargo check` step in CI; it catches some structural errors before publish.

**`spacetime publish` skips the build if the WASM is current.** If you tweak only the deploy script and the WASM didn't rebuild, your "fix" doesn't deploy. The deploy script in nexus-chat explicitly rebuilds the WASM as a separate step before `--bin-path`. This is fragile and worth knowing.

**Generated bindings must be regenerated.** Every schema change updates the TypeScript bindings. Forgetting to commit the regenerated bindings causes silent client/server skew — the client thinks the schema is the old shape, the server delivers the new shape, and rows arrive with garbage data. Our deploy script runs `spacetime generate` then asserts no uncommitted changes. CI failure is better than runtime failure.

**Auto-prune logic on tables that grow unbounded.** Identity links, agent memory, audit logs, scheduled jobs. Each has a natural cap (10 most recent, 1 hour TTL, etc.) but only if you write the prune. STDB doesn't auto-prune. Forgetting this is how a working table becomes a 100M-row performance problem six months later.

**Replit + Rust 1.93 needs `GLIBC_TUNABLES=glibc.rtld.optional_static_tls=65536`.** Specific to Replit dev environments — without it, Rust 1.92+ binaries fail with "cannot allocate memory in static TLS block" because Replit's `LD_AUDIT` runtime loader consumes static TLS space. Set in deploy scripts and `cargo check` invocations. This is on the deploy infrastructure side, not a STDB concern, but it bites STDB module builds first because they're large compilations.

**`features = ["unstable"]` is required for RLS as of STDB 2.0.** Don't remove it from `Cargo.toml` — the `#[client_visibility_filter]` macro is gated behind it, and dropping the feature flag silently drops your RLS rules. Code review for `features = []` changes on STDB modules. (Once Views API replaces RLS, this gate may relax — but until then, treat the unstable flag as load-bearing.)

**On managed cloud, you don't control when the server upgrades — so a breaking change can arrive unbidden.** RLS is gated `unstable`, which means it's allowed to break across versions, and maincloud auto-upgrades on Clockwork's schedule, not yours. The 2.6 `:sender` retype (Pattern 5) shipped as an internal refactor marked "breaking changes: N/A" and silently broke every `String = :sender` RLS filter at next-publish. Defenses: keep the backup/restore harness ready *before* you need it, pin your CLI/crate versions so the toolchain is reproducible, and treat "anonymous read of an RLS table returns a type error" as the canary that the server's `:sender` typing changed under you. Your module schema must satisfy the *server's* validator — and on managed cloud the server can move first.

## In the skill

The official [`spacetimedb-rust` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-rust) covers the table macro syntax, `Option<T>` semantics, and the publish flow. The [`spacetimedb-cli` skill](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills/spacetimedb-cli) covers `--clear`, `--bin-path`, and the migration plan output.

The migration compatibility table in this chapter expands on the skill content with the specific failure modes we've hit in production. The skill is correct on the rules; this chapter is correct on what they cost when you violate them.

## In the code

| Concept | File | Purpose |
|---|---|---|
| `Option<T>` at end pattern | `module/src/tables/users.rs`, every table | Schema additivity |
| Shadow index table | `module/src/tables/reaction_room_index.rs` | `Option<T>` workaround |
| Dual-write reducer | `module/src/reducers/reactions.rs:62-99` | Source + shadow atomic write |
| Cascade-delete dual-write | `module/src/reducers/messages.rs:393-415` | Delete from both tables |
| Idempotent backfill | `module/src/reducers/shadow_index_backfill.rs` | Populate from existing rows |
| Deploy script (data-safe) | `scripts/spacetimedb-publish-chat.sh` | Default is preserve, `--clear` separate |
| Verbatim import reducers | `module/src/reducers/import_recovery.rs` | `import_<table>(_batch)` for backup/restore (Pattern 5) |
| Claim-based recovery owner | `import_recovery.rs` (`recovery_owner` + `claim_recovery_owner`) | Authorize the post-`--clear` restore without a roster |
| Backup / restore / verify harness | `scripts/stdb-{backup.sh,restore.mjs,verify.mjs}` | Owner-token export, batched `/call` replay, count-diff verify |

---

[← Chapter 3: Subscription Engineering](./03-subscription-engineering.md) · [Cookbook index](../README.md) · [Chapter 5: Durability & Consistency →](./05-durability-and-consistency.md)
