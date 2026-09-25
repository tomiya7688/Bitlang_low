# Bitlang Low Language Specification

Status: Draft

Bitlang Low is a C-like low-level language positioned between `Bitlang preprocessed` and backend representations such as C.

The baseline rule is simple: when ordinary C syntax and semantics can be reused without conflicting with Bitlang Low requirements, they are reused directly.

This document records the C-compatible surface and the Bitlang-specific rules that must survive lowering.

A foundational exception is the numeric type system: Bitlang Low inherits Bitlang's canonical numeric type model, including integer and floating-point types, instead of redefining numeric semantics around C primitive types. See [`numeric-types.md`](numeric-types.md).

Bitlang-specific ownership, lifetime, property, reference, class/module, and functional lowering rules are defined in [`semantic-lowering.md`](semantic-lowering.md). Remaining non-C decisions are tracked in [`open-decisions.md`](open-decisions.md).

## 1. Translation unit

A Bitlang Low source file is a translation unit containing declarations and definitions in a C-like form.

Statements are terminated by `;` unless the grammar of a compound construct provides its own block termination.

Blocks use braces:

```c
{
    statement;
    statement;
}
```

Bitlang source preprocessing has already completed before this stage. Bitlang preprocessor functions are not part of Bitlang Low semantics.

## 2. Comments

C-style comments are accepted.

```c
// line comment

/* block comment */
```

## 3. Identifiers

Identifiers use the ordinary C-like lexical form: letters, digits, and `_`, with the first character not being a digit.

```c
value
player_hp
_temp2
```

Semantic identifier identity remains case-insensitive as inherited from Bitlang Preprocessed. `_` remains significant.

Therefore capitalization alone must not create a distinct Bitlang Low symbol. Compiler-generated output should use one deterministic canonical spelling, and C backend name mangling must avoid collisions when emitting into C's case-sensitive identifier namespace.

String and character contents remain case-sensitive and are not subject to identifier normalization.

Compiler-generated identifiers are not required to be pleasant for humans to read. They may encode module/class ownership, scope, type, source information, or other data required for deterministic symbol identity.

## 4. Variables

Variable declarations use a C-like declaration shape, but canonical Bitlang types are retained.

```c
Int10x32 value;
Int10x32 count = 0;
Float10x32 ratio = 1.0;
```

Multiple declarations may use the same general C form where doing so does not introduce ambiguity.

```c
Int10x32 a, b, c;
```

Compiler output may choose to emit one declaration per variable for simpler analysis and rewriting.

### 4.1 Canonical numeric types

Numeric type representation is inherited from Bitlang rather than redefined by Bitlang Low.

For numeric types carrying both radix and bit width, the canonical form is:

```text
<TypeName><Radix>x<BitWidth>
```

Examples:

```text
Int10x32
Uint10x32
Int2x32
Int16x64
Float10x32
Float10x64
```

The bit width and radix are explicit type information. Signedness is explicit for integer families. Bitlang Low must not replace this information with target-dependent C types such as plain `int`, `long`, `float`, or `double`.

Source shorthand has already been resolved before compiler-generated Bitlang Low is produced. Therefore source-level shorthand such as `int` is represented by its canonical Bitlang type, normally `Int10x32`, before this stage.

Different radix types are distinct. No C integer promotions, usual arithmetic conversions, or ordinary implicit numeric conversions are inherited. Widening, narrowing, signedness changes, radix changes, and integer/floating-point conversions must be explicit.

Bitlang's distinction between cast and pulse remains valid in Bitlang Low. The Bitlang Low representation preserves the already-resolved conversion operation rather than asking a C backend to infer one.

Numeric overflow behavior is inherited from Bitlang and is an error by default unless an explicit operation specifies another policy.

Floating-point types follow the same inheritance rule as integer types. Their canonical Bitlang type and already-defined semantics are preserved through Bitlang Low; `float`, `double`, and related C types are backend representation choices only.

## 5. Assignment

C-style assignment syntax is used.

```c
value = 10;
value += 2;
value -= 2;
value *= 2;
value /= 2;
value %= 2;
```

Compound assignment may be normalized into explicit assignment and operation form.

Compiler-generated Bitlang Low output is permitted to prefer the already-expanded form inherited from Bitlang Preprocessed rather than reintroducing source-level sugar.

## 6. Arithmetic and comparison operators

The following C-style operators are part of the baseline surface where the operand types support them:

```text
+  -  *  /  %
== != < <= > >=
```

Unary `+` and `-` use C-like syntax.

Bitwise and logical operators are also written in the C form:

```text
& | ^ ~
&& || !
<< >>
```

Operand types must already satisfy Bitlang's explicit compatibility rules. C's implicit integer promotions and target-dependent conversion rules do not apply.

Bitlang Low must not depend on C's unspecified operand evaluation order. When side effects make order observable, lowering must sequence them explicitly using statements and compiler-generated temporaries before C emission. See [`semantic-lowering.md`](semantic-lowering.md).

Shift operators act on the value's fixed semantic bit width rather than on a C carrier width.

For a value of semantic width `W`, the shift count must satisfy `0 <= count < W`. A statically provable invalid count is a compile error; a dynamically determined count must be checked before the operation and becomes a runtime error when invalid.

`value << count` shifts the fixed-width bit representation left, inserts zero bits on the low side, and discards bits shifted beyond the high end.

`value >> count` shifts right. Unsigned values use a logical right shift with zero fill. Signed values use a deterministic arithmetic right shift that preserves the sign by filling from the sign side.

Bits discarded by either shift do not themselves produce numeric overflow. Shift is a bit-representation operation, not shorthand for checked multiplication or division by a power of two. Arithmetic that requires numeric overflow checking must use the corresponding arithmetic operation.

The result retains the original semantic numeric type, including its bit width, radix, and signedness. No C integer promotion is introduced.

A C backend must reproduce these semantics explicitly and must not rely on C undefined behavior or implementation-defined signed-right-shift behavior.

## 7. Increment and decrement

C-style increment and decrement syntax may be accepted by a Bitlang Low parser where defined for the type:

```c
i++;
++i;
i--;
--i;
```

However, Bitlang source and canonical preprocessing do not rely on increment/decrement value-expression semantics. Compiler-generated Bitlang Low output may normalize mutation into ordinary explicit arithmetic assignment and need not emit these forms.

## 8. Functions

Functions use a C-like declaration and definition form while retaining canonical Bitlang types.

```c
Int10x32 add(Int10x32 a, Int10x32 b) {
    return a + b;
}
```

A function without a return value uses `void`.

```c
void reset(void) {
}
```

Function prototypes use the same form:

```c
Int10x32 add(Int10x32 a, Int10x32 b);
```

High-level methods are lowered into ordinary functions with explicit receiver/object representation where needed.

```c
void Player_damage(struct Player* self, Int10x32 amount);
```

Captured functions/closures must use an explicit environment representation before backend emission. See [`semantic-lowering.md`](semantic-lowering.md).

## 9. Return

C-style return syntax is used.

```c
return value;
```

or:

```c
return;
```

for a `void` function.

## 10. Conditional branching

C-style `if`, `else if`, and `else` syntax is used.

```c
if (value > 0) {
    positive();
} else if (value < 0) {
    negative();
} else {
    zero();
}
```

## 11. switch

C-style `switch`, `case`, `default`, and `break` syntax is used where the controlling type is valid.

```c
switch (value) {
case 0:
    zero();
    break;
case 1:
    one();
    break;
default:
    other();
    break;
}
```

C-compatible fallthrough behavior may be reused unless a later Bitlang-specific restriction requires explicit fallthrough annotation or diagnostics.

## 12. Loops

### while

```c
while (condition) {
    work();
}
```

### do-while

```c
do {
    work();
} while (condition);
```

### for

```c
for (Int10x32 i = 0; i < count; i++) {
    work(i);
}
```

The compiler may normalize loop forms internally when performing optimization.

## 13. break and continue

C-style loop control is used.

```c
break;
continue;
```

## 14. struct

C-style structures are retained as a first-class low-level facility.

```c
struct Player {
    Int10x32 hp;
    Int10x32 mp;
};
```

Member access uses the C form:

```c
player.hp
player_ptr->hp
```

Bitlang high-level classes may be lowered into one or more `struct` definitions plus ordinary functions.

Nested struct values do not need to be flattened field-by-field. Struct preservation is an acceptable default lowering.

### 14.1 Ordinary struct layout

Ordinary Bitlang Low structs preserve field declaration order.

Their physical layout is target/backend dependent rather than globally Bitlang-fixed. Layout is computed after each field has been assigned its selected backend storage representation.

For the C backend, ordinary structs follow the selected target C ABI/layout rules for field alignment, inter-field padding, tail padding, and overall struct alignment.

This means the same ordinary Bitlang Low struct is allowed to have different physical sizes, offsets, or alignment on different targets.

The backend must not reorder fields.

### 14.2 Arbitrary-bit-width fields

A field's semantic bit width does not by itself determine how many physical bits it occupies inside an ordinary struct.

For example, if `Int10x24` is represented by a 32-bit carrier on the selected backend, an ordinary struct field of that type participates in layout using that carrier's storage size and alignment rather than being packed into exactly 24 physical bits.

Exact bit packing is a separate explicit-layout concern and is not implied by using an arbitrary-bit-width numeric type.

### 14.3 Padding

Padding inserted by the selected target layout is not part of the semantic value of a struct.

Padding bytes/bits have no stable Bitlang value and must not be used to define ordinary struct equality, hashing, serialization, or other semantic behavior.

Operations that require stable byte-for-byte layout must use an explicit ABI/exact-layout contract rather than relying on incidental ordinary-struct padding.

### 14.4 sizeof, alignof, and field offsets

`sizeof(struct T)`, struct alignment, and field offsets report the selected target/backend physical layout.

They are therefore target-dependent for ordinary structs.

Code that requires these values to remain identical across targets must use an explicit fixed-layout facility.

### 14.5 Explicit alignment requests

A C-compatible explicit alignment request may be used when it does not conflict with Bitlang safety rules.

The selected backend must either satisfy the requested alignment or reject the program for that target.

Reducing alignment below the natural safe alignment, packed storage, exact byte offsets, or exact bit offsets are not ordinary struct behavior and belong to the explicit-layout facility.

### 14.6 ABI and exact-layout boundary

Ordinary struct layout is suitable for internal target-native data and for C interoperability only when the selected C ABI is intentionally the contract.

Protocols, persistent binary formats, memory-mapped hardware, cross-target stable layouts, or other cases requiring exact physical representation must use the separate explicit layout / bit-field facility.

Full field flattening is an optimization only when all struct layout, address, aliasing, `sizeof`, `alignof`, field-offset, and ABI observability is proven irrelevant.

## 15. enum

C-style enumeration syntax is part of the baseline surface.

```c
enum State {
    STATE_IDLE,
    STATE_RUNNING,
    STATE_STOPPED
};
```

The deterministic underlying Bitlang numeric representation remains to be defined separately when enum range, storage, or ABI is observable.

## 16. Arrays

C-like fixed-size array syntax is used.

```c
Int10x32 values[16];
```

Indexing uses square brackets:

```c
values[index]
```

Multi-dimensional C-like syntax may be represented directly:

```c
Int10x32 matrix[4][4];
```

Array bounds are part of Bitlang semantics and out-of-range access is always an error.

If an out-of-range index can be proven statically, compilation must fail.

If the index is only known at runtime, lowering must preserve a bounds check before the access. A failed runtime bounds check enters Bitlang's runtime error path; the exact common runtime error/trap mechanism is specified separately.

The C backend must not emit unchecked indexing for an access whose validity has not already been proven.

## 17. Pointers, references, Address, and unsafe operations

C-like pointer-shaped low-level representation may be used where appropriate.

Bitlang `Ptr<T>` and `Ref<T>` remain semantically distinct until their guarantees have been discharged by lowering.

### 17.1 Ref<T>

`Ref<T>` is the safe reference form.

It is non-null, cannot participate in pointer arithmetic, cannot be rebound after binding, and must not outlive its referent.

A C backend may eventually represent `Ref<T>` with pointer-shaped storage, but it must not reintroduce an operation that violates the already-validated reference guarantees.

### 17.2 Ptr<T>

`Ptr<T>` is the low-level raw-pointer form. It may be null and may be copied or reassigned according to the declaration's ordinary Bitlang properties.

The following are ordinary pointer operations and do not require an unsafe operation merely because the value is a `Ptr<T>`:

- pointer creation from a valid object/address-producing operation,
- copy and reassignment,
- null comparison,
- equality and inequality comparison.

Dereference is an ordinary operation only when the compiler can prove the pointer refers to a live object of a compatible type with sufficient alignment for the access.

If those conditions cannot be proven, dereference requires an explicit unsafe operation.

A pointer value that is statically known to be null, dangling, one-past, misaligned for the requested access, or otherwise invalid for dereference is rejected as an ordinary dereference.

### 17.3 Pointer arithmetic

Ordinary pointer arithmetic is permitted only when the compiler can prove that the result remains within the same allocated object/array, or is the one-past position associated with that same region.

A one-past pointer may participate in allowed pointer calculations/comparisons but is not valid for dereference.

Pointer arithmetic that cannot satisfy or prove the ordinary same-region rule requires an explicit unsafe operation.

Relational pointer ordering is ordinary only when the compared pointers are proven to belong to the same allocation/array region. Ordering unrelated pointers directly is not an ordinary `Ptr<T>` operation.

Code that intentionally wants to compare raw numeric addresses must first convert the pointer explicitly to `Address`.

### 17.4 Address

`Address` is the dedicated low-level type for an address value.

Pointer/integer conversion is not implicit and an ordinary Bitlang numeric integer is not automatically interchangeable with `Ptr<T>`.

A pointer may be explicitly converted to `Address` when raw-address inspection/manipulation is required.

An `Address` may be explicitly converted to `Ptr<T>`, but the resulting pointer receives no compiler safety guarantee merely from that conversion. In particular, liveness, compatible object type, alignment, range, and provenance may be unknown.

Dereferencing such a pointer therefore requires either later proof that re-establishes the ordinary safety conditions or an explicit unsafe operation.

Address width and backend representation follow the selected target. A C backend may use `uintptr_t`, an equivalent implementation-defined carrier, or another representation that preserves the target address value.

### 17.5 Provenance and safety metadata

Detailed provenance is compiler analysis metadata rather than a source-visible semantic type hierarchy.

The compiler may track allocation identity, lifetime, range, alignment, and related pointer facts to prove that an operation is ordinary-safe.

Operations that destroy or obscure those facts, such as converting through raw `Address`, may cause the resulting pointer to lose ordinary safety proof without changing its `Ptr<T>` type.

### 17.6 Explicit unsafe pointer operations

Unsafe pointer operations are opt-in exceptions to the ordinary proof requirements.

They are intended for low-level tasks such as operating-system interfaces, memory-mapped hardware, foreign ABIs, allocators, custom memory managers, and similar code where the compiler cannot prove the required pointer facts.

Unsafe may permit operations such as:

- dereference without compiler proof of liveness/range/alignment/provenance,
- pointer arithmetic whose same-allocation bounds cannot be proven,
- explicit raw-address reconstruction,
- explicit unaligned load/store operations where supported by the target.

Unsafe does not silently weaken `Ref<T>`; code must use the appropriate raw-pointer/address operation instead.

The exact source spelling of unsafe operations is specified separately. The semantic requirement is that the unsafe intent is explicit and cannot arise accidentally from ordinary C-like syntax.

An unsafe operation is allowed to rely on target/backend-specific low-level behavior only where that operation's contract explicitly permits it. Ordinary Bitlang Low operations remain governed by the defined-behavior and default-error rules.

The exact textual spelling used for `Ptr<T>` versus `Ref<T>` inside canonical Bitlang Low output remains a separate syntax decision.

## 18. Function pointers

Function pointer representation may use C-compatible syntax where required for direct C translation.

```c
Int10x32 (*operation)(Int10x32, Int10x32);
```

Captured closures require an explicit environment in addition to a callable target and are not represented as a bare C function pointer unless no capture state is required.

## 19. typedef

A C-compatible `typedef` form may be used for low-level aliases.

```c
typedef Uint10x32 CounterType;
```

Aliases must not erase the resolved semantic type information required for validation or backend lowering.

## 20. const

`const` uses C-like placement and syntax where appropriate, but its Bitlang semantics are stronger than C `const`.

```c
const Int10x32 value = 10;
```

Bitlang `Const` means the semantic value remains fixed after initialization. For aggregates or objects, mutation through another alias is not permitted merely because the C backend type system could express such an alias.

C `const` may be emitted as part of backend representation, but the compiler must already have enforced the stronger Bitlang invariant. See [`semantic-lowering.md`](semantic-lowering.md).

## 21. static and storage-related declarations

When `static` represents Bitlang static retention, it means storage/lifetime retention and does not implicitly mean C internal linkage.

```c
static Int10x32 counter;
```

Linkage/export visibility is a separate semantic concern and must be lowered independently. A C backend must not hide a symbol solely because its Bitlang value has static lifetime.

## 22. Explicit casts

C-style explicit cast syntax is used as the baseline representation for a resolved cast operation:

```c
Int10x32 value = (Int10x32)ratio;
```

The legality and failure behavior of a cast follow Bitlang's explicit conversion rules rather than C's permissive conversion model.

No ordinary implicit conversion is performed merely to make an operation type-compatible.

Representation-changing pulse operations remain semantically distinct from casts even if a backend eventually implements both through generated conversion code.

## 23. sizeof / bitsizeof / alignment

Bitlang Low separates semantic bit width from backend storage size.

`sizeof(type-or-value)` reports the physical storage size in bytes for the selected backend representation. It is therefore suitable for C-compatible layout, allocation, pointer stepping, ABI work, and other operations that depend on actual storage.

`bitsizeof(type-or-value)` reports the semantic Bitlang bit width when the type has a defined semantic bit width.

For example, a semantic `Int10x24` may use a 32-bit C carrier:

```text
bitsizeof(Int10x24) == 24
sizeof(Int10x24)    == 4   // when the selected backend stores it in a 32-bit carrier
```

The two sizes are intentionally allowed to differ.

The programmer or generated Bitlang Low code may use whichever measurement is appropriate to the operation. Backend lowering must not substitute one for the other.

Alignment is a storage/layout property rather than a semantic numeric-width property. Any `alignof`-equivalent operation therefore reports the alignment of the selected backend representation.

For types that do not define a meaningful semantic bit width, `bitsizeof` is invalid unless that type's own Bitlang specification defines what semantic bit size means.

## 24. C-compatible lowering principle

A valid Bitlang Low construct should, where possible, translate to straightforward C without reconstructing lost high-level semantics.

For example, a high-level class method:

```text
player.damage(amount)
```

may be represented in Bitlang Low approximately as:

```c
struct Player {
    Int10x32 hp;
};

void Player_damage(struct Player* self, Int10x32 amount) {
    self->hp -= amount;
}
```

The C backend then maps canonical Bitlang types and validated properties to C representations that preserve their semantics.

A defined Bitlang operation must not be translated into C in a way whose correctness depends on C undefined behavior. When direct C is insufficient, the backend must use checks, helpers, carrier representations, explicit control flow, or another defined lowering strategy.

The exact compiler-generated identifier names are an implementation concern and need not be optimized for human readability.

## 25. Property and state lowering

Bitlang Preprocessed contains the final resolved semantic property state. Bitlang Low does not need to preserve every property name textually after its constraints have been validated and lowered.

Important inherited rules include:

- read, write, and reassignment capability remain independent during validation;
- ownership, borrow state, lifetime, copy/move capability, move state, release policy/capability/state, initialization, nullability, optionality, and const state must not be guessed from C defaults;
- `Auto_release` ownership must become explicit release behavior on applicable exit paths unless ownership was transferred or release is otherwise no longer required;
- `Moved`, `Released`, and `Uninitialized` states must not survive as ordinary valid reads/releases in emitted code;
- absence and null are separate, so `Optional nullable T` must preserve distinct absent, present-null, and present-non-null states where reachable;
- analysis-only metadata may be removed once the required low-level behavior and safety constraints have been established.

Detailed rules are in [`semantic-lowering.md`](semantic-lowering.md) and [`borrow-state-lowering.md`](borrow-state-lowering.md).

### 25.1 Destruction safety gate

Before release/finalization properties are consumed or erased during lowering, the compiler must verify that every statically provable destruction path is valid.

A provably invalid release/finalization is a compile error even when it was generated by a language converter, the Bitlang preprocessor, automatic cleanup insertion, or compiler lowering itself.

The compiler must reject double destruction, invalid non-owner release, release through an unreleasable path, destruction that leaves a live dependent borrow/reference dangling, and destruction that contradicts resolved lifetime/finalization constraints.

## 26. High-level construct elimination

Bitlang Low is procedural and low-level. The C backend must not be responsible for reconstructing high-level language meaning.

Before backend emission:

- class methods become ordinary functions with explicit receiver representation;
- dynamic dispatch becomes explicit call targets/tables where required;
- closures become an explicit callable target plus explicit environment representation;
- pattern matching becomes ordinary control flow;
- sum/variant-like values receive an explicit low-level representation;
- unresolved source-level generic/currying/preprocessor syntax does not remain.

Module/class ownership needed for symbol identity is encoded into generated names or explicit symbol metadata.

## 27. Runtime failure model

Runtime-detectable violations of ordinary Bitlang Low operations use a common **trap** failure model by default.

Examples include:

- numeric overflow where the operation uses the default checked-overflow policy,
- failed checked conversions when used in trapping form,
- runtime array-bounds violations,
- invalid dynamic shift counts,
- other runtime semantic guards inserted by lowering.

If the violation can be proven statically, compilation fails instead; a trap is not emitted for a program that is already known to be invalid.

A trap terminates normal program execution immediately. It is not ordinary error propagation and does not silently produce a fallback value.

Operations for which the program intentionally wants to handle failure must use an explicit checked/non-trapping operation. Such an operation returns an explicit success/failure result that can be lowered into an ordinary tagged result value, status plus out-value, or an equivalent backend representation.

The trapping and checked forms are semantically distinct. The backend must not silently convert an ordinary trapping operation into error-return control flow, or a checked operation into a trap.

The exact textual spelling of each checked operation may be defined with that operation family; the common rule is that recoverable failure must be explicit in Bitlang Low rather than changing the default operation semantics.

A C backend may implement the trap through a runtime helper, target trap instruction/intrinsic, or equivalent immediate-failure mechanism, provided the observable Bitlang behavior is preserved.

## 28. C undefined-behavior policy

Bitlang Low does not inherit C undefined behavior as ordinary language behavior.

If an operation would require the generated C program to enter undefined behavior, that operation is an error by default.

This includes cases that can be proven statically and cases that only become invalid for particular runtime values. Static violations are compile errors. Dynamic violations must be guarded by generated checks when the operation is otherwise permitted to execute.

An exception may exist only when the behavior is intentionally useful for low-level programming and Bitlang Low explicitly defines an unsafe or backend-specific operation for it. Such an exception must be opt-in and documented; accidental reliance on C undefined behavior is never valid lowering.

Therefore, the C backend may not use undefined behavior as an optimization assumption for a Bitlang operation whose semantics require a defined result or defined error.

This policy does not automatically adopt C implementation-defined behavior either. Where implementation-defined C behavior is observable and Bitlang has not explicitly adopted it, Bitlang Low must either define its own behavior, lower through a deterministic helper/representation, or reject the operation.

## 29. Remaining Bitlang-specific decisions

The remaining decisions that cannot simply inherit C behavior are tracked in [`open-decisions.md`](open-decisions.md). The major unresolved areas are:

- deterministic enum underlying representation,
- external C/native ABI and symbol contract,
- Bitlang string/character low-level representation,
- static/module initialization and destruction order,
- concurrency/atomic memory model,
- exact canonical Ptr/Ref textual representation,
- exact-layout / bit-field facility for protocols, hardware, and ABI-specific layouts.

Until one of these areas is explicitly defined, similarity to C syntax does not imply that C implementation-defined behavior becomes Bitlang semantics. C undefined behavior is already rejected by the general policy above unless an explicit Bitlang unsafe/backend-specific exception is defined.
