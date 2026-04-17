# domainc Architecture

domainc is a proc-macro crate. It reads veric's verified
program.rkyv at compile time and expands into per-program
domain types with rkyv derives. semac and rsc both invoke
the same macro on the same program.rkyv to get identical
types.

```rust
domainc::domains!(env!("PROGRAM_RKYV"));
```

See veric ARCHITECTURE.md for the full pipeline redesign
that defines domainc's input.
