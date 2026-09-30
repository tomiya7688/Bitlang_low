# Bitlang Low Target Model

Status: Draft / required backend contract

Bitlang Low keeps semantic language meaning separate from target-dependent physical representation.

A validated Low program may be translated to C, Bitlang VM Assembly, or another backend. Any operation whose result depends on physical storage or ABI layout is evaluated against an explicit **target descriptor** rather than against the machine that happens to run the compiler.

## 1. Target-parametric language boundary

The following remain semantic and target-independent until backend selection/lowering:

- canonical Bitlang numeric type identity, radix, semantic width, and signedness;
- ownership/release effects already made explicit by lowering;
- trap/check behavior;
- Ptr/Ref semantic guarantees;
- deterministic control flow and evaluation order.

The following are target-dependent:

- physical storage size;
- alignment;
- field offsets and ordinary struct padding;
- pointer/address width and representation;
- byte order of physical multi-byte objects;
- native numeric carrier selection;
- calling convention and stack/frame details;
- function/code address representation;
- backend/runtime helper ABI.

## 2. Bitlang byte

A Bitlang byte is exactly **8 bits**.

This is the unit used by Bitlang `xx<N>` source notation and by Bitlang Low byte-counting storage/layout operations.

A backend whose native C implementation or machine ABI does not use an 8-bit byte must not silently reinterpret a Bitlang byte as the host C `char` unit. It must either provide an explicit representation layer that preserves 8-bit Bitlang byte semantics or reject operations/ABI contracts that it cannot preserve.

The required Bitlang VM architecture families (x64, ARM64, and RISC-V) use an 8-bit addressable byte and therefore map directly.

## 3. Target-sized Low scalar types

Bitlang Low defines target-sized physical-count types that are intentionally separate from Bitlang's canonical arbitrary-width numeric families.

### Size

`Size` is an unsigned target-sized integer used for physical sizes and non-negative physical counts.

Its exact width/range is supplied by the selected target descriptor.

Typical uses include:

- `sizeof` results;
- `alignof` results;
- field byte offsets;
- dynamic storage allocation sizes;
- physical array element counts/strides where a target-sized count is required.

`Size` is not an alias for `Uint10x32`, `Uint10x64`, or any other canonical Bitlang integer type. Conversion between `Size` and a canonical numeric type is explicit and checked when the target range may not fit.

For the C backend, `Size` normally maps to the selected target's `size_t` when that type preserves the required contract.

### Offset

`Offset` is a signed target-sized integer used for pointer differences and signed physical offsets.

Its exact width/range is supplied by the selected target descriptor.

Pointer subtraction returns `Offset`. Ordinary pointer arithmetic uses an explicit `Offset` (or a value explicitly converted to it).

`Offset` is distinct from both `Size` and ordinary canonical Bitlang integers.

For the C backend, `Offset` normally maps to the selected target's `ptrdiff_t` when suitable.

### Address remains distinct

`Address`, `Size`, and `Offset` are three different Low types even when the selected target represents all of them with the same machine width.

- `Address` represents a raw target address value.
- `Size` represents a non-negative size/count.
- `Offset` represents a signed difference/offset.

No implicit conversion exists among them.

## 4. Required target descriptor information

A target descriptor must provide enough information to determine at least:

- target/profile identity;
- data-address width and representation;
- `Size` width/range;
- `Offset` width/range;
- function/code-address representation where it differs;
- semantic null lowering for pointer-shaped values;
- byte order;
- supported native load/store widths;
- natural and maximum alignment rules;
- canonical numeric carrier availability, size, and alignment;
- array element stride/layout rules;
- ordinary struct layout rules;
- stack alignment and call-frame requirements;
- parameter passing;
- return-value passing;
- runtime/helper call convention;
- trap/halt lowering requirements.

Backend-specific extensions may add more information, but Low semantics must not be reconstructed from undocumented backend defaults.

## 5. Physical queries

`sizeof`, `alignof`, field-offset queries, `Address` width, `Size`/`Offset` width, ordinary struct layout, and array stride are resolved using the selected target descriptor.

Therefore the same portable Low source may produce different physical sizes and offsets for different targets while preserving the same language-level semantics.

Code requiring a stable cross-target binary layout must use the explicit-layout facility rather than ordinary target-native layout.

## 6. C backend target descriptor

The Bitlang C Backend uses the **selected target C ABI**, not the host ABI of the process running the translator.

Direct use of a C primitive, pointer, struct, or calling convention is valid only when the selected C target preserves the corresponding Low contract.

When C cannot express a required Low behavior directly, the backend must use explicit helper types/functions, generated checks/control flow, or reject the target. It must not weaken Low semantics to match a convenient C representation.

For C23-capable targets, features such as two's-complement signed integers and fixed-underlying-type enumerations may permit more direct lowering, but Low does not require the generated C dialect itself to be C23 when an equivalent defined lowering is available.

## 7. Bitlang VM target descriptor

The Bitlang VM is a target in its own right.

Its target descriptor is defined by the Bitlang VM ABI/layout contract and must not inherit:

- the Go host's pointer size;
- Go struct layout;
- the host C ABI;
- the CPU executing the Go VM.

The Bitlang VM Backend and VM runtime must consume compatible descriptor/ABI fixtures so that `sizeof`, alignment, pointer representation, call/return behavior, and guest memory layout agree exactly.

## 8. Build reproducibility

The selected target descriptor/profile is part of build identity.

Diagnostics, canonical build metadata, conformance tests, and backend caches should identify the target profile used to resolve target-dependent Low behavior.
