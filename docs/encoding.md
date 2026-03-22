# Encoding Guide

This document describes scalar, vector, string, and compression encoding rules used by Gowe. It complements `docs/format.md` (message forms) and `docs/transport.md` (state behavior).

## 1. Metadata Integers

Lengths, counts, and ids SHOULD use `Gowe-PV` varuint unless a profile explicitly overrides it.

Typical targets:

- lengths and counts
- `key_id`, `shape_id`, `string_id`
- `schema_id`, `template_id`, `base_id`

## 2. Scalar Numbers

### 2.1 Integer Scalars

For one-shot integer values, use smallest-width integer selection.

- `0..255` -> `uint8`
- `0..65535` -> `uint16`
- `0..2^32-1` -> `uint32`
- larger -> `uint64`

Negative integers MAY use zigzag transform plus smallest-width storage.

### 2.2 Range-Aware Packing

When schema bounds are known, encode `value - min` with minimal bit width.

```text
bits = ceil(log2(max - min + 1))
```

## 3. Presence and Null

Optional fields use a presence bitmap by default.

- `1` = present
- `0` = absent

An inverted mode MAY be used when absent bits are sparse. If all optional fields are guaranteed present by profile/schema, the bitmap MAY be omitted.

## 4. Boolean and Enum

- `bool` values are bit-packed as 1 bit each.
- enum values use `ceil(log2(N))` bits for `N` symbols.
- bool/enum streams in vector or batch contexts MAY be moved into `CONTROL_STREAM`.

## 5. Integer Vector Codecs

Integer vectors/columns MAY use the following codecs:

- `DIRECT_BITPACK`
- `DELTA_BITPACK`
- `FOR_BITPACK`
- `DELTA_FOR_BITPACK`
- `DELTA_DELTA_BITPACK`
- `RLE`
- `PATCHED_FOR`
- `SIMPLE8B`

### 5.1 Codec Selection Heuristics

Practical default order:

1. `DELTA_DELTA_BITPACK` for near-constant adjacent deltas (timestamps).
2. `RLE` for long repeated runs.
3. `FOR_BITPACK` for tightly clustered values.
4. `DELTA_FOR_BITPACK` for monotonic or nearly monotonic sequences.
5. `DELTA_BITPACK` when delta width is much smaller than plain width.
6. `PATCHED_FOR` when most values fit and few overflow.
7. `DIRECT_BITPACK` when width is still smaller than plain.
8. `PLAIN` for tiny blocks or non-beneficial cases.

## 6. Float Vector Codecs

### 6.1 `XOR_FLOAT`

`XOR_FLOAT` is recommended for smooth float series.

High-level flow:

1. Emit first value as full 64-bit payload.
2. XOR each subsequent value with previous value bits.
3. Emit compact control for zero/non-zero XOR.
4. For non-zero XOR, emit leading/trailing zero spans and meaningful middle bits.

Fallback to plain float encoding or generic compression when XOR residuals are unstable.

## 7. String Modes

Gowe supports multiple string modes to reduce repeated literal cost.

- `EMPTY`
- `LITERAL`
- `REF`
- `PREFIX_DELTA`
- `INLINE_ENUM`

### 7.1 `LITERAL`

```text
[length][utf8 bytes]
```

### 7.2 `REF`

```text
[string_id]
```

References a previously registered string.

### 7.3 `PREFIX_DELTA`

```text
[base_ref][prefix_len][suffix bytes]
```

Effective for related paths, ids, or shared textual prefixes.

### 7.4 `INLINE_ENUM`

A string field with stable low cardinality MAY be promoted to compact enum code form.

## 8. Dictionary Scope

String dictionaries MAY be maintained at:

- stream scope
- shape/schema scope
- field-local scope

Field-local dictionaries are often best for low-cardinality columns. Static dictionaries MAY be predefined by profile for common vocabularies.

## 9. Typed Vector Payloads

`TYPED_VECTOR` combines type metadata with a vector codec.

```text
[element_type][count][codec][payload]
```

Typical payload families:

- bit-packed bool streams
- integer codecs listed above
- float `XOR_FLOAT` or plain float blocks
- string dictionary/offset/prefix-delta forms

## 10. Zero Packing

For fixed-width payloads with many zeros, optional word-level zero packing MAY be used.

Mechanism:

- one tag byte covers one 8-byte word
- tag bit `1` keeps literal byte
- tag bit `0` implies zero byte

This is useful for sparse/default-heavy fixed layouts.

## 11. Generic Compression Layer

After data-aware encoding, a block-level generic compressor MAY be applied.

Recommended defaults:

- `LZ4` for low-latency transport
- `zstd` for larger blocks
- zstd with trained dictionaries for stable small-message families

Compression SHOULD be skipped for very small or already dense blocks.

## 12. Determinism Requirement

Within one fixed profile and equivalent learning/session state, encoder decisions MUST be deterministic.

Implementations MAY score candidate codecs, but tie-break behavior MUST be fixed.
