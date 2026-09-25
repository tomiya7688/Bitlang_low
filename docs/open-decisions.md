# Bitlang Low Open Semantic Decisions

Status: Draft

This document lists only decisions that cannot be inherited mechanically from ordinary C behavior and are not already fixed by Bitlang / Bitlang Preprocessed semantics.

C-compatible syntax and behavior do not need a separate Bitlang Low decision unless they conflict with Bitlang semantics.

## 1. Enum underlying representation

C-like enum syntax exists, but Bitlang Low still needs a deterministic rule for the underlying numeric type when layout or ABI matters.

Options include requiring an explicit canonical Bitlang integer type, inferring the smallest representable Bitlang integer type, or adopting another fixed rule.

The backend must not silently choose a different semantic range merely because a C compiler chooses a particular enum representation.

## 2. External ABI and symbol contract

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

## 3. Strings and characters

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

## 4. Static/module initialization order

Static and module lifetime are defined semantically, but initialization/destruction order across declarations and modules still needs a deterministic rule.

The C backend must not simply inherit whichever initialization ordering happens to result from translation-unit or linker behavior when Bitlang observable behavior depends on the order.

## 5. Concurrency and atomics

If Bitlang Low exposes `volatile`, atomic operations, threads, or shared-memory concurrency, their memory model must be defined explicitly.

Ordinary C syntax may be reused where compatible, but the Bitlang contract must establish which C/C11/C23 memory-model behavior is intentionally inherited and which behavior is restricted.

## 6. Exact Ptr/Ref textual representation

The semantic distinction between `Ptr<T>` and `Ref<T>` is fixed, but Bitlang Low still needs a final textual representation if both remain visible after lowering.

Possible approaches include:

- retain `Ptr<T>` / `Ref<T>` as canonical type spellings,
- use C-like pointer spelling plus an explicit reference qualifier,
- erase `Ref<T>` only after all reference guarantees have been statically discharged.

This is primarily a Bitlang Low syntax/IR readability decision; it must not change the already-defined semantics.

## 7. Explicit layout / bit-field facility

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
