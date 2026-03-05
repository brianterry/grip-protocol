# Contributing to the GRIP Protocol Specification

## How to Propose Changes

The GRIP specification is maintained as a versioned document in `spec/`. Changes to the protocol follow a structured process.

### Minor Clarifications

For clarifications, typo fixes, or improved examples that do not change protocol semantics:

1. Open a pull request with the proposed change.
2. Describe what is being clarified and why.
3. Changes are reviewed and merged by the spec maintainer.

### Substantive Changes

For changes that affect protocol semantics, primitive interfaces, conformance requirements, or the GRIP loop structure:

1. Open an issue describing the proposed change, its motivation, and its impact on existing implementations.
2. Discussion happens in the issue thread.
3. If consensus is reached, the proposer submits a pull request with the change and a CHANGELOG entry.
4. Substantive changes are deferred to the next minor version (e.g., v0.2).

### New Primitives or Conformance Levels

Proposals for new primitives or conformance levels require:

1. An issue with a detailed design document covering purpose, request/response schemas, error conditions, and provider interface contract.
2. A reference implementation demonstrating the primitive works in practice.
3. Review by the spec maintainer and at least one additional reviewer.

## JSON Schemas

Changes to JSON schemas in `spec/schemas/` must remain backward-compatible within a minor version. New required fields constitute a breaking change and require a minor version increment.

## Style

The specification is written in RFC-style technical prose. Avoid marketing language, subjective claims, and superlatives. Use MUST, SHOULD, MAY, and MUST NOT per RFC 2119 semantics.
