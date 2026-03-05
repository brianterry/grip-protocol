# GRIP Protocol Changelog

## v0.1 (2026-03-05)

Initial specification release.

- Defined four-step GRIP loop: HYPOTHESIZE → GROUND → VERIFY → ACT
- Specified four primitives: grip.schema, grip.security, grip.verify, grip.graph
- Defined provider interface contract (cloud-provider-neutral)
- Defined three conformance levels: GRIP-Core, GRIP-Security, GRIP-Full
- Established determinism requirement for grip.verify
- grip.graph: interface defined, reference implementation in development
- Defined JSON schemas for session, hypothesis, verification evidence, and finding data types

Reference implementation: https://github.com/brianterry/grip (v0.3, GRIP-Security conformant)
