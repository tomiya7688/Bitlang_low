# Bitlang Low Open Semantic Decisions

Status: Draft

This document lists only decisions that cannot be inherited mechanically from ordinary C behavior and are not already fixed by Bitlang / Bitlang Preprocessed semantics.

C-compatible syntax and behavior do not need a separate Bitlang Low decision unless they conflict with Bitlang semantics.

## 1. Concurrency and atomics

If Bitlang Low exposes `volatile`, atomic operations, threads, or shared-memory concurrency, their memory model must be defined explicitly.

Ordinary C syntax may be reused where compatible, but the Bitlang contract must establish which C/C11/C23 memory-model behavior is intentionally inherited and which behavior is restricted.

## 2. Explicit layout / bit-field facility

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
