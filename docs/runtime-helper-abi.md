# Bitlang Low Runtime Helper ABI

Status: Draft / canonical semantic helper contract

Bitlang Low uses explicit runtime/helper operations when behavior cannot be represented as an ordinary target-independent instruction or declaration.

These helpers are semantic operations. A backend may inline them or map them to target/runtime functions, but it must preserve the contract.

## 1. Byte storage type

Raw allocated storage is addressed as:

```text
Ptr<Uint2x8>
```

where `Uint2x8` is the canonical 8-bit byte value type.

Physical allocation sizes and alignments use `Size`.

## 2. Allocation

The ordinary trapping allocation operation is conceptually:

```text
memory_alloc(byte_count: Size, alignment: Size)
    -> Owned unnullable Ptr<Uint2x8>
```

Rules:

- `byte_count > 0`;
- `alignment > 0`;
- alignment must be a power of two;
- the selected target/runtime must support or emulate the requested alignment;
- invalid size/alignment known statically is a compile error;
- invalid size/alignment discovered at runtime traps;
- allocation failure / out-of-memory traps;
- success creates a fresh live allocation/provenance identity and transfers ownership to the returned pointer.

A typed allocation is lowered mechanically by computing a checked physical byte size/alignment and explicitly converting the resulting byte pointer to the required `Ptr<T>`.

The size calculation itself is checked; overflow while calculating `sizeof(T) * count` is an error/trap and must not wrap.

## 3. Checked allocation

Recoverable allocation failure uses a distinct non-trapping operation:

```text
memory_try_alloc(byte_count: Size, alignment: Size)
    -> Owned nullable Ptr<Uint2x8>
```

- non-null means allocation succeeded;
- null means allocation failed because storage was unavailable;
- invalid arguments are still semantic errors/traps and are not converted into an OOM-null result;
- `byte_count == 0` is invalid rather than being given target-dependent C `malloc(0)` behavior.

The explicit nullable result keeps the checked/trapping distinction visible in canonical Low.

## 4. Zero-length containers

Core allocation does not accept a zero byte count.

A zero-length array/string/container does not need a heap allocation. Its descriptor may use the type's defined empty representation, including a null data pointer where that type permits it, while retaining length zero.

This prevents target-specific zero-size allocator behavior from becoming Low semantics.

## 5. Release

The canonical release operation is conceptually:

```text
memory_free(pointer)
```

It accepts a releasable owned allocation-base pointer.

Rules:

- null is a no-op when the pointer type/property permits null;
- a non-null pointer must denote the live base of an allocation compatible with this allocator/runtime domain;
- interior, one-past, already-released, borrowed-only, or otherwise invalid release is an error;
- a valid release consumes the live allocation ownership and transitions the resource to released state;
- the allocator/runtime is responsible for any physical allocation metadata needed to release the block.

Low does not require size/alignment arguments on release. A backend allocator that requires them must retain or reconstruct the required metadata as part of its implementation.

## 6. Reallocation

Reallocation is **not a core Low primitive**.

Resizable containers and libraries implement growth using explicit checked size calculation, allocation/try-allocation, copy/move of valid contents, and release.

A backend may optimize that sequence to a native realloc-like facility only when all observable ownership, failure, alignment, and pointer-invalidation semantics remain identical.

## 7. Trap helper

Low's semantic trap is non-returning.

Backends may lower it to a target trap instruction, VM trap object, runtime abort helper, or equivalent immediate-failure mechanism.

Trap reason categories must preserve at least the distinction needed for diagnostics/conformance testing, including:

- overflow;
- division/remainder by zero;
- invalid shift;
- bounds violation;
- invalid conversion;
- initialization re-entry/cycle discovered at runtime;
- allocation failure;
- another explicitly defined runtime semantic violation.

Source/debug location may be carried as metadata but is not required to affect program semantics.

## 8. Other runtime helpers

Arbitrary-width numeric operations, string operations, task-local retention support, and other helper-backed Low operations follow the same rules:

- helper use is explicit in canonical Low or backend-required reachable support;
- helper ABI comes from the selected target descriptor/runtime profile;
- unused helper code is removable by reachability;
- host runtime behavior is not implicitly adopted as Low semantics.

## 9. C backend mapping

The C backend may implement allocation using `malloc`, `aligned_alloc`, platform allocation APIs, or generated wrappers.

Direct mapping is allowed only when the selected C target/API matches the Low contract. In particular the backend must normalize:

- OOM behavior;
- alignment requirements;
- zero-size behavior;
- ownership/release behavior.

The generated C program must not expose implementation-defined or unspecified allocator behavior as Low semantics.

## 10. Bitlang VM mapping

The VM uses guest allocation identities/addresses inside the VM target address space.

Host Go pointers are not guest allocation addresses.

The VM backend/runtime may use an internal heap manager or another implementation, but it must preserve the same alloc/try-alloc/free/trap contract and the VM target's `Size`/alignment rules.
