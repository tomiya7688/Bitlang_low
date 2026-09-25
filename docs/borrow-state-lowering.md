# Borrow-state lowering

This document defines how borrow-state semantics arriving from Bitlang Preprocessed are handled when producing **Bitlang Low**.

Source-facing borrow syntax and preprocessing belong to `tomiya7688/Bitlang`. Canonical borrow-state properties belong to `tomiya7688/Bitlang-Explicit`.

## Input contract

Before Bitlang Low is emitted, the incoming Bitlang Preprocessed program has already resolved applicable borrow state explicitly as one of:

```text
Unborrowed
Shared_borrowed
Exclusive_borrowed
```

The lowering stage must use that resolved state when validating access, aliasing, lifetime, move, and release behavior.

## Bitlang Low representation

Bitlang Low is not required to retain these high-level property names verbatim.

Borrow-state information may be consumed during lowering and represented through ordinary low-level constructs such as:

- explicit pointers or references,
- explicit function parameters,
- lifetime-valid access paths,
- restricted aliasing established by analysis,
- explicit move or ownership transfer operations,
- explicit release/destruction operations,
- compiler-generated temporaries or helper calls where required.

The important requirement is preservation of validated behavior, not preservation of the original property spelling.

## Required validation before or during lowering

A program must not be lowered as valid Bitlang Low when analysis proves any of the following:

- a resource is released or destroyed while an incompatible active borrow remains,
- an exclusive borrow conflicts with another live borrow or access path,
- a borrowed value outlives the resource on which it depends,
- a move invalidates an active borrow that is still used,
- a preprocessor override produced a borrow state that contradicts a provable live dependency.

Suspicious cases that cannot be proven invalid may have been diagnosed earlier as warnings. Bitlang Low itself should receive a deterministic resolved program rather than re-run source-level heuristic inference.

## Optimization

After validity has been established, borrow-state metadata that has no remaining runtime effect may be discarded. Optimizations may eliminate dead handles, temporary references, or analysis-only state as long as observable behavior and memory-safety constraints are preserved.
