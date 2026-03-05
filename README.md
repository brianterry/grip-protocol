# GRIP Protocol

**Grounded Resource Intelligence Protocol** — a protocol specification for LLM-based cloud infrastructure security analysis.

## What is GRIP?

GRIP specifies how an AI agent should evaluate cloud infrastructure security: form a hypothesis, ground it in live schema state, verify it deterministically, then act. No finding reaches the surface without passing through a verification gate backed by real rule evaluation — not LLM inference.

## Why GRIP?

LLMs plateau at 60–77% precision on cloud security analysis tasks because they generate plausible answers rather than verifying them — a failure mode termed *phantom concordance*. GRIP resolves this with a protocol-level constraint: the verification gate blocks ungrounded findings before they surface. The same resource evaluated 1000 times returns the same result every time.

## Specification

The formal protocol specification:

[spec/grip-v0.1.md](spec/grip-v0.1.md)

JSON schemas for protocol data types:

[spec/schemas/](spec/schemas/)

## Reference Implementation

[github.com/brianterry/grip](https://github.com/brianterry/grip) — Python, AWS Cloud Control API, CFN Guard via guardpy, AWS Guard Rules Registry (185 rules, 67 resource types, 50 compliance frameworks).

## Status

v0.1 — pre-stable. Interfaces may change between minor versions.

## Author

Brian Terry — University of North Dakota / Amazon Web Services

## License

Apache-2.0
