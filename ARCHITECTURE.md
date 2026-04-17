# domainc Architecture

domainc reads askic's rkyv parse tree and generates a per-program
Rust domain crate. The generated crate IS the semac-rsc rkyv
contract — same pattern as aski-core (askicc-askic) and aski
(askic-semac).


## Input

A single `.rkyv` file produced by askic. Contains
`Vec<aski::RootChild>` serialized with rkyv.

domainc deserializes this using the aski crate (the same types
askic serialized with). Zero-copy via rkyv — domainc reads
ArchivedRootChild directly from the byte buffer.

### What's in the parse tree

RootChild is a nine-variant enum:

```
Module(ModuleDef)         — module name, exports, imports
Enum(EnumDef)             — bare, data-carrying, struct, nested
Struct(StructDef)         — typed fields, self-typed, nested
Newtype(NewtypeDef)       — wraps one type
Const(ConstDef)           — typed constant with literal value
TraitDecl(TraitDeclDef)   — trait name + method signatures
TraitImpl(TraitImplDef)   — method bodies on a type ← SKIP
Ffi(FfiDef)               — foreign function declarations ← SKIP
Process(Block)            — entry point block ← SKIP
```

### What domainc reads (the nouns)

- **ModuleDef** — `name: TypeName`, `exports: Vec<ExportItem>`,
  `imports: Vec<ModuleImport>`. Gives module scope structure.

- **EnumDef** — `name: TypeName`, `children: Vec<EnumChild>`,
  `generic_params: Vec<GenericParamDef>`, `visibility`, `derives`.
  EnumChild has five variants:
  - `Variant { name }` — bare
  - `DataVariant { name, payload: TypeExpr }` — tuple-like
  - `StructVariant { name, fields: Vec<StructField> }` — struct-like
  - `NestedEnum(EnumDef)` — recursive
  - `NestedStruct(StructDef)` — recursive

- **StructDef** — `name: TypeName`, `children: Vec<StructChild>`,
  `generic_params`, `visibility`, `derives`.
  StructChild has four variants:
  - `TypedField { name, typ: TypeExpr }` — explicit type
  - `SelfTypedField { name }` — name IS the type
  - `NestedEnum(EnumDef)` — recursive
  - `NestedStruct(StructDef)` — recursive

- **NewtypeDef** — `name: TypeName`, `wraps: TypeExpr`,
  `generic_params`, `visibility`, `derives`.

- **ConstDef** — `name: TypeName`, `typ: TypeExpr`,
  `value: LiteralValue`. LiteralValue is `Int(i64) | Float(f64)
  | Str(String) | Bool(bool) | Char(u32)`.

- **TraitDeclDef** — `name: TraitName`,
  `signatures: Vec<MethodSig>`. domainc only needs the trait
  name for the scope index enum. It does NOT process signatures
  or method bodies.

### What domainc skips (the verbs)

- **TraitImpl** — has method bodies with expressions. semac
  compiles these.
- **Process** — entry point block. semac compiles it.
- **Ffi** — foreign function declarations. semac generates
  extern blocks.

domainc generates the NOUNS (types). semac generates the
VERBS (implementations).


## Output

A Rust crate. Source tree written to an output directory:

```
<program>-domains/
  Cargo.toml
  src/
    lib.rs
```

### What it generates

For each module in the parse tree, domainc generates:

#### 1. Scope index enums

One enum per domain kind that exists in the module:

```rust
pub enum <Module>Enums { Element, Quality, ... }
pub enum <Module>Structs { Point, ... }
pub enum <Module>Newtypes { Counter, ... }
pub enum <Module>Traits { Describe, ... }
pub enum <Module>Consts { MaxSigns, ... }
```

These are O(1) static hashmaps. The enum discriminant IS
the index. No strings, no lookup tables.

#### 2. Domain enums

```rust
pub enum Element { Fire, Earth, Air, Water }
```

Bare enums get: `Copy, Eq, Hash` derives. Data-carrying
enums get: rkyv serialize/deserialize bounds. Same logic as
corec's `emit_derives`.

#### 3. Domain structs

```rust
pub struct Point {
    pub horizontal: f64,
    pub vertical: f64,
}
```

All fields public. PascalCase names converted to snake_case
via corec's `to_snake`. Self-typed fields: field name IS the
type (`Name` field has type `Name`).

#### 4. Newtypes

```rust
pub struct Counter(pub u32);
```

#### 5. Constants

```rust
pub const MAX_SIGNS: u32 = 12;
```

Name converted to SCREAMING_SNAKE_CASE. Type mapped through
corec's `map_primitive`.

#### 6. Nested types

Nested enums and nested structs inside enums/structs are
extracted and emitted as top-level types. Scoped by parent
name to avoid collision:

```aski
{Engine (| State Ready Active Done |) (Current State)}
```

Generates:

```rust
pub enum EngineState { Ready, Active, Done }

pub struct Engine {
    pub current: EngineState,
}
```

### rkyv derives on everything

All generated types have rkyv Archive + Serialize +
Deserialize derives. Data-carrying types additionally get
serialize_bounds and deserialize_bounds. Fields with Box,
Vec, or Option get `#[rkyv(omit_bounds)]`.

This is EXACTLY what corec's codegen already does. domainc
reuses corec as a library — no codegen duplication.

### Cargo.toml

```toml
[package]
name = "<program>-domains"
version = "0.1.0"
edition = "2021"

[dependencies]
rkyv = { version = "0.8", default-features = false, features = [
    "little_endian", "alloc"
] }
```

No other dependencies. The domain crate is self-contained.
It doesn't depend on aski or aski-core — those contracts
are upstream. The domain crate IS the downstream contract.


## What domainc does NOT generate

- **Trait definitions** — semac. Needs method body compilation.
- **Trait implementations** — semac. Expression compilation.
- **Process bodies** — semac. Expression compilation.
- **FFI extern blocks** — semac.
- **Any .sema binary** — semac only.


## TypeExpr mapping

aski's TypeExpr has 18 variants. domainc only encounters a
subset of them in domain definitions (the others appear in
method signatures, expressions, etc.):

| TypeExpr variant | Where it appears | domainc handling |
|---|---|---|
| `Named(TypeName)` | Field types, variant payloads | Map via `map_primitive`, else pass through |
| `Application(TypeApplication)` | `[Vec Item]`, `[Option X]` | Emit as `Vec<Item>`, `Option<X>`, etc. |
| `Param(TypeParamName)` | Generic params `$Value` | Emit as Rust type parameter |
| `BoundedParam { bounds }` | `$Clone&Debug` | Emit as bounded generic |
| `SelfType` | Recursive reference | Emit as `Box<Self>` |
| `Ref { inner }` | Borrow | Emit as `&inner` |
| `MutRef { inner }` | Mut borrow | Emit as `&mut inner` |
| `Boxed(inner)` | Box indirection | Emit as `Box<inner>` |
| `Tuple { elements }` | Tuple types | Emit as `(A, B, ...)` |
| `Unit` | Unit type | Emit as `()` |

The remaining TypeExpr variants (FnPtr, DynTrait, ImplTrait,
InstanceRef, QualifiedPath, Array, Slice, Never) appear only
in method signatures and expressions — domainc never
encounters them in domain definitions.


## GenericParamDef handling

```aski
(Option (Some $Value) None)
(Result (Ok $Output) (Err $Failure))
```

domainc reads `generic_params: Vec<GenericParamDef>` from
EnumDef/StructDef/NewtypeDef. Each GenericParamDef has:
- `name: TypeParamName`
- `bounds: Vec<TraitBound>`
- `default: Option<TypeExpr>`

Generated output:

```rust
pub enum Option<Value> { Some(Value), None_ }
pub enum Result<Output, Failure> { Ok(Output), Err(Failure) }
```

Bounded params:

```aski
(Serialize (Data $Clone&Debug) Empty)
```

```rust
pub enum Serialize<CloneDebug: Clone + Debug> {
    Data(CloneDebug),
    Empty,
}
```


## corec as library

domainc does NOT lex or parse .aski text. It reads rkyv
binary. But it needs corec's codegen to emit Rust.

### What domainc uses from corec

domainc constructs corec's internal types (Module, Domain,
EnumDef, StructDef, EnumVariant, StructField, TypeExpr) and
passes them to `Codegen::emit_module()`.

The translation is:

```
aski::ArchivedEnumDef    → corec::parse::EnumDef
aski::ArchivedStructDef  → corec::parse::StructDef
aski::ArchivedEnumChild  → corec::parse::EnumVariant
aski::ArchivedStructChild → corec::parse::StructField
aski::ArchivedTypeExpr   → corec::parse::TypeExpr
```

This is a straightforward conversion. Each aski archived type
maps to one corec type. domainc walks the archived tree,
constructs corec types, feeds them to codegen.

### Required corec changes

corec is currently binary-only. To use it as a library:

1. Add `[lib]` to Cargo.toml
2. Create src/lib.rs re-exporting pub mod lex, parse, codegen
3. Make codegen methods pub (they're currently private):
   - `emit_enum`, `emit_struct`, `emit_variant`, `emit_field`
   - `emit_derives`, `type_to_rust`, `needs_omit_bounds`
   - `map_primitive`, `escape_variant`, `to_snake`

No code changes needed in parse.rs or lex.rs — types and
Parser are already pub.

### What domainc adds beyond corec

corec's codegen handles the basic emit pattern. domainc adds:

1. **Scope index enums** — not in .aski source, synthesized
   by domainc from the module's domain inventory.

2. **Const generation** — corec doesn't handle consts (they
   don't appear in .aski domain files). domainc emits
   `pub const NAME: type = value;` directly.

3. **Nested type extraction** — corec handles nested
   enum/struct variants in its parse→emit pipeline. domainc
   needs to extract nested types and emit them at top level
   with scoped names.

4. **Cargo.toml generation** — corec outputs a single .rs
   file. domainc outputs a complete crate.

5. **Generic parameter synthesis** — corec doesn't see `$`
   sigils (they're aski syntax, not .aski domain syntax).
   domainc reads GenericParamDef from the parse tree and
   emits Rust generics.


## Nix integration

```
askic output ──→ domainc ──→ domain crate (nix derivation)
                                   │
                              semac (depends on domain crate via flake-crates/)
```

### domainc's flake.nix

```nix
inputs:
  corec       — library dependency (Rust crate)
  aski        — rkyv contract types (for deserialization)
```

domainc depends on:
- **corec** as a Rust library (via flake-crates/corec)
- **aski** as a Rust library (via flake-crates/aski)
- **rkyv** (Cargo dependency)

### How semac uses domainc's output

semac's flake.nix takes domainc's output (a source tree)
and copies it into flake-crates/:

```nix
domainc-output = domainc.packages.${system}.domains;

postUnpack = ''
  cp -r ${domainc-output} $sourceRoot/flake-crates/<program>-domains
'';
```

semac then depends on it as a normal Cargo path dependency.


## CLI interface

```
domainc <input.rkyv> <output-dir>
```

One input (rkyv file from askic). One output (crate directory).

The program name is derived from the module name in the parse
tree (first RootChild::Module). Module name "Elements" with
conventional `-domains` suffix → crate name `elements-domains`.


## Open questions

### 1. Module name → crate name

The module convention is: repo name minus `-aski` suffix.
`astro-aski` → module `Astro` → domains crate `astro-domains`.

But the module name in the parse tree is PascalCase (`Astro`).
The crate name should be lowercase-kebab (`astro-domains`).

**Proposed:** domainc lowercases the module name for the crate
name. `Elements` → `elements-domains`.

### 2. Multi-module programs

The current spec assumes one module per .aski file, one .rkyv
per askic invocation. If a program has imports
(`[Core ParseState Token]`), those imported types come from
other modules' domain crates.

domainc processes ONE module at a time. Each module gets its
own domain crate. Cross-module references become cross-crate
dependencies — but domainc does NOT wire those up. That's
semac's job (it sees the full program scope).

**For now:** domainc generates types for one module. Imported
types from other modules are left as-is in TypeExpr::Named —
they resolve at semac time, not domainc time.

### 3. Self-typed fields and circular references

```aski
{Drawing (Shapes [Vec Shape]) Name}
```

`Name` is self-typed: field `name` has type `Name`. But
`Name` might not be defined in the same module. domainc emits
it literally — `pub name: Name`. If `Name` is undefined,
Rust compilation of the domain crate fails, which is correct
(the .aski source is wrong).

### 4. Nested type naming

```aski
(Token (Ident String) (| Delimiter LParen RParen |) Newline)
```

The nested enum `Delimiter` inside `Token` needs a top-level
name. Convention: parent name + nested name → `TokenDelimiter`.

But what if there's already a top-level `TokenDelimiter`?
This would be a name collision in the .aski source — the
programmer's bug, not domainc's.

**Proposed:** domainc prefixes nested type names with parent
name. Collision = .aski source error.

### 5. Visibility

EnumDef, StructDef, NewtypeDef all carry `visibility:
Visibility` (Public/Private). Currently corec emits
everything as `pub`. Should domainc respect visibility?

Visibility is a semac concern (scope resolution). The domain
crate needs all types pub for semac and rsc to access them.
Visibility enforcement happens at the sema level, not at the
Rust crate level.

**Proposed:** domainc emits everything as `pub`. Visibility
is a sema concern, not a Rust concern.

### 6. DeriveAttr

EnumDef/StructDef/NewtypeDef carry `derives: Vec<DeriveAttr>`.
DeriveAttr has variants: Debug, Clone, Copy, PartialEq, Eq,
PartialOrd, Ord, Hash, Default, RkyvArchive, RkyvSerialize,
RkyvDeserialize.

domainc already adds rkyv derives unconditionally. Should it
also respect user-specified derives from the parse tree?

**Proposed:** domainc adds rkyv derives (always) plus any
user-specified derives from the DeriveAttr list. The two sets
are merged (no duplicates).


## Summary

```
INPUT:  Vec<RootChild> as rkyv binary (.rkyv file from askic)
OUTPUT: Rust crate with rkyv derives (<program>-domains/)

READS:  ModuleDef, EnumDef, StructDef, NewtypeDef, ConstDef, TraitDeclDef
SKIPS:  TraitImpl, Ffi, Process (those have expressions → semac)
USES:   corec as library (codegen only, not lex/parse)
ADDS:   scope index enums, const generation, nested extraction, generics, Cargo.toml
```
