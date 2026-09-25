# Bitlang compiled Semantic Lowering

Status: Draft / inherited semantic contract

This document defines Bitlang-specific semantics that must be preserved while lowering `Bitlang preprocessed` into `Bitlang compiled`.

C-compatible syntax and ordinary C-compatible constructs are not repeated here. The purpose of this document is to define the places where Bitlang semantics must not be lost merely because Bitlang compiled is C-like.

## 1. Stage contract

Bitlang compiled consumes an already-normalized Bitlang Preprocessed program.

The compiled stage must not reconstruct source shorthand, infer omitted semantic properties, or re-run source-level preprocessing rules.

If required semantic information is missing or contradictory at the Preprocessed boundary, compilation must diagnose the input rather than silently selecting a C-like default.

Bitlang compiler-generated compiled output may consume analysis-only properties once their constraints have been proven and represented by lower-level structure. Preservation of behavior is required; preservation of every original property spelling is not.

## 2. Identifiers

Identifier identity remains case-insensitive as inherited from Bitlang Preprocessed. String and character contents remain case-sensitive.

`_` is a significant identifier character.

Compiler-generated compiled output should use one deterministic canonical spelling for each semantic symbol. A C backend, whose identifiers are case-sensitive, must emit collision-free symbols and must not create distinct backend symbols solely from capitalization differences that are semantically identical in Bitlang.

Module, class, and other ownership hierarchy may be encoded into generated symbol names when the corresponding high-level container no longer exists in compiled form.

## 3. Explicit evaluation order

Bitlang compiled must not depend on C's unspecified or implementation-dependent operand evaluation order.

When evaluation order could affect observable behavior, lowering must make the order explicit through statements and compiler-generated temporaries before C emission.

For example, multiple side-effecting operations must not be left inside one expression if correctness would depend on which operand C evaluates first.

After normalization, a remaining expression may be emitted as an ordinary C expression only when its observable result is independent of C's permitted operand-evaluation ordering.

## 4. Semantic-property lowering

Bitlang Preprocessed exposes semantic properties explicitly. Bitlang compiled is allowed to consume those properties during validation and lowering.

A property may disappear from textual compiled output only after its required behavior has been represented by one or more of:

- explicit low-level operations,
- explicit control flow,
- explicit storage or lifetime decisions,
- explicit generated checks,
- symbol/linkage information,
- compiler-known restrictions that have already been proven,
- backend metadata whose semantics are part of the lowering contract.

Properties must never disappear merely because C lacks an equivalent qualifier.

## 5. Read, write, and reassignment capability

`Readable / Unreadable`, `Writeable / Unwriteable`, and `Reassignable / Unreassignable` remain independent semantic restrictions during validation.

The compiler must reject an operation that violates the resolved capability state before emitting ordinary low-level operations.

After all relevant accesses have been validated, capability metadata that has no remaining runtime effect may be omitted from generated compiled text.

`Unreassignable` does not imply that reachable object state is immutable. `Unwriteable` does not by itself imply that the binding cannot be replaced. Lowering must preserve this distinction.

## 6. Const

Bitlang `Const` is stronger than C `const`.

A `Const` value is semantically fixed after initialization. For an aggregate or object, state that forms part of that value must not be mutated through another alias merely because C's type system would permit such an alias.

A C backend may use `const` where useful, but C `const` alone is not sufficient to implement Bitlang `Const`. The compiler must already have rejected mutation paths that would violate the stronger Bitlang rule.

## 7. Ownership and release

`Owned / Borrowed`, release policy, release capability, and release state are distinct input properties.

An owned resource with `Auto_release` must be lowered so that every applicable normal exit path performs the required release unless analysis proves that ownership was transferred, the resource was already validly released, or the release itself is removable as an optimization.

`Manual_release` must not cause the compiler to invent an automatic release merely to imitate C scope behavior.

A `Borrowed` path must not independently destroy the owned resource unless an explicit, valid ownership transformation has already occurred.

`Released` resources must not be emitted as ordinary live-resource accesses. Double release and use-after-release that are statically provable are compilation errors.

The variable slot or handle may remain in compiled form after the underlying resource has been released when doing so is required for control flow or diagnostics.

## 8. Destruction and finalization safety

The compiler must treat destruction safety as a mandatory validation gate.

The origin of an operation is irrelevant: handwritten input, language-adapter output, preprocessor-generated cleanup, and compiler-generated cleanup are checked under the same rules.

Compilation must fail when the compiler can prove that a release, finalizer/destructor execution, or owner teardown would violate resolved Bitlang semantics. This includes, at minimum:

- double release or double finalization of the same live resource;
- release through an `Unreleasable` or non-owning path without a valid ownership transfer;
- destruction that invalidates a still-live borrow/reference;
- destruction inconsistent with the resolved lifetime or finalization trigger;
- generated control flow containing multiple executable cleanup paths for the same resource.

The compiler must not assume that a transformation is safe merely because it was produced by the Bitlang preprocessor or another trusted tool. Fully explicit Preprocessed input reduces ambiguity; it does not remove the compiler's obligation to reject provably invalid destruction.

Unsafe/raw-pointer facilities do not automatically waive ownership/release/finalization invariants. A separate explicitly specified destruction model would be required to permit behavior that ordinary Bitlang destruction rules reject.

## 9. Move and copy

Copy and move capability are distinct from ownership.

A valid copy produces a separate value according to the type's resolved copy semantics.

A valid move transfers the represented value or resource and invalidates ordinary use of the source according to the resolved move state.

Compiler-generated compiled output must not contain an ordinary read, second move, or release through a source that remains semantically `Moved`.

Move-state metadata may be removed after the compiler has transformed the program into explicit transfers and proven that no invalid source use remains.

## 10. Initialization state

An `Uninitialized` declaration must never be read as a value.

The compiler must lower only valid initialization transitions into ordinary stores or construction operations. If an execution path can read a value before valid initialization, compilation must fail rather than inherit C's indeterminate-value behavior.

After definite initialization has been proven, initialization-state metadata may be discarded.

## 11. Nullability and optionality

Nullability and presence are separate semantics and must not be collapsed into one C pointer convention.

`unnullable` means that `null` is not a valid semantic value for that declaration or access path. A backend must not manufacture a nullable representation and then treat null as valid merely because the corresponding C representation is pointer-shaped.

`nullable` permits the semantic null value when the type otherwise supports it. The physical null representation is a backend concern unless an ABI contract explicitly fixes it.

`Optional` means a value may be absent. `Required` means it must be present.

Absence is not the same as null. In particular, `Optional nullable T` must be capable of distinguishing at least these semantic states when they are reachable:

- absent,
- present with null,
- present with a non-null value.

The backend may choose any representation that preserves those states.

## 12. Lifetime and borrow state

Lifetime properties and borrow state are validation inputs, not necessarily permanent runtime data.

A dependent value or borrow must not outlive the value on which it depends.

Conflicting shared/exclusive borrows, release while an incompatible borrow remains active, and move operations that invalidate a still-used borrow must be rejected.

After these constraints have been established, analysis-only lifetime and borrow metadata may be removed. See [`borrow-state-lowering.md`](borrow-state-lowering.md).

## 13. Ptr and Ref

Bitlang `Ptr<T>` and `Ref<T>` are semantically distinct even if a C backend eventually represents both using pointer-shaped storage.

`Ptr<T>` is the raw low-level pointer form and may be nullable, reassignable, participate in pointer arithmetic, and expose raw-address operations when permitted by the target and the resolved properties.

`Ref<T>` is a safe reference. It must remain non-null, cannot participate in pointer arithmetic, cannot be rebound after binding, and must not outlive its referent.

Lowering may erase the Ptr/Ref distinction only after all `Ref<T>` guarantees have been discharged into validated low-level behavior. A backend must not reintroduce operations that would violate those guarantees.

The final textual spelling used to distinguish Ptr and Ref inside Bitlang compiled remains a separate syntax decision.

## 14. Static retention is not C internal linkage

Bitlang's static-retention semantics and C's file-scope internal-linkage use of `static` are not the same concept.

When a compiled construct represents Bitlang static retention, the backend must preserve the required storage lifetime.

The backend must not infer that the symbol should have C internal linkage solely because the Bitlang value has static retention. Linkage/export visibility is a separate concern and must be lowered separately.

## 15. Class, module, and method lowering

High-level class and module containers do not need to survive as runtime language constructs in Bitlang compiled.

A class may lower into:

- one or more `struct`-like data representations,
- ordinary functions,
- explicit object/self parameters,
- function-pointer tables or other explicit dispatch data when dynamic dispatch is required.

A method call must therefore become an ordinary low-level call whose target and receiver representation are explicit.

Module and class ownership information that is still needed for symbol identity is encoded into generated names or symbol metadata.

Nested `struct` values do not need to be flattened field-by-field merely because a class was lowered. Preserving a struct as a struct is the default acceptable representation. Full field flattening is an optimization and is legal only when layout, aliasing, and observable behavior are preserved.

## 16. Functional constructs

Bitlang compiled is procedural and must not require a backend to reconstruct high-level functional-language semantics.

First-class functions without captures may lower to ordinary function pointers or another equivalent callable representation.

A closure with captured state must lower to an explicit code target plus an explicit environment representation. The environment may be a generated struct or equivalent low-level object.

Calls through closures must make the required environment explicit before backend emission.

Pattern matching and other high-level control constructs must be reduced to ordinary control flow. Sum/variant-like values must have an explicit low-level representation before backend translation.

Unresolved source-level generics, currying syntax, or other high-level functional sugar must not be left for the C backend to interpret.

## 17. No Bitlang preprocessor at the compiled stage

Bitlang preprocessor functions have already executed before Bitlang Preprocessed is produced and are not part of Bitlang compiled runtime or compile-time semantics.

A C backend may generate C preprocessor directives as an implementation technique, but those directives are backend output and are not Bitlang compiled preprocessing semantics.

## 18. C backend safety rule

A behavior that is defined by Bitlang must not be lowered into C in a form whose correctness depends on C undefined behavior.

When ordinary C cannot express a required defined behavior directly, the backend must use an explicit check, helper operation, wider/carrier representation, generated control flow, or another defined mechanism.

If the target cannot implement the required Bitlang semantics, translation must fail explicitly instead of silently weakening the language contract.

Raw-pointer operations whose own Bitlang semantics intentionally allow unsafe behavior remain subject to the separately defined raw-pointer rules; this section does not invent additional guarantees for them.
