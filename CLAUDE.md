# domainc — Per-Program Domain Type Generator (proc macro)

domainc is a Rust **procedural macro** (not a binary). It reads
`program.rkyv` (from veric) at compile time and generates Rust
domain types for that specific program — enums, structs, newtypes,
consts — with rkyv derives.

## Role in the Pipeline

```
veric        — per-module rkyv → program.rkyv (veri-core types)
domainc      — program.rkyv → Rust domain types (proc macro) — THIS REPO
semac        — program.rkyv + domain types → .sema (pure binary)
rsc          — .sema + domain types → .rs (Rust projection)
```

## Pattern

Invocation in consuming crates:

```rust
domainc::domains!(env!("PROGRAM_RKYV"));
```

The macro expands into one generated Rust type per entity in the
program. This is the **input-specific** counterpart to corec's
output (corec generates ecosystem-wide contract types; domainc
generates per-program application types).

## Why a Proc Macro

Per-program types must be known at Rust compile time so downstream
tools (semac, rsc) can produce correctly-typed binaries and
projections. A build-time proc macro reading rkyv is the natural
shape (same pattern as askic-assemble, which will read dsls.rkyv).

## ⚠️ v0.20 Status: STUB — NOT IMPLEMENTED

**Current state:** stub repo. Only contains CLAUDE.md and git scaffolding.
No Cargo.toml, no src/, no proc-macro code.

**Why implementation is blocked:**

1. veri-core's D6 redesign hasn't landed — `program.rkyv` doesn't
   yet have the shape (EntityRef, parallel typed entities, embedded
   RelatesTo) that domainc would read from.
2. veric hasn't been ported to v0.20 aski-core — so program.rkyv
   doesn't exist as real output yet.
3. askic hasn't been rewritten — so veric has no input to consume.

**Dependency chain to resolution:**

1. askic-assemble proc-macro crate created (v0.20)
2. askic rewritten against askic-assemble
3. veri-core D6 redesign (parallel with 1-2 or after askic output exists)
4. veric ported to v0.20 aski-core + v0.20 veri-core
5. domainc THEN can be implemented against veri-core's shape

**When domainc is implemented:**

Invocation pattern in consuming crates:

```rust
domainc::domains!(env!("PROGRAM_RKYV"));
```

Expands into one generated Rust type per entity in the program
(enums, structs, newtypes, consts). This is the input-specific
counterpart to corec's ecosystem-wide contract generation.

Per-program types must be known at Rust compile time so downstream
tools (semac, rsc) can produce correctly-typed binaries and
projections. Same build-time proc-macro pattern as askic-assemble
(which reads dsls.rkyv).

## Dependencies (planned)

- veri-core (input rkyv types)
- proc-macro2, quote, syn
- rkyv

## VCS

`jj` mandatory. Git is storage backend only.
