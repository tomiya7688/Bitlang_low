# Bitlang Low C / Native ABI Interoperability

Status: Draft / explicit boundary contract

Ordinary Bitlang Low declarations are **not** automatically C/native ABI declarations.

Bitlang module visibility, Low program symbol identity, and native ABI exposure are separate concepts.

## 1. Explicit C ABI boundary

A C ABI import is declared explicitly:

```text
extern(c, "c_symbol_name") ReturnType local_name(ParameterTypes...);
```

A C ABI export is declared explicitly:

```text
export(c, "c_symbol_name") ReturnType local_name(ParameterTypes...) {
    ...
}
```

The external link-name string is case-sensitive and is the exact native symbol contract.

The Low local identifier remains subject to Bitlang/Low case-insensitive identifier rules.

An external ABI declaration without an explicit link name is not canonical generated Low.

## 2. Calling convention

`c` selects the default calling convention of the selected **target C ABI profile**, not the compiler host.

Additional calling-convention tags may be added later as explicit ABI profiles. Unsupported ABI tags are compile/backend errors.

The target descriptor supplies stack alignment, register/stack argument classification, return conventions, aggregate rules, and other physical ABI details.

## 3. ABI-safe direct types

A Low type may cross the C boundary directly only when the selected C target profile proves an ABI representation with matching semantics, size, alignment, and calling convention classification.

Common direct candidates include:

- compatible fixed-width `Int` / `Uint` types;
- `Size` and `Offset`;
- `Address`;
- raw `Ptr<T>` when the pointee ABI contract is valid;
- `Bool` when the target C boolean ABI is explicitly compatible;
- floating-point types only after their Bitlang semantics are defined and a matching C ABI type exists;
- structs whose complete recursively reachable field/layout contract is C-ABI-compatible.

A canonical Low type is never replaced merely because a roughly similar C primitive exists.

## 4. Types requiring explicit boundary representation

The following do not automatically cross the C ABI in their ordinary Low representation:

- `Ref<T>`;
- runtime-length `Array<T>`;
- `Str`;
- Optional/presence representations;
- arbitrary-bit numeric types without a proven matching C ABI type;
- helper/software numeric representations;
- closure environments / captured function values;
- variant/result values without an explicit ABI struct contract.

Such values use an explicit wrapper/adaptor signature composed of ABI-safe types.

Examples include:

```text
Str
    -> Ptr<Uint2x8> + Size

Array<T>
    -> Ptr<T> + Size

Optional T
    -> explicit tag + payload representation
```

These are examples of ABI adapters, not implicit language conversions.

## 5. Ref and foreign pointers

A raw pointer received from foreign C code does not automatically satisfy `Ref<T>` provenance, liveness, alignment, ownership, or non-null guarantees.

Foreign pointer parameters therefore enter as `Ptr<T>` / `Address` unless an explicit trusted/validated adapter establishes the stronger reference contract.

An exported wrapper must validate any dynamic conditions required before constructing a Low `Ref<T>`.

## 6. Ownership transfer

Ownership is part of the ABI semantic contract even though C itself cannot enforce it.

Parameter/return ownership properties determine whether a call:

- borrows a pointer for the duration of the call;
- transfers ownership into foreign code;
- returns newly owned storage;
- returns a borrowed view.

The Low compiler updates move/release state according to the declared boundary contract.

A C wrapper/header should document ownership direction generated from these properties.

## 7. Structs and enums

An ordinary Low struct may cross directly only when the selected target C ABI layout matches and every field has an ABI-safe representation.

Otherwise an explicit boundary struct/wrapper is required.

Low enums cross the ABI by their explicit underlying integer representation unless a selected C ABI profile explicitly guarantees an identical fixed-underlying enum representation.

## 8. Strings

Ordinary `Str` is not a C NUL-terminated string.

A C ABI wrapper may expose:

```text
data + byte_length
```

or may explicitly convert to/from a NUL-terminated C string according to the string interoperability rules.

No implicit `Str -> char*` conversion exists.

## 9. Optional and nullability

Presence and nullability remain distinct at the ABI boundary.

An `Optional nullable T` must not be collapsed into one C null pointer because that would lose the difference between absent and present-null.

Boundary wrappers use an explicit tag/payload representation where multiple semantic states exist.

## 10. C variadic functions

C-style variadic `...` is allowed only inside an explicit `extern(c,...)` ABI declaration.

Low does not apply C default argument promotions implicitly.

Every variadic argument must already be explicitly converted to the exact ABI type expected by the foreign function contract before the call.

Canonical generated Low should prefer typed wrappers around C varargs where practical.

Bitlang Low does not define a general native variadic function-export mechanism in the core language.

## 11. Failure and unwinding

A Low trap is non-returning and does not become a C exception.

Low does not permit C++ exception unwinding, C `longjmp`, or another foreign non-local control transfer to cross a Low frame unless a separately specified adapter catches/translates it at the ABI boundary.

Recoverable errors cross the ABI only through explicit result/status/out-parameter representations.

## 12. Backend support

The C backend emits the external symbol/calling convention directly or via generated wrappers.

A non-C backend, including Bitlang VM, may reject `extern(c)` / `export(c)` when its selected target profile does not provide a C/native bridge.

A future VM host ABI may provide explicit adapter profiles without changing ordinary Low semantics.
