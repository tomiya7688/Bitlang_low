# Bitlang Low

Bitlang Low is a low-level, C-like **programming language** used after `Bitlang Explicit` in the Bitlang toolchain. It is also a valid language for direct human-authored programs; it is not merely an internal compiler IR.

Its primary goals are:

- provide a stable low-level language boundary between the Bitlang Lowerer and backends,
- preserve a structure that can be translated to C mechanically,
- remain suitable for optimization and static analysis,
- remove high-level ambiguity before backend translation,
- keep the language itself directly writable and readable enough for inspection, auditing, and debugging; compiler-generated output may prioritize determinism over elegance, but must remain meaningfully inspectable.

## Canonical boundary before C

For the C backend path, Bitlang Low is the final canonical Bitlang language representation before translation into C.

```text
Bitlang
    -> Bitlang Explicit
    -> Bitlang Low   <- final canonical/auditable Bitlang form for C output
    -> C
    -> native toolchain
```

Because of this role, Bitlang Low must remain understandable to a programmer who needs to inspect what the Bitlang toolchain actually decided before C translation.

Compiler-generated Low may use deterministic generated names and expanded low-level control flow, but it must not become an opaque private encoding whose meaning can only be understood by the compiler implementation.

For assembly-oriented paths, another lower stage may follow Bitlang Low, such as Bitlang VM Assembly. Assembly is inherently less convenient to inspect, so Bitlang Low remains the practical human-readable audit point even when it is not the absolute last intermediate representation.

## Human-authored Bitlang Low

Bitlang Low is intentionally writable by programmers.

The normal Bitlang pipeline may generate Bitlang Low automatically, but generated origin is not a validity requirement. A programmer may write a Bitlang Low translation unit directly and compile/translate it through the Bitlang Low validator/backend pipeline.

Conceptually:

```text
Bitlang source
    -> Bitlang Explicit
    -> Bitlang Lowerer
    -> Bitlang Low
    -> backend

or

human-authored Bitlang Low
    -> Bitlang Low validation
    -> backend
```

Directly written Bitlang Low must satisfy the same Bitlang Low language rules as compiler-generated Bitlang Low. It does not need a fictitious upstream Bitlang/Explicit source artifact merely to be considered valid.

This is important to the Bitlang family principle that programmers may choose the abstraction level and writing style appropriate to their task.

## Position in the toolchain

```text
Bitlang source
    -> Bitlang Preprocessor
Bitlang Explicit
    -> Bitlang Lowerer
Bitlang Low
    -> Bitlang C Backend / other backends
C / other backend representations
```

## Tool naming

The stage tools use role-based names:

- **Bitlang Lowerer**: transforms the fully explicit Bitlang form into Bitlang Low.
- **Bitlang C Backend**: translates Bitlang Low into C.
- Other backend translators follow the same `Bitlang <Target> Backend` naming pattern.

`Bitlang Low compiler` and `Bitlang compiled compiler` are not canonical tool names.

## Design rule

Where a C language construct can be adopted without conflicting with Bitlang Low's safety, determinism, or backend requirements, Bitlang Low follows the C form directly.

Differences from C are specified explicitly rather than inventing alternative syntax unnecessarily.

Bitlang-native semantics are not weakened merely to fit a C primitive or C undefined/implementation-defined behavior. When direct C representation is insufficient, the backend must use explicit lowering, checks, helpers, carrier representations, or another defined mechanism.

A major intentional exception is the numeric type system. Bitlang Low retains Bitlang's canonical radix-and-bit-width type representation, such as `Int10x32` and `Uint10x64`, rather than reverting to target-dependent C primitive widths.

## Specification

- [`docs/language-spec.md`](docs/language-spec.md) — main language specification and C-like surface
- [`docs/numeric-types.md`](docs/numeric-types.md) — Bitlang-native numeric type semantics and C lowering rules
- [`docs/semantic-lowering.md`](docs/semantic-lowering.md) — Bitlang-specific property, ownership, lifetime, reference, class/module, and functional lowering rules
- [`docs/borrow-state-lowering.md`](docs/borrow-state-lowering.md) — borrow-state lowering requirements
- [`docs/target-model.md`](docs/target-model.md) — target descriptor / C and VM physical data-model contract
- [`docs/coverage-audit.md`](docs/coverage-audit.md) — Bitlang / C / VM semantic coverage audit
- [`docs/open-decisions.md`](docs/open-decisions.md) — remaining decisions that cannot simply inherit C behavior
