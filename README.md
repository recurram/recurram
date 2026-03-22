# Gowe

This repository contains a documentation-first specification for a compact binary format for structured data.

The format is intended to remain easy to use in schema-less workflows while becoming materially smaller than plain MessagePack when repeated structure, repeated strings, homogeneous arrays, batching, or session reuse is present.

## Goals

- reduce repeated object-key overhead
- support schema-aware compact encoding when schemas are available
- support learned structure in dynamic mode
- support row-wise, columnar, and stateful compression strategies
- keep deterministic wire behavior within a fixed profile

## Non-Goals

- replacing every existing JSON or binary protocol
- defining application semantics
- mandating a single transport handshake for every deployment

## Repository Layout

```text
gowe/
├ README.md
├ LICENSE
├ CONTRIBUTING.md
├ SPEC.md
├ docs/
│  ├ format.md
│  ├ encoding.md
│  └ transport.md
├ versions/
│  └ v1.md
├ examples/
│  ├ README.md
│  ├ basic.json
│  └ schema-example.json
├ diagrams/
│  ├ protocol-flow.md
│  └ encoding-structure.md
├ .editorconfig
├ .gitignore
├ .markdownlint.jsonc
├ .prettierrc.json
└ package.json
```

## Read In This Order

1. `SPEC.md` for the core model and format overview.
2. `docs/format.md` for top-level kinds, object forms, batches, patches, and resets.
3. `docs/encoding.md` for scalar rules, vector codecs, string modes, and compression.
4. `docs/transport.md` for session-scoped state and transport assumptions.
5. `versions/v1.md` for the reference interoperability profile.
6. `examples/` and `diagrams/` for small concrete artifacts.

## Reference Profile

This repository includes a reference profile in `versions/v1.md`.

That profile fixes:

- supported kinds
- required reset operations
- default deterministic heuristics
- recommended metadata and codec rules
- constraints for stateful operation

## License

This repository is distributed under `CC-BY-4.0`. See `LICENSE` for the full license text.
