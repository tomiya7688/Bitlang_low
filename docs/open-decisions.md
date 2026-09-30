# Bitlang Low Open Semantic Decisions

Status: Draft

This document lists only decisions that cannot be inherited mechanically from ordinary C behavior and are not already fixed by Bitlang / Bitlang Preprocessed semantics.

C-compatible syntax and behavior do not need a separate Bitlang Low decision unless they conflict with Bitlang semantics.

## 1. External ABI and symbol contract

Internal generated code may use backend-private representations, but interoperability with C or other native code requires explicit rules for:

- calling convention,
- exported symbol naming/mangling,
- parameter and return layout,
- arbitrary-bit-width numeric types,
- strings,
- Optional/nullable representations,
- structs and alignment,
- ownership responsibility across the boundary,
- error/failure propagation across the boundary.

Internal compilation does not need to use the external ABI representation unless a symbol crosses an ABI boundary.

## 2. Strings and characters

Bitlang defines `Str` and bounded string forms independently from C strings.

Bitlang Low still needs a concrete low-level semantic contract for:

- character encoding,
- length unit,
- whether storage is null-terminated,
- whether length is stored explicitly,
- representation of bounded and unbounded strings,
- ownership of string storage,
- C-string conversion behavior,
- `Char` / `Str1x1` backend representation.

This must be decided before C ABI mapping can be stable.

## 3. Concurrency and atomics

If Bitlang Low exposes `volatile`, atomic operations, threads, or shared-memory concurrency, their memory model must be defined explicitly.

Ordinary C syntax may be reused where compatible, but the Bitlang contract must establish which C/C11/C23 memory-model behavior is intentionally inherited and which behavior is restricted.

## 4. Explicit layout / bit-field facility

Ordinary arbitrary-bit-width numeric values should not be represented as C bit-fields merely because their semantic width is unusual.

A separate exact-layout facility may still be useful for hardware registers, protocols, packed structs, and ABI-specific structures.

If added, it needs rules for:

- bit order,
- byte order,
- field offsets,
- cross-byte fields,
- signed interpretation,
- alignment and packing,
- backend support/fallback.

This should remain separate from the ordinary `Int<Radix>x<BitWidth>` / `Uint<Radix>x<BitWidth>` type system.


## 5. Runtime helper / allocator ABI

Bitlang Low requires explicit runtime/helper operations for behavior that is not represented as ordinary C syntax or a direct machine operation.

Allocation and release are never implicit Low behavior. The remaining contract must define the canonical helper/intrinsic ABI used for operations such as:

- heap allocation with explicit size/alignment;
- deallocation/release;
- optional reallocation;
- out-of-memory behavior and checked allocation;
- runtime trap support;
- arbitrary-width numeric helpers;
- task-local storage support;
- optional residual-collector integration.

The general failure model is already fixed: ordinary failing operations trap, while recoverable behavior uses an explicit checked/non-trapping operation.

The exact helper names, signatures, ownership of returned memory, zero-size allocation behavior, and C/VM helper mapping still need to be specified.
