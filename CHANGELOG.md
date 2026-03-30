# GRIP Protocol Changelog

## v0.3 (2026-03-30)

Three-tier verification, schema validation, and provider-agnostic framing.

### Protocol changes

- **Three-tier verification strategy.** The verification gate now defines three ordered tiers: Tier 0 (schema validation), Tier 1 (policy rules), Tier 2 (heuristic fallback). All three tiers are deterministic.
- **Schema validation (Tier 0).** New verification layer that validates proposed properties against the live schema: unknown properties with fuzzy-match suggestions, read-only property violations, missing required properties, and type mismatches. Provider-agnostic — works with any schema that exposes property definitions.
- **Template authoring construct recognition.** Schema validation recognizes IaC authoring constructs (e.g., CloudFormation `Ref`, `Fn::GetAtt`, `Fn::Sub`; Terraform `var.name`) as type-compatible with the schema's expected type.
- **Grounding failure behavior.** If the resource type does not exist in the schema registry, grounding fails and the loop stops. This is now explicitly specified as terminal — no verification is attempted against an ungrounded hypothesis.
- **Iterative CREATE loop.** Specified the revision loop for CREATE operations: when VERIFY returns VIOLATED, violation evidence is fed back to the hypothesis source for targeted correction, up to a configurable maximum iteration count.
- **Composite rule correctness.** New conformance requirement: implementations MUST suppress sub-rule failures when a composite (OR-branch) parent rule has passed. Prevents false violations that break iterative convergence.
- **Provider-agnostic framing.** Spec language updated throughout to describe GRIP as a provider-neutral protocol. Example bindings for Terraform and Azure added alongside the AWS reference.

### Schema changes

- `grip.schema` response: added `requiredProperties` (array) and `readOnlyProperties` (array) as required fields
- `grip.verify` request: added `operation`, `proposedProperties`, `resourceSchema`, `requiredProperties`, `readOnlyProperties` fields
- `grip.verify` response: added `schemaViolations` and `policyViolations` count fields
- `verification-evidence`: added `grip.schema` as a valid source; added schema-specific rule IDs (`SCHEMA_UNKNOWN_PROPERTY`, `SCHEMA_READ_ONLY_PROPERTY`, `SCHEMA_REQUIRED_PROPERTY`, `SCHEMA_TYPE_MISMATCH`)
- `session`: updated default `gripVersion` from `0.1` to `0.3`

### Conformance changes

- GRIP-Core now requires Tier 0 schema validation and grounding failure detection
- New Section 4.3: Composite Rule Correctness requirement

### References

- Added Trivy, OPA, and Sentinel to references
- Updated provider interface table with Terraform and Azure example bindings

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
