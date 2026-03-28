# Format Guide

This document explains the wire-level shape of Recurram messages. It focuses on message kinds, payload layout patterns, and promotion paths between profiles.

## 1. Top-Level Message Kind

Every message starts with one byte `kind`.

| Kind   | Name             | Primary use                               |
| ------ | ---------------- | ----------------------------------------- |
| `0x00` | `SCALAR`         | One scalar root value                     |
| `0x01` | `ARRAY`          | Dynamic heterogeneous arrays              |
| `0x02` | `MAP`            | Dynamic object/map                        |
| `0x03` | `SHAPED_OBJECT`  | Dynamic object using `shape_id`           |
| `0x04` | `SCHEMA_OBJECT`  | Schema-bound object                       |
| `0x05` | `TYPED_VECTOR`   | Homogeneous typed array                   |
| `0x06` | `ROW_BATCH`      | Row-wise records with shared shape/schema |
| `0x07` | `COLUMN_BATCH`   | Columnar records with shared shape/schema |
| `0x08` | `CONTROL`        | Table registration and state control      |
| `0x09` | `EXT`            | Extension payload                         |
| `0x0A` | `STATE_PATCH`    | Delta payload against prior/base message  |
| `0x0B` | `TEMPLATE_BATCH` | Micro-batch using recent template         |
| `0x0C` | `CONTROL_STREAM` | Packed control lane (presence/ops/etc.)   |
| `0x0D` | `BASE_SNAPSHOT`  | Full payload registered as patch base     |

Implementations MAY support a subset, but wire behavior within a declared profile MUST remain deterministic.

## 2. Profile-Oriented Message Forms

### 2.1 Dynamic Profile

Dynamic mode starts with self-describing forms:

- `SCALAR`
- `ARRAY`
- `MAP`

As repetition appears, an encoder SHOULD promote to:

- `SHAPED_OBJECT` for repeated key sequences
- `TYPED_VECTOR` for homogeneous arrays

### 2.2 Bound Profile

Bound mode uses `SCHEMA_OBJECT` and omits key names and per-field type tags.

Canonical shape:

```text
[optional schema_id][presence bitmap?][field payloads...]
```

### 2.3 Batch Profile

Two batch forms are available for repeated rows of one shape/schema.

`ROW_BATCH`:

```text
[count][row_0][row_1]...[row_n-1]
```

`COLUMN_BATCH`:

```text
[count]
[column_0 header][column_0 payload]
[column_1 header][column_1 payload]
...
```

### 2.4 Stateful Profile

Stateful mode adds reuse-oriented forms:

- `BASE_SNAPSHOT`
- `STATE_PATCH`
- `TEMPLATE_BATCH`
- optional `CONTROL_STREAM`

On state mismatch or unsupported transport assumptions, implementations SHOULD fall back to stateless full messages.

## 3. Dynamic Object Forms

### 3.1 `MAP`

`MAP` is the baseline dynamic object representation.

```text
[field_count][entries...]
```

Each entry is either:

- key literal + value
- `key_id` + value

### 3.2 `SHAPED_OBJECT`

`SHAPED_OBJECT` references a previously registered key sequence.

```text
[shape_id][presence bitmap?][values...]
```

This removes repeated key literals and map structure bytes from subsequent objects.

## 4. Array Forms

### 4.1 `ARRAY`

Used for heterogeneous arrays and short one-shot arrays.

### 4.2 `TYPED_VECTOR`

Used when all elements share one physical type.

```text
[element_type][count][codec][payload]
```

This allows dense numeric codecs and compact bool/string vector forms.

## 5. Batch Forms

### 5.1 Row-Wise Batch

`ROW_BATCH` is simpler and often preferred for low latency or small counts.

### 5.2 Columnar Batch

`COLUMN_BATCH` is usually better when columns have strong statistical structure.

Each column header contains at least:

- field identity
- null strategy
- codec
- optional dictionary metadata

## 6. Stateful Forms

### 6.1 `BASE_SNAPSHOT`

Registers a full payload as a reusable base.

```text
[base_id][schema_or_shape_ref][payload]
```

### 6.2 `STATE_PATCH`

Sends only changed parts against `previous` or explicit `base_id`.

```text
[base_ref][patch_opcode_stream][changed_fields][optional literals]
```

### 6.3 `TEMPLATE_BATCH`

Reuses recent batch headers for small bursts.

```text
[template_id][count][changed-column-mask][column payloads]
```

## 7. Control Messages

`CONTROL` carries table updates and resets, including operations like:

- `REGISTER_KEYS`
- `REGISTER_SHAPE`
- string/enum promotion controls
- `RESET_TABLES`
- `RESET_STATE`

`CONTROL_STREAM` MAY be used when many control bits/opcodes need separate entropy-friendly packing.

## 8. Encoder Promotion Flow (Informative)

A typical deterministic progression is:

1. Start with `MAP` / `ARRAY` / `SCALAR`.
2. Intern keys and shapes as repetition appears.
3. Promote homogeneous arrays to `TYPED_VECTOR`.
4. Promote repeated rows to `ROW_BATCH` or `COLUMN_BATCH`.
5. In session mode, promote similar messages to `STATE_PATCH`.

See `docs/encoding.md` for codec details and `docs/transport.md` for state synchronization rules.
