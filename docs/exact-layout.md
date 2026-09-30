# Bitlang Low Exact Layout

Status: Draft / deterministic physical-layout contract

Exact layout is a separate facility from ordinary target-native structs.

Ordinary structs follow the selected target ABI. Exact-layout structs define a deterministic physical bit/byte representation that must remain stable across backends that support the declaration.

C bit-fields, compiler packing defaults, and host struct layout are never the semantic source of truth for exact layout.

## 1. Declaration model

An exact-layout struct defines:

- explicit byte order: `little` or `big`;
- explicit bit numbering inside each byte: `lsb0` or `msb0`;
- explicit alignment in Bitlang bytes, default 1;
- fields with exact bit offsets;
- field types whose semantic bit representation/width is fully defined.

Illustrative canonical form:

```text
exact struct Header byte_order(little) bit_order(lsb0) align(1) {
    Uint2x3  flags @bit(0);
    Uint2x5  kind  @bit(3);
    Uint16x16 code @bit(8);
}
```

The exact parser spelling may use the equivalent canonical attribute grammar adopted by the Low frontend; the semantic fields above are mandatory.

## 2. Field width

By default a field occupies exactly `bitsizeof(field_type)` bits.

A field type whose semantic bit representation/width is not defined cannot be placed directly in an exact layout.

Examples:

- fixed-width Int/Uint are allowed;
- fixed arrays are allowed when element exact layout is known;
- nested exact-layout structs are allowed;
- ordinary target-sized `Address`, `Size`, and `Offset` are not cross-target exact types;
- runtime-length `Array<T>` and ordinary `Str` descriptors are not exact-layout fields;
- floating-point fields are allowed only after the corresponding Bitlang float bit representation is fully specified.

Signed integers use their canonical W-bit two's-complement representation.

## 3. No implicit padding

Exact layout inserts **no implicit padding bits or bytes**.

Every physically meaningful bit must be represented by a declared field.

If a protocol/ABI contains reserved or padding bits, the declaration represents them explicitly as fields, for example with a suitably sized `Uint2xN` field.

This makes byte-for-byte materialization deterministic and auditable.

## 4. Overlap

Fields must not overlap.

Overlay/union semantics are not part of the initial exact-layout facility.

A future explicit overlay facility may be added separately; ordinary exact fields never alias the same physical bit range.

## 5. Offset and size

Each field has a non-negative exact bit offset.

A field occupies:

```text
[field_bit_offset, field_bit_offset + bitsizeof(field_type))
```

The layout's semantic bit size is the highest occupied ending bit.

Because exact layout has no implicit padding, any intentional trailing bits must be represented explicitly.

Physical byte size is:

```text
ceil(bitsizeof(layout) / 8)
```

The unused high bits of the final physical byte, when the total bit size is not byte-aligned, are not addressable fields and must be written as zero by deterministic serialization/materialization.

`bitsizeof` returns the exact semantic layout bit count. `sizeof` returns the physical byte count.

## 6. Byte and bit order

`bit_order(lsb0)` means bit offset 0 within each physical byte addresses the least-significant bit of that byte.

`bit_order(msb0)` means bit offset 0 within each physical byte addresses the most-significant bit of that byte.

For a multi-byte numeric field, `byte_order` determines the order of successive 8-bit groups of the field's canonical semantic bit representation in physical memory.

Partial-byte groups for non-multiple-of-8 widths follow the same declared bit-order rule. Backends must use generated masks/shifts/helpers when their native load/store representation does not match directly.

The declaration never inherits host/native endianness implicitly.

## 7. Alignment

Exact layout defaults to alignment 1 Bitlang byte.

An explicit larger alignment may be requested.

Alignment changes where an instance may be placed; it does not insert hidden bits inside the exact layout.

A target that cannot satisfy the requested alignment must reject the declaration for that target rather than silently changing it.

## 8. Loads, stores, and safety

A backend may implement an exact-layout field with:

- native aligned load/store when provably identical;
- byte loads/stores;
- masks/shifts;
- generated helper functions;
- explicit unaligned operations under the normal pointer/unsafe rules.

The field's declared representation remains the semantic truth.

Access to memory-mapped hardware still obeys the explicit unsafe/volatile-hardware-access contract when such a contract is selected. Exact layout by itself does not create concurrency or volatile semantics.

## 9. C backend

The C backend must not rely on ordinary C bit-field ordering to implement exact layout.

It may emit a C packed struct or bit-field representation only when compile-time checks prove that the selected C target produces exactly the required offsets, size, byte order, bit order, and alignment.

Otherwise it emits byte storage plus explicit accessors/masks/shifts.

Static assertions should be emitted where a direct C layout is selected.

## 10. Bitlang VM backend

The VM backend lays out exact objects directly in guest byte-addressed memory according to this specification.

Complex field access is lowered before Assam Core architecture translation into simple loads/stores/masks/shifts or target-independent runtime helpers.

Architecture translators must not rediscover exact-layout semantics.

## 11. Intended uses

Exact layout is appropriate for:

- binary protocols;
- persistent binary formats;
- firmware/hardware register layouts;
- explicit foreign ABI records;
- deterministic on-disk/network records.

Ordinary in-memory application structs should normally use target-native layout instead.
