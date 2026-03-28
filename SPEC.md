# Recurram Specification v1

## 1. Purpose

This specification defines a binary format that keeps MessagePack-like usability while aiming for smaller representations than MessagePack, especially under the following conditions.

- Objects with the same shape appear repeatedly.
- Messages under the same schema appear repeatedly.
- Repeated strings and similar strings are common.
- Homogeneous arrays and typed vectors are common.
- Multiple records with the same schema can be sent together.
- Optional-field distributions and integer series are biased.

The goal is to combine self-describing, schema-less convenience with the compression efficiency of schema-aware, columnar, dictionary, and bit-packing approaches in a single format family.

---

## 2. Design Principles

This specification follows these principles.

1. **Usable through the same API**
   - Provide a dynamic mode that can encode/decode arbitrary values as-is.
   - Allow promotion to compact mode when a schema is provided.

2. **Deferred optimization**
   - The first transmission may be self-describing.
   - Once repeated shape/key set/string/field distribution is observed, automatically switch to a more compact representation.

3. **Minimize one-shot cost**
   - Do not introduce excessive control metadata for small one-shot values.
   - Control information used for learning or dictionaries must be local and recoverable.

4. **Win decisively on repetition**
   - shape interning
   - key interning
   - string interning
   - typed vector
   - columnar batch
   - delta / frame-of-reference / dictionary / RLE

5. **Deterministic wire format**
   - Under the same profile and learning state, the same value maps to the same bytes.

6. **Optional stateful optimization**
   - Stateless one-shot messages remain directly usable.
   - Stateful patching/shared dictionary/template reuse can be used only when a session or channel exists.
   - Receivers that do not use stateful mode must still be able to fall back to stateless mode.

---

## 3. Profiles

This specification defines three primary profiles.

### 3.1 Dynamic Profile

A MessagePack-like profile.

- Any root value can be sent.
- No schema definition is required.
- Map/list/scalar/binary/string can be represented directly.
- If shape/key/string tables exist, compact forms may be used automatically.
- Otherwise, fall back to self-describing forms.

### 3.2 Bound Profile

A profile based on a shared schema.

- Message type is unique by schema id or context.
- Field order is fixed.
- Field names are not sent.
- Type tags are generally not sent.
- Range-aware bit packing is used.

### 3.3 Batch Profile

A profile that bundles multiple records with the same shape or schema.

- row-wise batch
- columnar batch
- per-column codec selection
- optional generic compression

### 3.4 Stateful Profile

A profile that compresses against previous messages and shared dictionaries over the same stream/session/channel.

- previous-message patch
- base snapshot reference
- session-local template reuse
- trained dictionary reference
- control-stream entropy coding
- micro-batch reuse

This profile is optional. On transports that cannot support it, Dynamic/Bound/Batch alone must be sufficient.

---

## 4. Message Kinds

The wire top level begins with a 1-byte kind.

| Kind | Name           | Purpose                                           |
| ---- | -------------- | ------------------------------------------------- |
| 0x00 | SCALAR         | scalar root                                       |
| 0x01 | ARRAY          | dynamic heterogeneous array                       |
| 0x02 | MAP            | dynamic object / map                              |
| 0x03 | SHAPED_OBJECT  | object using shape_id                             |
| 0x04 | SCHEMA_OBJECT  | schema-aware object                               |
| 0x05 | TYPED_VECTOR   | homogeneous typed array                           |
| 0x06 | ROW_BATCH      | row-wise batch                                    |
| 0x07 | COLUMN_BATCH   | columnar batch                                    |
| 0x08 | CONTROL        | key/shape/string table updates                    |
| 0x09 | EXT            | extension type                                    |
| 0x0A | STATE_PATCH    | delta message vs previous or specified base       |
| 0x0B | TEMPLATE_BATCH | micro-batch based on template/schema/shape        |
| 0x0C | CONTROL_STREAM | control lane for presence/enum/opcode and similar |
| 0x0D | BASE_SNAPSHOT  | snapshot used as the base for subsequent patches  |

Multiple profiles may coexist on the same transport.

---

## 5. Dynamic Profile

## 5.1 Purpose

The Dynamic Profile provides MessagePack-like usability where objects/arrays/scalars can be handled directly without a predeclared schema.

Unlike MessagePack, when repetition is observed, shape/key/string interning can automatically promote encoding to compact forms.

## 5.2 key table

Maintain a key table per stream.

- First-seen keys are sent as literals.
- Registered keys may be referenced by key_id.
- key_id uses a small varuint.

### 5.2.1 REGISTER_KEYS

Multiple keys may be pre-registered through a control message.

```text
CONTROL REGISTER_KEYS
[count]
[key_0]
[key_1]
...
```

## 5.3 shape table

A shape represents an object's key sequence.

Example:

```text
["id", "name", "admin"]
```

Objects that repeat the same key ordering can be represented by shape_id.

### 5.3.1 REGISTER_SHAPE

```text
CONTROL REGISTER_SHAPE
[shape_id]
[field_count]
[key_id or key literal]...
```

### 5.3.2 SHAPED_OBJECT

```text
[shape_id][presence bitmap?][values...]
```

This avoids resending key strings and map structure from the second occurrence onward, even for dynamic objects.

## 5.4 MAP

Objects whose shape is not registered are sent as MAP.

```text
[field_count][entries...]
```

Each entry is encoded as one of:

- literal key + value
- key_id + value

Implementations may send REGISTER_SHAPE and promote to SHAPED_OBJECT once the same key sequence repeats.

## 5.5 ARRAY

Heterogeneous arrays are sent as ARRAY.

If homogeneous structure is detectable, promotion to TYPED_VECTOR is allowed.

---

## 6. Bound Profile

## 6.1 schema

A schema-aware object assumes a shared schema.

Each field has at least:

- field number
- field name
- logical type
- physical encoding
- required / optional
- default value
- value range or allowed set
- string constraints

## 6.2 schema_id

If the message type is unique in context, schema_id may be omitted.

Attach schema_id only when mixed message types must be disambiguated.

## 6.3 SCHEMA_OBJECT

```text
[optional schema_id][presence bitmap?][field payloads...]
```

- required fields are always sent
- optional fields follow the presence bitmap
- field numbers and field names are not sent

---

## 6.4 Zero-Copy Layout (optional)

In Bound Profile implementations that require extremely fast decode/encode, a **zero-copy layout** may be selected. In this layout, in-memory structure and wire layout are made nearly identical. Fields are aligned naturally, and offsets/pointers are represented as relative values so data can be memory-mapped and accessed directly. This follows a Cap'n Proto-like design philosophy and reduces extra decode/encode work. However, because padding and pointer overhead can increase size, compact layout should be preferred when minimizing size is the primary goal. Zero-copy layout is an optional extension within Bound Profile.

---

## 7. Null / Presence

## 7.1 presence bitmap

Use a presence bitmap for objects with optional fields.

- 1 = present
- 0 = absent

## 7.2 inverted presence bitmap

If most fields are present, a 1-bit invert flag is allowed.

- 0 = normal
- 1 = inverted

In inverted mode, interpret 0 = present and 1 = absent.

## 7.3 all-present elision

Even with optional fields, if all fields are known to be present, the presence bitmap may be omitted.

This behavior must be fixed by schema or profile.

---

## 8. Numeric Encoding

## 8.1 scalar integer

The default for a single integer is smallest-width integer encoding.

- 0..255 -> uint8
- 0..65535 -> uint16
- 0..2^32-1 -> uint32
- above that -> uint64

Negative numbers may use zigzag plus smallest-width selection.

## 8.2 range-aware bit packing

When schema range is known, store `value - min` with the minimum required bit width.

```text
bits = ceil(log2(max - min + 1))
```

## 8.3 metadata varuint

Use Recurram-PV for metadata such as lengths, IDs, and counts.

Targets:

- lengths
- counts
- key_id
- string_id
- shape_id
- schema_id
- prefix_len

## 8.4 vector integer codecs

For integer series in batch/typed vectors, the following column codecs are allowed.

- DIRECT_BITPACK
- DELTA_BITPACK
- FOR_BITPACK
- DELTA_FOR_BITPACK
- DELTA_DELTA_BITPACK
- RLE
- PATCHED_FOR
- SIMPLE8B

### 8.4.1 DIRECT_BITPACK

Compute the maximum bit width in a block and pack with a fixed width.

### 8.4.2 DELTA_BITPACK

Take deltas first, then apply bit packing.

### 8.4.3 FOR_BITPACK

Subtract the block minimum, then bit-pack.

### 8.4.4 DELTA_FOR_BITPACK

Take deltas, then subtract the minimum delta in the block, then bit-pack.

### 8.4.5 PATCHED_FOR

Use a patch-list scheme when most values fit in a small bit width and only some values overflow.

Implementations may adopt ORC/Parquet-style patched-base strategies.

### 8.4.6 DELTA_DELTA_BITPACK

**Delta-of-delta bit packing** is a codec for efficiently representing integer series where adjacent deltas are nearly constant, such as time series. Send the first two values and the first delta, then encode `delta_i - delta_{i-1}` with zigzag and store continuously with minimal required bit width. This is especially effective for monotonic increases and regular-step sequences.

### 8.4.7 SIMPLE8B

**Simple-8b** packs variable-length integers into 60-bit blocks, using a 4-bit header per block to indicate value count and per-value bit width. Because all integers in a block share one bit width, headers are simple and decoding is fast. When outliers exist, other codecs may be more advantageous.

---

## 8.5 float vector codecs

Floating-point series have error characteristics different from integer series, so specialized codecs are used. In time-series data especially, adjacent values are often similar, and XOR-based compression is effective.

### 8.5.1 XOR_FLOAT

The **XOR_FLOAT** codec converts each 64-bit float to its bit pattern and XORs consecutive values to detect similarity. The first value is sent as a full 64 bits. For subsequent values, XOR is taken against the previous value; if the result is zero, only a 1-bit flag is emitted. If nonzero, encode leading/trailing zero lengths of the XOR word and send only meaningful bits. This is particularly effective for smoothly changing series.

- Prefer XOR_FLOAT for time series with small incremental changes.
- If values fluctuate widely and XOR differences are unstable, prefer plain or generic compression.

---

## 9. Boolean / Enum

## 9.1 bool

Store bool in 1 bit.

## 9.2 enum

Store enum in `ceil(log2(N))` bits.

## 9.3 bool stream

In batch/typed vectors, bool columns may be emitted as a bitstream and then processed with byte-RLE or generic compression.

---

## 10. Strings

## 10.1 string policy

Because strings are often a weak point of MessagePack, the following modes are defined.

- EMPTY
- LITERAL
- REF
- PREFIX_DELTA
- INLINE_ENUM

## 10.2 LITERAL

```text
[length][utf8 bytes]
```

Send first-seen strings as-is.

## 10.3 REF

```text
[string_id]
```

Reference previously seen strings.

## 10.4 PREFIX_DELTA

```text
[base_ref][prefix_len][suffix bytes]
```

Suffix length must be derivable from the outer field frame or vector boundary.

## 10.5 string table

A string table may be kept at stream scope or shape/schema scope.

- literals may be registered
- reconstructed prefix_delta results may also be registered
- REF may point only to already registered values

## 10.6 field-local dictionary

When repeated strings concentrate in a specific field, a field-local dictionary may be used.

This often yields a smaller id width than a global string table.

## 10.7 static dictionary

If a profile/application has fixed vocabularies, a static dictionary may be predefined.

Examples:

- status
- method
- role
- locale
- country

## 10.8 INLINE_ENUM

Even in dynamic mode, if a string field takes only a small fixed set of values, it may be promoted to inline enum by control message.

```text
CONTROL PROMOTE_STRING_FIELD_TO_ENUM
[field identity]
[value_count]
[string literals...]
```

After that, the field is sent using compact integer codes.

---

## 11. binary / string pooling

In Dynamic Profile, FlexBuffers-like automatic pooling is allowed.

- repeated key strings
- repeated string values
- repeated binary blobs

Also, offset/width/id/count may be automatically narrowed to smallest-width 8/16/32/64.

---

## 12. TYPED_VECTOR

## 12.1 Purpose

Homogeneous arrays are usually smaller when represented as TYPED_VECTOR instead of ARRAY.

## 12.2 header

```text
[element_type][count][codec][payload]
```

## 12.3 benefits

- per-element type tags can be omitted
- bit packing/delta/FOR can be applied to homogeneous numeric series
- bool series can be emitted as bitstreams
- repeated scalar lists can be packed

## 12.4 typed string vector

For string arrays, the following are allowed.

- offsets + values
- dictionary ids
- prefix_delta sequence
- run-end encoding for repeated strings

---

## 13. Batch

## 13.1 ROW_BATCH

Concatenate records with the same shape/schema in row-wise form.

```text
[count][row_0][row_1]...[row_n-1]
```

## 13.2 COLUMN_BATCH

Store records with the same shape/schema in column-wise form.

```text
[count]
[column_0 header][column_0 payload]
[column_1 header][column_1 payload]
...
```

Each column header includes:

- field identity
- null strategy
- codec
- optional dictionary info

## 13.3 column codecs

Each column may choose from:

- PLAIN
- BITPACK
- DELTA
- FOR
- DELTA_FOR
- RLE
- DICTIONARY
- PATCHED_FOR
- STRING_REF
- PREFIX_DELTA

## 13.4 codec selection guidance

Implementations may choose codecs based on statistics.

- monotonic integers -> DELTA / DELTA_FOR
- clustered integers -> FOR
- low-cardinality strings -> DICTIONARY
- repeated values -> RLE or RUN_END
- mostly-null / mostly-present -> bitmap with inversion

---

## 13.5 Stateful Transport Extensions

## 13.5.1 session state

In stateful mode, encoder/decoder share session state.

State may include at least:

- shape table
- key table
- string table
- field-local dictionary
- recent base snapshots
- recent template ids
- optional trained compression dictionary id

State is session-local and must not be implicitly inherited across transports.

## 13.5.2 BASE_SNAPSHOT

`BASE_SNAPSHOT` is a full message used as a base for later patch references.

```text
[base_id][schema_or_shape_ref][payload]
```

The encoder may register reusable objects/rows/batches as snapshots.

The decoder must maintain a base-retention window, and base_id outside the window must be unreferenceable.

## 13.5.3 STATE_PATCH

`STATE_PATCH` sends only the delta from the previous message or a specified `base_id`.

```text
[base_ref][patch_opcode_stream][changed_fields][optional appended literals]
```

Patch opcodes may define at least:

- KEEP
- REPLACE_SCALAR
- REPLACE_VECTOR
- APPEND_VECTOR
- TRUNCATE_VECTOR
- DELETE_FIELD
- INSERT_FIELD
- STRING_REF
- PREFIX_DELTA

In Bound Profile where field order is fixed, patches may target field position instead of field number.

## 13.5.4 previous-message patch

When similarity to the immediately previous message is high, implicit previous-message may be used as `base_ref`.

This is highly effective for messages such as:

- status updates
- telemetry ticks
- periodically refreshed objects
- paginated responses containing cursor/offset
- nearly identical API response families

## 13.5.5 TEMPLATE_BATCH

To gain columnar benefits without waiting for a large batch, define `TEMPLATE_BATCH`.

```text
[template_id][count][changed-column-mask][column payloads]
```

`template_id` represents the same schema/shape/null strategy/codec set as a recent batch.

Unchanged column headers are not retransmitted.

## 13.5.6 CONTROL_STREAM

Control information such as presence bitmaps, enum streams, patch opcode streams, and string-mode streams may be separated from data payload and grouped into `CONTROL_STREAM`.

The following may be applied to the control stream.

- RLE
- bitpack
- Huffman
- FSE

By compacting frequent opcodes such as KEEP/PRESENT/SAME_STRING, stateful patch efficiency can be maximized.

## 13.5.7 trained dictionary reference

In small-message families, a trained dictionary may be referenced by `dict_id`.

```text
[dict_id][compressed block]
```

Dictionaries are recommended to be separated per data family. Methods for dictionary training, distribution, and invalidation must be fixed by transport profile.

## 13.5.8 RESET_STATE

To prevent state divergence, the encoder may send `RESET_STATE` control at any time.

After reset, the decoder must treat all base snapshot/template/dictionary references as invalid.

---

## 14. zero packing

For row-wise objects and fixed-width payloads, word-level zero packing may be applied optionally.

Method:

- 1-byte tag per 8-byte word
- bytes with tag bit = 1 are stored literally in payload
- bytes with tag bit = 0 are interpreted as zero
- special handling may be defined for all-zero/all-nonzero words

Use cases:

- structs with many default values
- integer series with many high-order zero bytes
- sparse fixed-width payloads

---

## 15. generic compression

## 15.1 principle

Apply data-aware encoding first, then apply generic compression at block level.

## 15.2 recommendations

- small correlated records -> zstd dictionary
- high-speed transport -> LZ4
- larger blocks -> zstd
- very high ratio with reasonable speed -> FSE

## 15.3 dictionary scope

Dictionaries are recommended to be separated by data family.

Examples:

- user records dictionary
- log records dictionary
- event records dictionary
- paginated API response dictionary
- telemetry tick dictionary

## 15.4 trained dictionary transport

Dictionaries may be distributed statically or negotiated at session start.

For small messages, trained dictionaries can strongly compress correlated families, and may be preferred over stateless zstd.

A dictionary transport profile must include at least:

- dictionary id
- version
- hash
- expiration/invalidation rule
- fallback behavior

---

## 16. API Policy For Usability

Implementations of this specification are recommended to provide at least these APIs.

```ts
encode(value);
encodeWithSchema(schema, value);
encodeBatch(schemaOrShape, values);
createSessionEncoder(options);
```

### 16.1 contractless mode

`encode(value)` accepts object/array/scalar directly like MessagePack.

Internal optimization order:

1. represent as dynamic scalar/map/array
2. use key_id if key table exists
3. use SHAPED_OBJECT if shape exists
4. use TYPED_VECTOR if homogeneous array
5. use ROW_BATCH or COLUMN_BATCH when batching is possible

### 16.2 schema mode

`encodeWithSchema(schema, value)` uses Bound Profile.

### 16.3 batch mode

`encodeBatch(...)` selects per-column codecs based on column statistics.

### 16.4 session encoder mode

`createSessionEncoder(options)` returns an encoder that handles stateful profiles.

Typical API:

```ts
const enc = createSessionEncoder({
  maxBaseSnapshots: 8,
  enableStatePatch: true,
  enableTemplateBatch: true,
  enableTrainedDictionary: true,
});

enc.encode(value);
enc.encodePatch(value);
enc.encodeMicroBatch(values);
enc.reset();
```

The session encoder may automatically choose stateless or stateful mode based on previous-message similarity, recent base/template, and dictionary state.

---

## 17. Compatibility

## 17.1 Dynamic Profile

- key_id, shape_id, and string_id may be treated as session-local
- define a RESET_TABLES control message
- fix per profile whether unknown control messages are ignored or fail-fast

## 17.2 Stateful Profile

- base_id, template_id, and dict_id are session-local
- when decoder receives unknown base_id/template_id/dict_id, behavior must be fixed as either fail-fast or stateless retry
- when state divergence is detected, encoder must be able to send RESET_STATE followed by a stateless full message
- on transports with packet loss, ordering/ack/epoch management required for stateful mode must be guaranteed by the transport side

## 17.3 Bound Profile

- field order is fixed
- appending optional fields at the tail is allowed
- removed fields must be reserved and their index must not be reused
- narrowing value ranges is a breaking change

---

## 18. Encoder Auto-Selection Rules

Implementations are recommended to observe value or batch statistics and automatically select representation in the order below.

Implementations may provide both lightweight and high-compression profiles, but decision rules must be deterministic within the same profile.

### 18.1 Basic policy

1. determine the shape of the root value first
2. classify into scalar/dynamic map/dynamic array/schema object/batch/stateful patch
3. prioritize interning when repetition can be detected
4. prioritize typed representation when homogeneous data is visible
5. prioritize columnar over row-wise when batch is sufficiently large
6. choose integer/string/float codec based on column statistics
7. consider stateful patch when overlap with previous message or recent base is large
8. apply generic compression last at block level

### 18.1A Stateful selection

#### Rule ST1: previous-message patch

Use `STATE_PATCH(previous)` as a candidate when:

- same schema / same shape
- changed_field_ratio <= 0.25
- appended_literal_bytes is sufficiently smaller than full_message_bytes

Strongly recommended thresholds:

- changed_field_ratio <= 0.10
- most fields can be expressed as KEEP

#### Rule ST2: base snapshot patch

Even when not against the previous message, if estimated patch size against a recent base snapshot is below 70% of full message size, consider `STATE_PATCH(base_id)`.

#### Rule ST3: template batch

If 4 or more small messages of the same schema/shape gather in a short time but waiting for a full column batch is too costly, consider `TEMPLATE_BATCH`.

#### Rule ST4: trained dictionary

If the payload family is stable and single-message encoded size is between 128B and 8KB, consider trained dictionary.

#### Rule ST5: stateless fallback

Keep stateless full message if any of the following applies.

- transport may reorder or lose messages
- base synchronization is weak
- changed_field_ratio is high
- decoder capability is unknown

### 18.2 Dynamic object selection

For input objects, decide in this order.

#### Rule D1: promotion to schema object

- if the call is `encodeWithSchema(schema, value)`, always use SCHEMA_OBJECT
- even in `encode(value)`, SCHEMA_OBJECT may be used if implementation can reliably determine schema for the object

#### Rule D2: promotion to shaped object

If the same key sequence is observed at least twice in a stream, treat it as a shape-registration candidate.

When all conditions below hold, send REGISTER_SHAPE and then use SHAPED_OBJECT.

- field_count >= 2
- same key sequence observation count >= 2
- `sum(key_literal_sizes) >= shape_registration_cost / expected_reuse_count`

For a simplified profile, the following threshold is acceptable.

- if field_count >= 3 and same key sequence appears >= 2 times, register shape

#### Rule D3: keep MAP

Keep MAP when any of the following is true.

- field_count <= 1
- key sequence is not reused
- object is estimated to be one-shot
- registration cost exceeds reuse benefit

### 18.3 key table selection

#### Rule K1: key interning

If one key literal is observed at least twice in the stream, treat it as a key-table registration candidate.

Recommended thresholds:

- key byte length >= 3
- same key occurrences >= 2

Even short keys may be registered when occurrence count is high.

#### Rule K2: keep key literal

Literal form is acceptable when:

- key byte length <= 2 and occurrences are low
- temporary keys used only in that object

### 18.4 array selection

#### Rule A1: promotion to typed vector

If all elements fit a single physical type, consider TYPED_VECTOR.

Recommended thresholds:

- primitive homogeneous array with count >= 4
- strongly recommend typed vector for bool arrays when count >= 8
- integer arrays: candidate when count >= 4
- float arrays: candidate when count >= 4
- string arrays: typed string vector candidate when count >= 4

#### Rule A2: keep dynamic ARRAY

Keep ARRAY when:

- heterogeneous array
- count <= 3
- mixed scalar/object/array is common

### 18.5 Presence / Null selection

#### Rule P1: presence bitmap elision

Even with optional fields, if schema/profile guarantees all fields are present, the presence bitmap may be omitted.

#### Rule P2: inverted presence bitmap

Let optional-field count be M, present count P, absent count A.

- if A < P, consider inverted bitmap
- as a simple threshold, recommend inverted when `A <= M / 4`
- if `A = 0` and all-present elision is allowed, bitmap itself may be omitted

#### Rule P3: normal bitmap

- recommend normal bitmap when `A > M / 4`

### 18.6 integer selection

#### Rule I1: scalar integer

For one-shot integers, use smallest-width integer.

- 0..255 -> uint8
- 0..65535 -> uint16
- 0..2^32-1 -> uint32
- above that -> uint64

Use Recurram-PV only for metadata count/id/length.

#### Rule I2: schema-aware bounded integer

If schema provides min/max, always prioritize range-aware bit packing.

#### Rule I3: integer vector codec selection

For an integer vector or integer column, compute per block:

- count
- min
- max
- range = max - min
- adjacent deltas
- min_delta
- max_delta
- delta_range = max_delta - min_delta
- run count

Then choose codec in the following order.

##### Rule I3-0: DELTA_DELTA_BITPACK

- `count >= 8` and `non_zero_delta_of_delta_ratio <= 0.25`
- or `delta_range_bits <= 2`
- strongly recommended for nearly constant adjacent-delta series such as regular-interval timestamps

##### Rule I3-1: RLE

- average run length of the most frequent value >= 3
- or total_run_savings > direct_bitpack_savings
- simple threshold: `repeated_ratio >= 0.5` and average run length >= 3

##### Rule I3-2: FOR_BITPACK

- `bit_width(max - min) + header_cost < plain_width`
- simple threshold: `range_bits <= plain_bits - 4`

##### Rule I3-3: DELTA_FOR_BITPACK

- count >= 8
- delta_range_bits < range_bits
- monotonic or nearly monotonic
- simple threshold: `delta_range_bits <= range_bits - 3`

##### Rule I3-4: DELTA_BITPACK

- `max_abs_delta_bits <= plain_bits - 3`
- may be preferred when header is cheaper than FOR

##### Rule I3-5: PATCHED_FOR

- 90% or more fit in `base_width`
- patch_count / count <= 0.1

##### Rule I3-6: DIRECT_BITPACK

- does not match rules above but is still smaller than plain integer

##### Rule I3-7: PLAIN

- use PLAIN when count < 4
- keep plain when block is too small or header overhead does not pay off

##### Rule I3-8: SIMPLE8B

- `count >= 8`
- `max_bit_width <= 16`
- not monotonic and gains from delta-of-delta/FOR are limited
- little benefit from run-length or patched-base strategies

### 18.6A floating-point vector selection

#### Rule F1: XOR_FLOAT

- strongly recommended when adjacent deltas are small and average non-zero XOR bit width is <= 16 bits
- select when 50% or more of XOR differences are representable as 0 or 1 bit
- effective for temporally smooth values

#### Rule F2: PLAIN / generic

- if XOR bit widths vary widely and remain large, consider plain encoding or downstream generic compression
- when float values are integer-like and narrow in range, integer conversion plus integer codec may be used

### 18.7 string selection

For string fields or string vectors, observe:

- unique_count
- unique_ratio = unique_count / count
- average_length
- repeated_count
- prefix similarity
- static dictionary hit ratio

#### Rule S1: EMPTY

Use EMPTY when length is zero.

#### Rule S2: REF

If the same string exists in table, treat REF as candidate.

Recommended thresholds:

- `ref_cost < literal_cost`
- simple rule: recommend REF when string_id fits in 1 or 2 bytes and literal length >= 2

#### Rule S3: PREFIX_DELTA

Let longest common prefix length with base string be L and suffix length be S.

- L >= 2
- `base_ref_cost + prefix_len_cost + suffix_cost < literal_cost`
- simple threshold: `L >= 3` and `S <= literal_length - 2`

#### Rule S4: field-local dictionary

- count >= 8
- unique_ratio <= 0.5
- average_length >= 3
- strongly recommended threshold: count >= 16 and unique_ratio <= 0.25

#### Rule S5: static dictionary

- candidate when hit_ratio >= 0.5
- strongly recommended when hit_ratio >= 0.8

#### Rule S6: INLINE_ENUM

- count >= 16
- unique_count <= 8
- unique_ratio <= 0.25
- low new-value appearance rate

#### Rule S7: LITERAL

Use LITERAL when none of the above applies.

### 18.8 Batch selection

#### Rule B1: row batch

If there are at least 4 rows with the same shape/schema, consider ROW_BATCH.

#### Rule B2: column batch

If there are at least 16 rows with the same shape/schema, consider COLUMN_BATCH.

Further, strongly recommend COLUMN_BATCH when any of the following holds.

- integer column exists and delta/FOR is effective
- low-cardinality string column exists
- many optional fields exist
- count >= 32

#### Rule B3: keep row-wise

Keep row-wise when:

- count <= 3
- row shapes are not stable
- fields have high entropy and column codecs are unlikely to help
- low latency is prioritized and waiting for batch formation is not acceptable

### 18.9 zero packing selection

#### Rule Z1: apply zero packing

Zero packing may be applied when zero-byte ratio is high in fixed-width payload or row-wise structs.

Recommended thresholds:

- candidate when zero_byte_ratio >= 0.25
- strongly recommended when zero_byte_ratio >= 0.5

#### Rule Z2: do not apply zero packing

- payload is too small
- zero-byte ratio is low
- downstream compression is clearly superior

### 18.10 generic compression selection

#### Rule G1: conditions to apply generic compression

- encoded_block_size >= 256 bytes
- or batch contains multiple records

#### Rule G2: zstd dictionary

- small correlated records
- stable record family
- relatively small encoded_block_size where one-shot zstd tends to be less favorable

#### Rule G3: LZ4

Recommended when low latency is prioritized and moderate repetition exists.

#### Rule G4: no compression

- block is too small
- already compact enough from bit packing/dictionary/delta
- CPU cost constraints are strict

### 18.11 scoring method

Implementations may estimate size for each candidate codec and choose the smallest.

```text
score(codec) = estimated_encoded_size(codec) + switching_penalty(codec)
```

- `estimated_encoded_size(codec)` includes header, table updates, patch lists, dictionary, and bitmap total
- `switching_penalty(codec)` is a hysteresis term for profile stability or low-latency behavior

### 18.12 simplified recommended heuristics

For minimal implementations, the following simplified rules may be used.

- register shape after the same key sequence appears twice
- use typed vector for homogeneous primitive arrays with count >= 4
- use column batch for 16+ rows of the same schema
- use dictionary for string columns with unique_ratio <= 0.25
- use delta_for for monotonic integer columns
- use delta_delta_bitpack for regular integer sequences
- use xor_float for smooth float columns
- use RLE for columns with repeated_ratio >= 0.5
- use previous-message patch for same-schema objects with changed_field_ratio <= 0.10
- treat small messages in the 128B..8KB range within the same family as trained-dictionary candidates
- use zero packing for fixed payload with zero_byte_ratio >= 0.5
- treat encoded block >= 256 bytes as generic-compression candidate

---

## 19. Why It Beats MessagePack

The main reasons this specification outperforms MessagePack are:

1. object keys are not sent every time
2. repeated shapes can be encoded as shape_id
3. repeated strings can be encoded as string_id
4. homogeneous arrays can become typed vectors
5. bounded integers can be bit-packed
6. advanced integer codecs such as delta-of-delta and Simple-8b are available
7. XOR_FLOAT compression can be applied to float columns
8. columnar + dictionary + delta/FOR can be used in batch
9. zero packing can be applied to default-heavy payloads
10. zstd/FSE can be layered after data-aware encoding
11. zero-copy layout can nearly eliminate extra decode/encode work
12. previous-message patch can further reduce mostly-unchanged message families
13. template batch can bring forward columnar gains even for small consecutive messages
14. trained dictionary reference can strongly compress small, highly correlated message families

On the other hand, for one-shot tiny scalar/short-string/tiny-array payloads, size may be close to MessagePack compact paths or occasionally worse. Therefore, this specification explicitly targets repeated-data amortization and stateful reuse rather than single-shot worst-case wins.

---

## 20. Summary

This specification integrates all of the following into one format family.

- MessagePack-like dynamic usability
- compact row encoding via shared schema
- learning of repeated shape/key/string patterns
- typed vectors
- columnar batch
- per-column adaptive codecs
- float optimization including XOR_FLOAT
- optional zero-copy layout
- previous-message patch / base snapshot reuse
- template batch / control-stream separation
- optional zero packing
- optional zstd/FSE/trained-dictionary compression

As a result, it is easy to use at first transmission and can automatically migrate to smaller representations as soon as repetition or batching appears. In environments where sessions can be maintained, stateful patching and dictionary reuse can reduce transfer size far beyond one-shot stateless mode.
