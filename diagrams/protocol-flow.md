# Protocol Flow Diagram

This diagram describes a typical session timeline from bootstrap to stateful optimization and recovery.

```mermaid
sequenceDiagram
    participant C as Client Encoder
    participant S as Server Decoder

    C->>S: Full message (MAP or SCHEMA_OBJECT)
    C->>S: CONTROL REGISTER_KEYS / REGISTER_SHAPE
    C->>S: Compact message (SHAPED_OBJECT or TYPED_VECTOR)
    C->>S: BASE_SNAPSHOT (base_id=7)
    C->>S: STATE_PATCH (base_ref=7, KEEP/REPLACE...)
    C->>S: TEMPLATE_BATCH (template_id=3, count=6)
    C->>S: RESET_STATE
    C->>S: Full stateless resync message
```

Operational intent:

- Start with stateless messages for safe bootstrap.
- Register keys/shapes/strings once repetition appears.
- Use snapshot + patch + template only while state alignment is healthy.
- Reset and resync immediately when unknown references or divergence is detected.
