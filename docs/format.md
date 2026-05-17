# Format Guide (v2)

This document describes the Twilic v2 wire layout in practical terms. It mirrors the normative rules in `SPEC.md`, but keeps the focus on byte structure and decode shape.

## 1. Wire Model Overview

v2 uses a tag-table model. The first byte is always a tag family or fixed literal tag. Unlike v1, there is no top-level message-kind envelope byte.

### 1.1 First-byte compact families

- `0x00..0x7F`: positive fixint (`0..127`)
- `0x80..0x9F`: fixstr (`len=0..31`)
- `0xA0..0xAF`: fixarray (`count=0..15`)
- `0xB0..0xBF`: fixmap (`count=0..15`)
- `0xE0..0xFF`: negative fixint (`-32..-1`)

### 1.2 Extended tags (`0xC0..0xDF`)

| Tag / Range  | Meaning           |
| ------------ | ----------------- |
| `0xC0`       | null              |
| `0xC1`       | false             |
| `0xC2`       | true              |
| `0xC3`       | float64 LE        |
| `0xC4..0xC7` | u8/u16/u32/u64    |
| `0xC8..0xCB` | i8/i16/i32/i64    |
| `0xCC..0xCE` | bin8/bin16/bin32  |
| `0xCF..0xD1` | str8/str16/str32  |
| `0xD2..0xD3` | array16/array32   |
| `0xD4..0xD5` | map16/map32       |
| `0xD6`       | `shape_def`       |
| `0xD7`       | `shape_ref`       |
| `0xD8`       | `key_ref`         |
| `0xD9`       | `str_ref`         |
| `0xDA`       | `typed_vec`       |
| `0xDB`       | `row_batch`       |
| `0xDC`       | `col_batch`       |
| `0xDD`       | `state_patch`     |
| `0xDE`       | `template_batch`  |
| `0xDF`       | extension (`ext`) |

## 2. Dynamic Container Shapes

### 2.1 Map body

Canonical map payload:

```text
[fixmap|map16|map32][key][value]...
```

Key representations:

- key literal (`fixstr` / `str8` / `str16` / `str32`)
- `key_ref`: `0xD8 [varuint key_id]`

Unknown `key_ref` id is a hard decode error.

### 2.2 Array body

Canonical array payload:

```text
[fixarray|array16|array32][value_0][value_1]...
```

Array may remain generic, or be promoted to:

- `typed_vec` for homogeneous primitive arrays
- shape forms for homogeneous map arrays

## 3. Message-Local Reuse Forms

v2 does not require session state for structural reuse in one message.

### 3.1 `shape_def` (`0xD6`)

Defines an ordered key sequence and registers `shape_id` in the current message.

```text
0xD6 [shape_id][key_count][key_0]...[key_n]
```

### 3.2 `shape_ref` (`0xD7`)

References prior shape in the current message and encodes only values.

```text
0xD7 [shape_id][value_0]...[value_n]
```

Unknown `shape_ref` id is a decode error.

### 3.3 `key_ref` / `str_ref`

- `key_ref` (`0xD8`) references key literals already emitted in the message.
- `str_ref` (`0xD9`) references string value literals already emitted in the message.

All intern tables (`key_id`, `str_id`, `shape_id`) reset at each top-level message boundary.

## 4. Batch and Stateful Forms

### 4.1 Batch

- `0xDB`: `row_batch`
- `0xDC`: `col_batch`

Both are valid v2 tags and optional to implement.

### 4.2 Stateful

- `0xDD`: `state_patch`
- `0xDE`: `template_batch`

These forms may carry session references (`base_id`, `template_id`, dictionary ids) and therefore require transport-level state agreement.

## 5. Informative Encode Promotion Flow

Typical encoder decisions:

1. Emit fix families whenever representable.
2. Register key/string literals in message-local tables.
3. Replace repeats with `key_ref`/`str_ref`.
4. Detect homogeneous map arrays and emit `shape_def` plus value rows.
5. Promote homogeneous primitive arrays to `typed_vec`.
6. Use stateful forms only when session guarantees exist.

## 6. Compatibility and Versioning

- v2 is a clean break from v1 wire framing.
- v2 decoders are not required to decode v1 payloads.
- Implementations supporting both versions should use an explicit external version discriminator.
