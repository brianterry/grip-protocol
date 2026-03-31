# GRIP Protocol

**Grounded Resource Intelligence Protocol** — a provider-agnostic protocol specification for grounding LLM infrastructure reasoning in deterministic verification.

## What is GRIP?

GRIP specifies how an AI agent should evaluate cloud infrastructure security: form a hypothesis, ground it in live schema state, verify it deterministically, then act. No finding reaches the surface without passing through a three-tier verification gate backed by real rule evaluation — not LLM inference. The protocol is provider-agnostic: the four-step loop and verification strategy are the same regardless of whether the backend is AWS, Terraform, Azure, or any other IaC platform.

## Why GRIP?

LLMs plateau at 60–77% precision on cloud security analysis tasks because they generate plausible answers rather than verifying them — a failure mode termed *phantom concordance*. GRIP resolves this with a protocol-level constraint: the verification gate blocks ungrounded findings before they surface. The same resource evaluated 1000 times returns the same result every time.

## The Protocol

Four-step loop (HYPOTHESIZE → GROUND → VERIFY → ACT) with three-tier deterministic verification:

- **Tier 0: Schema validation** — structural checks against the live schema (unknown properties, read-only violations, missing required fields, type mismatches)
- **Tier 1: Policy rules** — deterministic rule engine evaluation (Guard, Trivy, OPA, Sentinel)
- **Tier 2: Heuristic fallback** — property comparison against session-level constraints; returns UNVERIFIED if no rules can corroborate

## GRIP vs MCP

MCP (Model Context Protocol) is a general-purpose protocol for connecting LLMs to external tools and data sources. GRIP is domain-specific: it specifies a mandatory loop structure (HYPOTHESIZE → GROUND → VERIFY → ACT), a deterministic verification gate, and a session lifecycle — concepts MCP does not define. MCP says "the model can call tools"; GRIP says "no finding reaches the surface without passing through a deterministic verification gate backed by live schema state."

| | MCP | GRIP |
|---|---|---|
| **Scope** | General-purpose tool access | Cloud infrastructure security |
| **Loop structure** | None — individual tool calls | Mandatory 4-step loop (H → G → V → A) |
| **Tool enforcement** | Optional — model chooses whether to call tools | Mandatory — loop steps cannot be skipped |
| **Verification** | None — trusts model output | Three-tier deterministic gate |
| **Determinism** | Not required | Required — same input, same output, every time |
| **Hallucination guard** | None — model can skip tools and hallucinate | Architectural — ACT is gated by VERIFY result |
| **Session lifecycle** | Connection-level | Region, account, compliance frameworks |
| **Schema grounding** | Not defined | Mandatory — live schema fetched before verification |
| **Finding quality** | Model-dependent | VERIFIED, VIOLATED, or UNVERIFIED — never probabilistic |

**Why MCP alone is not sufficient.** An MCP server can expose cloud infrastructure tools — schema lookups, rule evaluation, resource state queries — but MCP has no way to enforce that those tools are called, called in order, or that results are gated before action. An LLM connected to an MCP server with a `check_compliance` tool can choose to skip that tool and generate an answer from its training data instead. Nothing in MCP prevents this. The result is phantom concordance: the model confidently asserts compliance without ever checking. GRIP eliminates this by making verification a protocol requirement, not a tool the model may optionally invoke. The loop is mandatory, the ordering is fixed, and the ACT step is gated by the VERIFY result. No finding surfaces without deterministic corroboration, regardless of what the model "thinks" the answer is.

The two protocols coexist. MCP handles general tool access; GRIP handles cloud infrastructure security operations where unverified LLM output is not acceptable. An agent system can use MCP for broad tool integration and GRIP for the subset of operations that require grounded, deterministic evaluation.

## Specification

| Version | Status | Date |
|---|---|---|
| [spec/grip-v0.3.md](spec/grip-v0.3.md) | Draft | 2026-03-30 |
| [spec/grip-v0.1.md](spec/grip-v0.1.md) | Superseded | 2026-03-05 |

JSON schemas for protocol data types: [spec/schemas/](spec/schemas/)

JSON schemas for the service wire format (HTTP API): [spec/schemas/service/](spec/schemas/service/)

## Reference Implementation

[github.com/brianterry/grip](https://github.com/brianterry/grip) — Python, AWS CloudFormation + Cloud Control API, CFN Guard via guardpy, AWS Guard Rules Registry (209 rules, 67 resource types, 50 compliance frameworks). GRIP-Security conformant.

A browser-based proof of concept with side-by-side verified vs. unverified comparison is available in the reference implementation's `poc/` directory.

## Implementing GRIP for Other Providers

GRIP is a protocol, not an AWS library. To implement GRIP for another provider, bind the four primitives to provider-specific APIs:

| Primitive | What to implement |
|---|---|
| `grip.schema` | Fetch resource schema + live state from the provider's registry |
| `grip.security` | Load policy rules, filter by compliance framework |
| `grip.verify` | Run three-tier verification (schema + rules + heuristic) |
| `grip.graph` | Traverse resource relationships |

See Section 5 of the [v0.3 spec](spec/grip-v0.3.md) for the provider interface contract and example bindings for Terraform and Azure.

## Status

v0.3 — pre-stable. Interfaces may change between minor versions.

## Author

Brian Terry — University of North Dakota / Amazon Web Services

## License

Apache-2.0
