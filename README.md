# Bitlang Low

Bitlang Low is the low-level, C-like language used after `Bitlang preprocessed` in the Bitlang toolchain.

Its primary goals are:

- provide a stable low-level language boundary between the Bitlang Lowerer and backends,
- preserve a structure that can be translated to C mechanically,
- remain suitable for optimization and static analysis,
- remove high-level ambiguity before backend translation,
- keep the language itself simple enough to be hand-written when useful, without requiring compiler-generated code to be human-friendly.

## Position in the toolchain

```text
Bitlang source
    -> Bitlang Preprocessor
Bitlang Preprocessed
    -> Bitlang Lowerer
Bitlang Low
    -> Bitlang C Backend / other backends
C / other backend representations
```

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
- [`docs/open-decisions.md`](docs/open-decisions.md) — remaining decisions that cannot simply inherit C behavior
