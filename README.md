# SpacetimeDB Production Patterns

> A cookbook of patterns for problems SpacetimeDB leaves to userland — auth, multi-device identity, cross-module communication, subscription scope, schema discipline, and the workarounds that make production deployments survive contact with real users.

This is a companion to the official [SpacetimeDB skills](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills) and reference implementations. The skills tell you the API rules. The reference implementations show you what's possible. This cookbook fills the gap: the real-world patterns we've shipped and what it cost to learn them.

Each chapter follows the same shape:

- **The problem** — what SpacetimeDB doesn't give you out of the box
- **The pattern** — what we built on top of the primitives
- **In production** — direct links to the canonical implementation in the [Nexus Chat](https://github.com/Spitfire-Products/nexus-chat) reference repo
- **In the skill** — cross-references to the corresponding section of the official STDB skills
- **Tradeoffs** — what we'd do differently, what edge cases bit us, when *not* to apply this pattern

## Why this exists

Building Nexus Terminal (a multi-module real-time platform on SpacetimeDB) surfaced patterns that aren't documented anywhere else and aren't in the official skills, but turn up the moment you need:

- Multiple devices logged in as the same user
- More than one SpacetimeDB module communicating with each other
- Production durability under crash and network partition
- Subscription scope that scales beyond toy-app patterns
- Schema migrations that don't lose data

These problems are off the well-trodden path because the STDB philosophy (reasonably) leaves application concerns to the application. But every real production deployment hits them eventually, and the answers tend to be reinvented per project. This cookbook collects the answers we've shipped.

## Three artifacts, one ecosystem

| Artifact | Audience | Format | Where it lives |
|---|---|---|---|
| **STDB Skills** | AI coding agents | Terse, prescriptive, anti-hallucination | [Upstream Clockwork Labs repo](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills) |
| **STDB Cookbook** *(this repo)* | Human developers | Narrative, pattern-level, motivated by real production problems | github.com/Spitfire-Products/stdb-cookbook |
| **Nexus Chat** | Both | Working reference implementation — Discord-parity chat with 40 tables, 95 reducers, agent integration | github.com/Spitfire-Products/nexus-chat |

The three reinforce each other: the cookbook explains *why* a pattern exists, the skills enforce the *rules* an agent must follow, and Nexus Chat is the *canonical implementation* you can build, deploy, and read. Each chapter cross-references the other two.

## Chapters

| # | Chapter | Status |
|---|---|---|
| 1 | [Identity & Multi-Device Auth](chapters/01-identity-and-multi-device-auth.md) | **Ready** |
| 2 | [Cross-Module Communication](chapters/02-cross-module-communication.md) | **Ready** |
| 3 | [Subscription Engineering](chapters/03-subscription-engineering.md) | **Ready** |
| 4 | [Schema Discipline & The `--clear` Discipline](chapters/04-schema-discipline.md) | **Ready** |
| 5 | [Durability & Consistency](chapters/05-durability-and-consistency.md) | **Ready** |
| 6 | [Performance Patterns](chapters/06-performance-patterns.md) | **Ready** |
| 7 | [Agent & Bot Architectures](chapters/07-agent-and-bot-architectures.md) | **Ready** |

If a pattern you've shipped isn't covered here, [open an issue](../../issues) describing it — the cookbook is a living document and additions from production deployments are welcome.

## Conventions

- **Code references** are linked to specific files and line numbers in the [Nexus Chat](https://github.com/Spitfire-Products/nexus-chat) repo at the version this cookbook was written against. The chapter header records that version.
- **STDB version** — patterns are written against SpacetimeDB 2.0+ unless noted. Anything that depends on `features = ["unstable"]` is called out explicitly.
- **Languages** — module-side examples are Rust. Client-side examples are TypeScript. C# clients should map analogously but aren't covered explicitly.

## How to read this

If you're **starting a new STDB project**, read the chapters in order — the patterns build on each other.

If you're **stuck on a specific problem**, jump straight to the chapter. Each one is self-contained.

If you're **debugging an existing module**, the cross-references to the Nexus Chat code let you compare your implementation to a working production version.

If you're **using AI agents to write STDB code**, point them at the [official skills](https://github.com/clockworklabs/SpacetimeDB/tree/master/skills) first; come here for the *why*.

## Status & contributions

This cookbook is maintained alongside the [Nexus Chat](https://github.com/Spitfire-Products/nexus-chat) reference repo and the upstream STDB skills. Issues are welcome. PRs are reviewed best-effort — if you've got a pattern from your own production deployment that fits, open one.

Patterns that prove themselves here may eventually graduate into the official STDB skills upstream — that's the natural promotion path for community wisdom that's been validated in production.

## License

[MIT](LICENSE) — use this cookbook however you want, including in commercial projects. Attribution appreciated but not required.
