# Encoding Guide (v2)

This guide covers scalar, reference, and vector encoding behavior for Recurram v2. It is detailed by encode-time rule so implementations can stay deterministic and interoperable.

## 1. Lengths and IDs

Lengths and ids use varuint for:

- dynamic lengths
- `key_id`, `str_id`, `shape_id`
- `base_id`, `template_id`, and other state ids when session features are enabled

Varuint domains in v2 are used for metadata, not for replacing fixed integer value tags.

## 2. Scalar Rules

### 2.1 Integers

- Use fixint for `-32..127` first.
- Otherwise use fixed-width `i8/i16/i32/i64` or `u8/u16/u32/u64`.
- Encoder SHOULD choose smallest valid width.

Recommended width order:

- signed: `fixint` -> `i8` -> `i16` -> `i32` -> `i64`
- unsigned: `fixint` -> `u8` -> `u16` -> `u32` -> `u64`

### 2.2 Float

- scalar float uses `f64` (`0xC3`).

### 2.3 Strings and Binary

- `fixstr` for short strings (`<=31` bytes)
- `str8/str16/str32` for larger strings
- `bin8/bin16/bin32` for binary

Length tags MUST match actual payload byte length exactly.

## 3. Per-Message Interning

### 3.1 Keys

Literal map keys are registered in first-seen order and may be replaced with `key_ref`.

Registration order is part of deterministic behavior and cannot be implementation-random.

### 3.2 String Values

Literal string values are registered in first-seen order and may be replaced with `str_ref`.

Interning state resets at each top-level message boundary.

Unknown `key_ref`/`str_ref` ids MUST fail decode.

### 3.3 Shape IDs

Shape ids are message-local and first-seen assigned when `shape_def` appears. `shape_ref` may only target prior shape ids in the same top-level message.

## 4. Typed Vector Encoding

`typed_vec` payload:

```text
0xDA [element_type][count][codec][payload]
```

Supported numeric/vector codecs include:

- `DIRECT_BITPACK`
- `DELTA_BITPACK`
- `FOR_BITPACK`
- `DELTA_FOR_BITPACK`
- `DELTA_DELTA_BITPACK`
- `RLE`
- `PATCHED_FOR`
- `SIMPLE8B`
- `XOR_FLOAT` for float vectors

Codec choice SHOULD be deterministic for equal input statistics and equal profile configuration.

## 5. Shape-Optimized Arrays

For same-shape map arrays:

- emit one `shape_def`
- emit row values without repeated key literals

This optimization is valid within one message and does not require session state.

Fallback behavior:

- if shape stability is not detected, encode as regular maps
- if unsupported value appears mid-stream, encoder may fall back to generic map/array tags

## 6. Determinism

Implementations MUST keep deterministic encode decisions for identical input and state:

- same fix-family selection
- same intern id assignment order
- same vector codec tie-break behavior

Additional deterministic expectations:

- same traversal order for map key emission under the same runtime representation
- same shape id assignment for equivalent arrays
- same fallback threshold behavior when optional codecs are available

## 7. Compatibility Contract

- v2 is a clean break from v1 wire behavior.
- v2 decoders are not required to decode v1 bytes.
- If both versions are supported, select version explicitly outside this encoding layer.

## 8. Encode Error Conditions

Encoders should fail early on:

- integer overflow for selected numeric domain
- unsupported value type for target profile
- invalid reference construction (negative ids, out-of-range ids)
- inconsistent typed vector payload length or codec metadata

Decoders should fail early on:

- unknown `key_ref` / `str_ref` / `shape_ref`
- malformed length fields
- truncated payload
- mismatched container element counts
