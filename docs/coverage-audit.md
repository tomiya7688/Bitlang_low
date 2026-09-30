# Bitlang Low Coverage Audit

Status: Living design audit

This document tracks whether Bitlang Low is sufficient to preserve the semantics required by Bitlang and by the Bitlang VM path while remaining mechanically translatable to C.

## Coverage matrix

| Area | Source of requirement | Low status |
| --- | --- | --- |
| Canonical arbitrary-width integers | Bitlang / VM | Covered; fixed-width signed representation and div/mod rules are specified in `numeric-types.md`. |
| Floating-point semantic format/rounding | Bitlang | **Blocked upstream**: Bitlang defines radix + bit width but not the complete float format, rounding, NaN/Inf semantics. |
| Boolean/condition semantics | Bitlang / C / VM | Covered: canonical `Bool`, no implicit scalar truthiness, conditions require `Bool`. |
| Checked/discard shift families | Bitlang | Covered; canonical operations preserve checked vs discard. C operators are only surface aliases. |
| Overflow / runtime failure | Bitlang / VM | Covered; static error or runtime trap, with explicit checked operations for recovery. |
| Array shape / runtime length | Bitlang / C / VM | Covered: fixed arrays are inline; non-fixed `Array<T>` is explicit pointer + `Size` length, non-resizable core descriptor. |
| Array bounds | Bitlang / VM | Covered. |
| Ordinary struct layout | C / VM | Covered as target-native layout. |
| Target data model | C / VM | Covered by `target-model.md`. |
| Ptr / Ref / Address | Bitlang / C / VM | Covered semantically; canonical spelling is retained in generated Low. |
| Ownership / release / destruction safety | Bitlang | Covered. |
| Retention domains | Bitlang | Covered by semantic lowering rules for process/thread/task retention. |
| Static/lazy initialization order | Bitlang | Covered by deterministic dependency-graph lowering. |
| Finalization order | Bitlang | Covered; reverse actual successful initialization order. |
| Generics | Bitlang | No Low runtime feature required; resolved before Bitlang Explicit. |
| Import capabilities | Bitlang | No Low runtime feature required; capability violations are resolved/validated before Low. |
| Namespace mounts / project inheritance | Bitlang | No Low runtime feature required; consumed before Explicit/Low. |
| Language-version compatibility | Bitlang | No Low runtime feature required; compatibility frontend disappears before Explicit. |
| Standard-library reachability | Bitlang | Low receives only reachable declarations/helpers; no monolithic runtime requirement. |
| Residual GC | Bitlang | Not a Low core memory model; if selected it appears as explicit reachable runtime/library support. |
| Closures / first-class functions | Bitlang | Covered by explicit code target + environment lowering. |
| Sum/variant/pattern matching | Bitlang | High-level form eliminated; exact optional payload representation remains an implementation/layout choice. |
| Enum physical representation | C / VM | Covered with explicit canonical underlying Bitlang integer type. |
| Strings / characters | Bitlang | **Open**: encoding and concrete low-level representation are not yet defined upstream/Low. |
| External/native ABI | C | Open. |
| Runtime helper / allocator ABI | C / VM | Open at the ABI/signature level; allocation/release must be explicit, never implicit. |
| Concurrency / atomics memory model | Bitlang / C / VM | Open. Retention domains alone do not define shared-memory concurrency. |
| Exact packed / bit-field layout | C / hardware / VM | Open. |
| Untyped C varargs / foreign varargs | C ABI | Not part of ordinary Low core; handle only through the explicit external ABI/FFI contract. |

## Review rule

When a new Bitlang semantic property or VM execution requirement is added, this matrix should be checked before implementation. A requirement must either:

1. disappear before Low with proof that no runtime/backend meaning remains;
2. have an explicit Low representation/lowering rule; or
3. be recorded as an open blocker with an owning specification repository.

The C backend must not be used as an implicit specification for an uncovered area.
