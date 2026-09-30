# Bitlang Low Language Specification

Status: Draft

Bitlang Low is a C-like low-level programming language positioned between `Bitlang Explicit` and backend representations such as C in the normal generated pipeline. It may also be written and compiled directly by humans.

The baseline rule is simple: when ordinary C syntax and semantics can be reused without conflicting with Bitlang Low requirements, they are reused directly.

This document records the C-compatible surface and the Bitlang-specific rules that must survive lowering.

A foundational exception is the numeric type system: Bitlang Low inherits Bitlang's canonical numeric type model, including integer and floating-point types, instead of redefining numeric semantics around C primitive types. See [`numeric-types.md`](numeric-types.md).

Bitlang-specific ownership, lifetime, property, reference, class/module, and functional lowering rules are defined in [`semantic-lowering.md`](semantic-lowering.md). Target-dependent physical representation is defined by [`target-model.md`](target-model.md). Remaining non-C decisions are tracked in [`open-decisions.md`](open-decisions.md).

## Bitlang Low is a programming language, not an IR-only format

Bitlang Low is a first-class programming language.

Its use as a compiler-generated lowering target does not make it an implementation-private IR, serialized AST, or compiler-only artifact.

A conforming implementation must be able to accept valid human-authored Bitlang Low source and process it according to the Bitlang Low specification.

Human-authored and compiler-generated Bitlang Low share the same language semantics. The compiler must not rely on hidden assumptions that only its own generator could satisfy unless those assumptions are made explicit as part of the Bitlang Low language contract.

Direct Bitlang Low source enters at the Bitlang Low validation/backend boundary:

```text
human-authored Bitlang Low
    -> parse + semantic validation
    -> C / assembly / another backend
```

It does not need to be reverse-engineered into Bitlang source or Bitlang Explicit first.

Bitlang Low may remain intentionally lower-level, stricter, and less convenient than ordinary Bitlang. Human writability means the language is completely specified and valid to author directly; it does not require compiler-generated output to prioritize readability.

This preserves the Bitlang family design goal that programmers can choose a writing style and abstraction level rather than being forced to use only the highest-level frontend.

## Canonical audit boundary for the C path

When Bitlang is compiled through the C backend, Bitlang Low is the final canonical Bitlang representation before backend-language translation.

This makes Bitlang Low an explicit audit/debug boundary, not merely a transport format.

A programmer should be able to inspect generated Bitlang Low to determine, before C emission:

- which concrete types and storage representations Bitlang selected;
- which calls and control-flow paths remain;
- where explicit cleanup/release occurs;
- which runtime checks remain;
- which declarations and standard-library support survived reachability analysis;
- which low-level operations the backend is required to preserve.

Generated Low may be verbose and compiler-oriented, but it must not depend on hidden meaning that exists only inside compiler memory.

For an assembly-oriented pipeline, a lower assembly representation may follow Bitlang Low. That does not remove Bitlang Low's role as the principal human-readable low-level audit representation; assembly readability is not required to match source-language readability.

## Backend complexity is intentional

Bitlang Low intentionally keeps its language surface and semantic model comparatively small and explicit.

This does **not** imply that a Bitlang Low compiler, C translator, or assembly translator should be a simple textual converter.

The backend is responsible for preserving Bitlang Low's already-defined behavior while mapping it onto a less safe or more target-specific representation.

Depending on the target, this may require substantial compiler work, including:

- preserving explicit lifetime and release behavior;
- preventing reintroduction of dangling access or invalid destruction;
- preserving deterministic evaluation order;
- inserting required runtime checks and trap paths;
- representing arbitrary semantic bit widths;
- implementing checked overflow and exact shift behavior;
- lowering `Ref<T>` without losing its validated guarantees;
- choosing target-specific storage and ABI layouts;
- avoiding C undefined or unintended implementation-defined behavior;
- materializing helper routines when a target instruction or C primitive cannot directly express the required semantics;
- register allocation, calling convention handling, stack/layout decisions, and instruction selection for assembly output.

Therefore the design intentionally allows:

```text
simple Bitlang Low language
    -> sophisticated translator/compiler
    -> simple, efficient, defined backend program
```

Complexity should be concentrated in compile-time tooling rather than pushed into runtime semantics.

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

Compiler-generated identifiers may encode module/class ownership, scope, type, source information, or other data required for deterministic symbol identity. They are not required to be aesthetically pleasant, but generated Bitlang Low must remain inspectable enough that a programmer can trace declarations, control flow, storage, and backend-relevant behavior.

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

### 4.2 Bool

`Bool` is a distinct non-numeric type with semantic values `false` and `true`.

The C backend may use `_Bool` / `bool` when the selected target/dialect provides a compatible representation. Otherwise it must use another representation that preserves exactly the two Low boolean values.

The Bitlang VM Backend may lower `Bool` to a simple integer/register carrier such as 0/1, but that carrier does not make `Bool` implicitly compatible with Low integer types.

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

Comparison operators produce `Bool`.

Bitwise operators are written in the C form where the operand type supports them. Logical operators `&&`, `||`, and `!` operate on `Bool` only.

Bitlang Low does not inherit C scalar truthiness: integers, pointers, enums, addresses, strings, and other values must not be used directly as boolean conditions without an explicit comparison/conversion.

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

Bitlang Low preserves the Bitlang distinction between **checked** and **discard** bit shifts. Canonical generated Low uses explicit intrinsics when that distinction matters:

```text
bit_shift_left_checked
bit_shift_left_discard
bit_shift_right_zero_checked
bit_shift_right_zero_discard
bit_shift_right_sign_checked
bit_shift_right_sign_discard
```

The C-like operators are human-facing shorthand for the discard forms:

```text
value << count
value >> count
```

`<<` zero-fills from the low side and explicitly permits bits leaving the high side to be discarded.

For `>>`, unsigned operands use zero-fill and signed operands use deterministic sign-fill; low bits leaving the semantic width are explicitly discarded.

A checked shift must not be silently rewritten to `<<` or `>>`. If its canonical Bitlang loss condition is violated, a statically known violation is a compile error and a runtime-dependent violation uses the ordinary trap path.

Shift is a bit-representation operation, not shorthand for checked multiplication or division by a power of two. Radix/digit shifting remains a separate numeric operation.

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

The condition expression must have type `Bool`. No C-style implicit conversion from a scalar value to boolean is performed.

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

The condition must have type `Bool`.

```c
while (condition) {
    work();
}
```

### do-while

The condition must have type `Bool`.

```c
do {
    work();
} while (condition);
```

### for

When present, the loop condition must have type `Bool`.

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

Bitlang Low enums are nominal integer-backed types with a deterministic canonical underlying Bitlang integer type.

Human-authored Low may use C-like syntax and may omit the underlying type:

```c
enum State {
    STATE_IDLE,
    STATE_RUNNING,
    STATE_STOPPED
};
```

When omitted, the Low default underlying type is:

```text
Int10x32
```

Canonical compiler-generated Low and canonical formatter output must make the underlying type explicit. The canonical spelling follows the fixed-underlying-type form:

```c
enum State : Int10x32 {
    STATE_IDLE = 0,
    STATE_RUNNING = 1,
    STATE_STOPPED = 2
};
```

Enumerator values use C-like constant rules: the first omitted value is zero and each later omitted value is the previous value plus one. Explicit duplicate numeric values are permitted.

Every enumerator value must be representable by the declared underlying Bitlang integer type. Failure to fit is a compile error; values are never silently widened or wrapped.

The enum remains a distinct semantic type. Bitlang Low does not inherit C integer promotions or implicit enum-to-integer conversion. Conversion to/from the underlying integer type must be explicit.

The enum's storage size, alignment, and physical representation are those of the selected backend representation of its explicit underlying type.

For the C backend, a fixed-underlying-type C enum may be emitted when the selected C dialect/ABI preserves the exact contract. Otherwise the backend may lower the enum to a typedef/constant representation or another defined C representation while preserving Low type checking before emission.

The Bitlang VM Backend lowers enum values directly through the explicit underlying integer representation and does not need a separate VM enum primitive.

## 16. Arrays

Bitlang Low has both fixed-length and runtime-length contiguous arrays.

### 16.1 Fixed-length arrays

C-like fixed-size syntax is used:

```c
Int10x32 values[16];
Int10x32 matrix[4][4];
```

A fixed length is part of the physical storage contract and may use inline storage.

### 16.2 Runtime-length arrays

The canonical runtime-length type is:

```text
Array<T>
```

It represents a contiguous region with an explicit element count.

Its semantic low-level descriptor contains:

```text
data   : Ptr<T>
length : Size
```

The descriptor does not imply ownership, allocation, capacity, or resize behavior. Those concerns are represented separately by Bitlang properties or library/container abstractions.

`array_length(value)` returns `Size`.

An explicit storage-view operation may expose the backing `Ptr<T>`/reference view according to the validated ownership and borrowing rules.

C array-to-pointer decay is not part of Low semantics. Conversion from fixed array to runtime-length view or pointer is explicit.

The C backend normally lowers a runtime-length array to an equivalent pointer-plus-`Size` representation. It must not lower it to a bare pointer or C VLA if doing so loses observable length/ownership semantics.

The Bitlang VM Backend lowers the descriptor using guest pointer representation plus the VM target's `Size` representation.

### 16.3 Indexing and bounds

Indexing uses square brackets:

```c
values[index]
```

Array bounds are part of Bitlang semantics and out-of-range access is always an error.

For a runtime-length array, validity requires `0 <= index < length`. The index is explicitly converted/validated against the target `Size` domain when needed; C implicit integer conversions are not used.

If an out-of-range index can be proven statically, compilation must fail.

If the index is only known at runtime, lowering must preserve a bounds check before the access. A failed runtime bounds check enters Bitlang's runtime trap path.

The C backend must not emit unchecked indexing for an access whose validity has not already been proven.

## 16A. Strings and characters

Bitlang Low preserves Bitlang's canonical string semantics.

A `Str` value is a sequence of Unicode scalar values materialized as valid UTF-8.

The minimum ordinary runtime descriptor is semantically equivalent to:

```text
data        : Ptr<Uint2x8>
byte_length : Size
```

The descriptor does not include an implicit trailing NUL byte.

U+0000 may occur inside the data and is an ordinary character.

Character count is the number of decoded Unicode scalar values. A backend may cache that count when useful, but cached character count is representation metadata rather than an additional semantic component of string identity.

Constraints are interpreted as:

```text
Strxxx<N> -> at most N Unicode scalar values
Strxx<N>  -> at most N UTF-8 bytes
Strx<N>   -> encoded byte_length * 8 <= N
```

A value violating an applicable bound is invalid. Statically provable violations are compile errors; runtime construction/conversion operations must check bounds before producing the constrained string.

`Char` / `Strxxx1` contains exactly one Unicode scalar value and may therefore occupy one to four UTF-8 bytes.

Bitlang Low performs no automatic Unicode normalization.

### C backend

Ordinary Low strings are not C NUL-terminated strings.

The C backend normally uses an explicit pointer-plus-`Size` byte-length representation and must not replace it with bare `char*` semantics.

Conversion to a C NUL-terminated string is an explicit interoperability operation. It must allocate/provide terminator storage and explicitly handle embedded U+0000 according to the target API contract.

### Bitlang VM backend

The VM representation uses guest UTF-8 bytes plus guest `Size` byte length. Host Go string layout and host string internals are not guest semantics.

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

Unsafe pointer operations use an explicit `unsafe` block:

```c
unsafe {
    value = *raw_ptr;
    raw_ptr = raw_ptr + offset;
}
```

Only operations whose Low specification explicitly permits unsafe relaxation gain that relaxation inside the block.

In particular, `unsafe` does **not** disable:

- ordinary type compatibility;
- ownership/move state;
- release/finalization rules;
- const/access restrictions;
- explicit cast requirements;
- array bounds semantics for ordinary array operations;
- runtime trap semantics unrelated to the permitted raw-pointer operation.

The initial unsafe permissions are limited to the raw-pointer/address operations defined in this section, including unproved dereference/arithmetic and explicit unaligned load/store intrinsics.

Nested `unsafe` blocks have no additional effect.

Compiler-generated canonical Low must retain an explicit `unsafe` block or equivalent explicitly marked unsafe operation node; a backend may not infer unsafe intent from the fact that C would accept the emitted operation.

An unsafe operation is allowed to rely on target/backend-specific low-level behavior only where that operation's contract explicitly permits it. Ordinary Bitlang Low operations remain governed by the defined-behavior and default-error rules.

Canonical Bitlang Low retains `Ptr<T>` and `Ref<T>` as distinct type spellings.

For C familiarity, a human-authored Low parser may accept C-style `T*` as surface shorthand for `Ptr<T>`. It is never shorthand for `Ref<T>`.

Canonical formatter/compiler-generated Low normalizes pointer types back to `Ptr<T>` and preserves `Ref<T>` explicitly so the audit boundary does not hide reference safety semantics.

A C backend may later lower both to pointer-shaped storage where valid, but only after the already-defined `Ref<T>` guarantees have been validated.

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

`sizeof(type-or-value)` reports the physical storage size in Bitlang bytes for the selected backend representation and has type `Size`. It is therefore suitable for C-compatible layout, allocation, ABI work, and other operations that depend on actual storage.

`bitsizeof(type-or-value)` reports the semantic Bitlang bit width when the type has a defined semantic bit width.

For example, a semantic `Int10x24` may use a 32-bit C carrier:

```text
bitsizeof(Int10x24) == 24
sizeof(Int10x24)    == 4   // when the selected backend stores it in a 32-bit carrier
```

The two sizes are intentionally allowed to differ.

The programmer or generated Bitlang Low code may use whichever measurement is appropriate to the operation. Backend lowering must not substitute one for the other.

Alignment is a storage/layout property rather than a semantic numeric-width property. Any `alignof`-equivalent operation therefore reports the alignment of the selected backend representation and has type `Size`. Field-offset queries likewise return `Size`.

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

## 29. Remaining specification boundary

There are currently no unresolved **Bitlang Low-owned** semantic items tracked in `open-decisions.md`.

Two important upstream Bitlang semantics remain blockers for complete cross-backend conformance:

- the complete `Float<Radix>x<BitWidth>` representation/rounding/exceptional-value contract;
- a future shared-memory concurrency/atomics model.

Until those upstream semantics are defined, Low and its backends must not invent C- or Go-specific behavior for them.

The living cross-stage status is tracked in [`coverage-audit.md`](coverage-audit.md).
