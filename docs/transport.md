# Transport Guide

This document explains transport and session behavior for Recurram, especially when stateful compression is enabled. Message layout details are in `docs/format.md`; scalar/codec behavior is in `docs/encoding.md`.

## 1. Transport Assumptions

Recurram can operate in stateless or stateful mode.

- Stateless mode works on any transport and sends self-sufficient messages.
- Stateful mode reuses learned/session data and requires stronger ordering guarantees.

If a transport cannot preserve state assumptions, implementations SHOULD keep or fall back to stateless mode.

## 2. Session State Model

In stateful mode, encoder and decoder share session-local state.

State may include:

- key table
- shape table
- string table
- field-local dictionaries
- `base_id` snapshots
- `template_id` descriptors
- optional trained dictionary references

Session state MUST NOT be implicitly reused across independent streams/connections.

## 3. Stateful Message Types

### 3.1 `BASE_SNAPSHOT`

Registers a full payload as a future patch base.

```text
[base_id][schema_or_shape_ref][payload]
```

### 3.2 `STATE_PATCH`

Encodes only changed parts against previous message or explicit base.

```text
[base_ref][patch_opcode_stream][changed_fields][optional literals]
```

Typical opcodes include `KEEP`, `REPLACE_*`, `APPEND_VECTOR`, and string-ref operations.

### 3.3 `TEMPLATE_BATCH`

Reuses recent batch metadata for small bursts of similar records.

```text
[template_id][count][changed-column-mask][column payloads]
```

### 3.4 `CONTROL_STREAM`

Moves dense control lanes (presence/opcodes/enum streams) into a separately compressed stream.

## 4. Synchronization and Resets

### 4.1 Table Reset

`RESET_TABLES` clears dynamic interning tables (`key_id`, `shape_id`, `string_id`).

### 4.2 Full State Reset

`RESET_STATE` invalidates patch/template/dictionary references.

After `RESET_STATE`, both sides MUST treat prior `base_id`, `template_id`, and session dictionary bindings as unusable.

## 5. Unknown Reference Handling

When a decoder receives unknown `base_id`/`template_id`/`dict_id`, profile behavior MUST be predefined.

Allowed policies:

- fail-fast with explicit error
- request or retry with stateless full message

A deployment SHOULD choose one policy and keep it stable.

## 6. Ordering and Reliability

Stateful mode depends on consistent message ordering.

Required properties:

- deterministic ordering for state-mutating messages
- no silent loss of `CONTROL`, `BASE_SNAPSHOT`, or reset frames
- bounded retention policy for `base_id` and `template_id`

If packet loss or reordering is possible, the transport layer SHOULD provide sequencing/ack mechanisms, or encoder SHOULD disable stateful forms.

## 7. Retention Windows

Encoders and decoders SHOULD agree on retention limits.

Common controls:

- max snapshot count (`maxBaseSnapshots`)
- max template count
- optional dictionary version pinning

Evicted references MUST NOT be used by either side.

## 8. Fallback Strategy

A robust transport strategy is:

1. Start in stateless mode.
2. Enable stateful forms after capability and stability checks.
3. If divergence is detected, send `RESET_STATE` and one full stateless message.
4. Resume stateful mode only after tables/bases are synchronized again.

This preserves interoperability while still enabling strong compression gains.

## 9. Security and Operational Notes

- Validate all ids before dereference.
- Enforce size limits for tables, dictionaries, and patch payloads.
- Apply timeouts for sessions with no forward progress.
- Log reset and fallback events for operational diagnosis.

These controls reduce risk from malformed or adversarial input while keeping stateful operation predictable.
