# Bitlang Low Numeric Types

Status: Draft / inherited core rule

Bitlang Low does **not** define a separate numeric type system.

Integer and floating-point numeric semantics are inherited from Bitlang's canonical type system. C compatibility applies to syntax and low-level program structure; it does not replace Bitlang numeric types with C primitive types.

This is a foundational exception to the general rule that C-compatible syntax and semantics should be reused where possible.

## Canonical type-name representation

For numeric types that carry both a radix/base and a bit width, the canonical form is:

```text
<TypeName><Radix>x<BitWidth>
```

Examples:

```text
Int10x32
Uint10x32
Int2x8
Int16x64
Float10x32
Float10x64
```

The radix and bit width are part of the type itself. They must not be reconstructed from a target C implementation's `int`, `long`, `float`, `double`, pointer size, or ABI defaults.

## Relationship to Bitlang source and preprocessed

Source-level shorthand, defaults, inferred types, and explicitly configured conversion automation are resolved before Bitlang Low is produced.

For example, Bitlang source may contain:

```text
int value
```

The canonical Bitlang representation is:

```text
Int10x32 value
```

Bitlang Low retains the canonical type rather than converting it back into an ambiguous C primitive type.

Consequently compiler-generated Bitlang Low should use forms such as:

```c
Int10x32 value;
Uint10x64 size;
Float10x32 ratio;
```

rather than using C primitive spellings as the semantic type:

```c
int value;
unsigned long size;
float ratio;
```

The declaration grammar remains C-like; the numeric type system remains Bitlang's.

## Signed and unsigned integers

`Int` is signed.

`Uint` is unsigned.

Signedness is explicit in the canonical type and is not inferred from target ABI defaults.

## Floating-point types

Floating-point types follow the same Bitlang canonical representation rule as integers when radix and bit width are meaningful:

```text
<TypeName><Radix>x<BitWidth>
```

The radix and bit width are therefore explicit type information for floating-point values as well.

If the radix is omitted at Bitlang source level, Bitlang's default radix is resolved before canonical Bitlang Low output is produced.

Bitlang Low does not redefine floating-point types according to C's `float`, `double`, or `long double`. Those are possible backend representations only.

Any floating-point behavior already defined by the Bitlang type system is inherited unchanged by Bitlang Low. Backend-specific implementation details remain the responsibility of lowering.

## Radix is semantic

Radix is part of the type, not only literal or display formatting.

For example:

```text
Int2x32
Int8x32
Int10x32
Int16x32
```

are distinct canonical types.

The same principle applies to other numeric families that carry radix information.

Values of different radix types are not directly compatible operands merely because they have the same bit width and numeric value.

A representation conversion must already be explicit, normally through Bitlang's pulse semantics, before incompatible radix types are combined.

Bitlang Low preserves this distinction until lowering has produced an equivalent backend representation.

## No C numeric promotions

Bitlang Low does not inherit C's integer promotions, usual arithmetic conversions, or implicit numeric conversions.

It must not silently:

- widen or narrow a numeric value,
- change signedness,
- change radix,
- choose a target-dependent primitive width,
- convert between integer and floating-point families,
- convert operands merely to make an expression compile.

Required conversions must already be explicit in the Bitlang Low representation.

## Cast and pulse

Bitlang's distinction between cast and pulse remains applicable.

A cast changes the semantic type of a value. A pulse changes representation while preserving the source value, including radix-oriented representation changes.

Bitlang Low must preserve the already-resolved operation rather than replacing it with a C implicit conversion.

## Overflow

Numeric overflow is an error by default, following Bitlang semantics.

Bitlang Low must not silently adopt C signed-overflow behavior, unsigned wraparound, or backend-specific floating-point behavior merely because C is a backend target.

If a distinct operation explicitly requests wrapping or another overflow policy, that operation must remain explicit through lowering.

## Unified C backend lowering rule

Integer and floating-point types follow the same backend principle.

Bitlang Low keeps the canonical Bitlang type unchanged. The C backend then chooses a physical C representation that preserves that type's semantics.

The backend should use a native C representation when the target provides one that is semantically compatible with the Bitlang type. Otherwise it must use a wider carrier, generated helper type, software representation, runtime/helper operation, or another defined lowering strategy.

This rule is intentionally common to all numeric families.

Examples of valid lowering strategies include:

```text
canonical Bitlang numeric type
    -> equivalent native C type, when one exists
    -> wider native carrier plus required checks, when sufficient
    -> generated/helper representation, when native C cannot preserve the semantics directly
```

The canonical Bitlang width is a semantic width. It does not have to equal the physical width of the C storage carrier.

Likewise, a C floating-point primitive may only be used when it satisfies the semantics required by the corresponding Bitlang floating-point type. If it does not, the backend must use another representation rather than silently changing the Bitlang type.

C bit-fields are not the general representation mechanism for Bitlang numeric types. They may be used only where an explicitly defined packed or layout-oriented lowering requires them.

The backend mapping is not allowed to redefine a Bitlang type according to whatever width, precision, range, signedness, or behavior a particular C implementation happens to assign to its primitive types.

## Core rule

Bitlang Low is C-like in syntax and low-level structure, but **numeric types and numeric semantics remain Bitlang-native**.

This applies uniformly to integer and floating-point numeric types and is a foundational language rule rather than an optional backend convention.
