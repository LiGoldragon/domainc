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

## v0.19 Status: NOT YET IMPLEMENTED

This repo is a stub. Implementation waits on veri-core's proper
redesign and veric's port to v0.19 aski-core.

## Dependencies (planned)

- veri-core (input rkyv types)
- proc-macro2, quote, syn
- rkyv

## VCS

`jj` mandatory. Git is storage backend only.
