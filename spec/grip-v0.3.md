# GRIP Protocol Specification v0.3

**Grounded Resource Intelligence Protocol**

| Field | Value |
|---|---|
| Status | Draft — Pre-stable |
| Version | 0.3 |
| Date | 2026-03-30 |
| Author | Brian Terry — University of North Dakota / Amazon Web Services |
| Reference Implementation | https://github.com/brianterry/grip |

---

## Abstract

GRIP (Grounded Resource Intelligence Protocol) specifies an interaction protocol for LLM-based agents operating on cloud infrastructure resources. The protocol enforces a four-step evaluation loop — HYPOTHESIZE, GROUND, VERIFY, ACT — in which no security finding reaches the ACT step without passing through a deterministic verification gate backed by live schema state.

GRIP addresses a documented precision bottleneck in LLM-based infrastructure security analysis: non-reasoning models plateau at 60–77% precision when evaluating security misconfigurations, a failure mode this specification terms *phantom concordance*. The protocol resolves this by supplying externally the grounding that reasoning-capable models generate internally through chain-of-thought traces.

The protocol is provider-agnostic. Each primitive defines an interface contract that provider-specific implementations satisfy. The four-step loop and three-tier verification strategy are the same regardless of provider; what changes is the binding layer.

---

## 1. Introduction

### 1.1 Background and Motivation

Foundation models applied to cloud infrastructure security analysis exhibit a persistent precision ceiling. In adversarial benchmark evaluation across 90 scenarios and eight foundation models (Terry et al., "Benchmarking Foundation Models on Infrastructure-as-Code Generation Across Adversarial Cloud Security Scenarios," IEEE CARS 2025), non-reasoning models — including Claude 3.5 Sonnet, GPT-4o, and Amazon Nova Pro — plateau at 60–77% precision on misconfiguration detection regardless of model size, prompt engineering, or few-shot examples. The models generate plausible-sounding evaluations that happen to be wrong at a rate that makes them unreliable for production security analysis.

The root cause is a failure mode this specification terms *phantom concordance*: the model confidently asserts that a resource property satisfies or violates a security constraint when no ground-truth verification has occurred. The model is pattern-matching against its training distribution rather than evaluating against the actual resource state. This is a structural limitation of autoregressive generation, not a capability gap that will close with scale. A model that has never seen the live state of a specific S3 bucket cannot determine whether that bucket's `PublicAccessBlockConfiguration.BlockPublicAcls` is `true` or `false` — it can only generate a plausible answer.

One architectural escape exists within the model itself: reasoning-capable models. DeepSeek-R1 achieves 89% precision on the same benchmark by generating explicit chain-of-thought traces that function as self-verification steps. The model reasons through each property check rather than pattern-matching to a conclusion. However, this solution is architecture-dependent — it requires a model trained with reinforcement learning on reasoning traces and adds significant latency and cost per evaluation.

GRIP provides the architectural alternative: supply the grounding externally, at the protocol level, so that any model — reasoning or not — achieves verified evaluation. The verification gate is a protocol requirement, not a model capability. A finding reaches the surface only if a deterministic rule evaluation confirms it against the live resource state. The precision bottleneck is resolved by construction, not by model improvement.

### 1.2 Design Goals

GRIP is built on four design principles.

**Grounding over generation.** Live schema state constrains evaluation before any finding is surfaced. The protocol fetches the resource's authoritative schema definition and current property values from the provider's schema registry. No property may be evaluated that does not exist in the schema. No value may be asserted that contradicts the live state. The model's role is hypothesis formation; the protocol's role is ground-truth validation.

**Verification before commitment.** The ACT step is gated by deterministic rule evaluation, not by LLM confidence. A finding is surfaced only when the verification gate returns VIOLATED with supporting evidence. Findings that cannot be verified return UNVERIFIED and are reported as such — they are not silently dropped, and they are not silently passed. The verification gate is mandatory; implementations that skip it are not conformant.

**Security as context, not lookup.** Security rules are loaded as session context via the grip.security primitive, available to all four loop steps from session initialization. The model reasons with security constraints already in context rather than discovering them during interaction. This eliminates a class of failures where a model fails to check a relevant constraint because it was never retrieved.

**Provider agnosticism.** The protocol primitives are cloud-provider-neutral. Each primitive defines an interface contract that provider-specific implementations satisfy. The AWS reference implementation uses CloudFormation Registry for schema, Cloud Control API for live state, and CFN Guard for rule evaluation. A compliant Terraform implementation would use provider schemas for schema, Terraform state for live state, and Trivy or OPA for rule evaluation. The protocol does not assume any provider-specific API shape.

### 1.3 Relationship to Existing Protocols

**Model Context Protocol (MCP).** MCP is a general-purpose protocol for connecting LLMs to external tools and data sources. GRIP is domain-specific: it specifies a session lifecycle, a mandatory loop structure, and a verification gate — concepts MCP does not define. GRIP may coexist with MCP in a broader agent system; MCP handles general tool access while GRIP handles cloud infrastructure security operations. GRIP does not extend MCP; the two protocols have different scopes and different architectural commitments.

**OpenAPI / tool-use specifications.** Tool-use specifications define individual function call interfaces. GRIP specifies session-level semantics: a session is initialized with region, account, and compliance framework context; a loop is executed with mandatory sequencing; results carry a full evidence chain. These are session lifecycle concerns, not individual call concerns.

**Policy rule engines (Guard, OPA, Trivy, Sentinel).** These are rule evaluation engines — they evaluate rules against resource configurations and return pass/fail results. GRIP uses them as backends for its verification primitive. The engine and rule corpus are pluggable; GRIP is the protocol that sequences hypothesis formation, schema grounding, rule evaluation, and finding surfacing into a verified pipeline. Policy rules are deterministic and repeatable; GRIP makes their use mandatory rather than optional.

---

## 2. Protocol Overview

### 2.1 The GRIP Loop

Every GRIP interaction follows a four-step loop. Steps MUST be executed in order. Steps MUST NOT be skipped. An implementation that allows the ACT step to execute without a preceding VERIFY step is not conformant.

```
┌──────────────────────────────────────────────────────────────┐
│                        GRIP Session                          │
│                                                              │
│  HYPOTHESIZE  →  GROUND  →  VERIFY  →  ACT                  │
│       ↓             ↓          ↓         ↓                   │
│  Form a          Fetch      Three-     Surface               │
│  structured      live       tier       verified              │
│  intent          schema +   det.       findings              │
│                  state      verify     only                  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Verification Gate                        │    │
│  │  Finding passes ACT only if VERIFY returns VERIFIED   │    │
│  │  or VIOLATED with evidence                            │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**HYPOTHESIZE.** The agent forms a structured intent about a specific resource. A hypothesis names the resource type (e.g., `AWS::S3::Bucket`, `aws_s3_bucket`), the operation type (`DETECT`, `CREATE`, or `REMEDIATE`), and either the proposed properties (for CREATE) or the resource identifier (for DETECT/REMEDIATE). A hypothesis is a structured object, not free text. It may be formed by an LLM, a CLI, or an API call.

**GROUND.** The protocol fetches the resource's authoritative schema from the provider's schema registry. Grounding serves two purposes:

1. **Existence validation.** If the resource type does not exist in the schema registry, grounding fails. A grounding failure is terminal — the loop MUST NOT proceed to VERIFY. The implementation SHOULD report the failure and suggest valid alternatives if possible.
2. **Schema extraction.** When the resource type is valid, GRIP extracts the full property definitions: property names, types, required/optional designations, read-only properties (computed by the provider), and relationship edges to other resource types. For DETECT and REMEDIATE operations, grounding also fetches the current live state of the resource via the provider's resource state API.

Grounding is blocking — the loop does not proceed to VERIFY until grounding completes. A hypothesis that references a resource type not present in the schema registry is rejected at this step.

**VERIFY.** The grounded hypothesis is evaluated against a three-tier deterministic verification gate (Section 2.3). Each tier is deterministic: the same input produces the same output on every invocation. The three tiers are:

- **Tier 0: Schema validation.** Structural checks against the live schema — unknown properties, read-only violations, missing required fields, type mismatches.
- **Tier 1: Policy rules.** Deterministic rule engine evaluation using the loaded rule corpus (Guard, Trivy, OPA, Sentinel, etc.).
- **Tier 2: Heuristic fallback.** Direct property comparison against session-level constraints for resource types without policy rule coverage.

Each finding evaluates to PASS, FAIL, or UNVERIFIED. There is no probabilistic middle ground.

**ACT.** Verified findings are surfaced with full evidence: the violated rule identifier, the property path, the current value, the expected constraint, severity, remediation recommendation, and any related rule identifiers. For CREATE operations, the ACT step blocks execution if the verification gate returns VIOLATED. For DETECT operations, all verified findings are surfaced. For REMEDIATE operations, a corrected configuration is produced that has itself been re-verified.

### 2.2 Session Lifecycle

A GRIP session proceeds through four phases.

**Initialization.** The client specifies the target provider, region, account identifier, and active compliance frameworks. The runtime confirms API access and loads the security rule set. If compliance frameworks are specified, rules are filtered to only those referenced by the specified frameworks. If no frameworks are specified, all available rules for each resource type are loaded.

**Registry loading.** The runtime indexes available security rules by resource type. This involves scanning the provider's rule corpus, parsing resource type annotations, and building a lookup table. Framework mappings are loaded from the corpus metadata.

**Loop execution.** One or more resources are evaluated within a session. Each resource evaluation executes the full HYPOTHESIZE → GROUND → VERIFY → ACT loop. Session context (provider credentials, compliance frameworks, loaded rules) is shared across all loop executions within a session.

**Termination.** The session releases any cached state. No persistent side effects are created by a DETECT session; CREATE and REMEDIATE sessions may have created or modified resources as authorized by the user.

### 2.3 Three-Tier Verification Strategy

The verification gate uses a three-tier evaluation strategy. All three tiers are deterministic. Tiers are evaluated in order; all applicable tiers execute for every hypothesis.

**Tier 0 — Schema validation.** Validates proposed properties against the live schema fetched during grounding. This is a structural check that catches four classes of error:

| Check | Description | Severity |
|---|---|---|
| Unknown properties | Property name not in schema. Fuzzy matching suggests corrections. | HIGH |
| Read-only properties | Property is computed by the provider and cannot be set in CREATE. | HIGH |
| Missing required properties | Property is required by the schema but absent from the proposal. Read-only required properties are excluded (auto-generated). | MEDIUM |
| Type mismatches | JSON type of the proposed value does not match the schema definition (e.g., object where string is expected). | MEDIUM |

Schema validation is provider-agnostic. Any IaC system that exposes a JSON Schema or equivalent property definition supports it. This tier catches structural errors that policy rules cannot detect — a Guard rule for `S3_BUCKET_LOGGING_ENABLED` cannot tell you that you misspelled `LogginConfiguration`.

**Template authoring constructs.** IaC systems use authoring-time constructs that resolve to their target type at deploy time. For example, CloudFormation uses `{"Ref": "..."}` (resolves to string), `{"Fn::GetAtt": [...]}` (resolves to string), and `{"Fn::Sub": "..."}` (resolves to string). Terraform uses `var.name` references and function calls. Schema validation MUST recognize these constructs as type-compatible with the schema's expected type. A property typed as `string` that contains `{"Ref": "my-subnet"}` MUST NOT be flagged as a type mismatch.

Implementations MUST maintain a list of recognized authoring constructs for their provider. For CloudFormation, this includes: `Ref`, `Fn::GetAtt`, `Fn::Sub`, `Fn::Join`, `Fn::Select`, `Fn::If`, `Fn::ImportValue`, `Fn::Base64`, `Fn::Cidr`, `Fn::GetAZs`, `Fn::Split`, `Fn::Transform`, `Fn::FindInMap`, and `Condition`.

**Tier 1 — Policy rules.** A deterministic rule engine evaluates security and compliance policies against the resource properties. The engine and rule corpus are pluggable:

| Provider | Rule Engine | Rule Corpus |
|---|---|---|
| AWS CloudFormation | CloudFormation Guard | AWS Guard Rules Registry |
| Terraform | Trivy | Aqua / CIS Terraform benchmarks |
| Terraform | OPA | Rego policies |
| Terraform | Sentinel | HashiCorp policies |
| Azure ARM / Bicep | OPA | Azure CIS benchmarks |

The rule engine evaluates each rule to PASS or FAIL. Policy rules are the primary verification mechanism for security and compliance analysis. The same input produces the same output every time.

**Composite rule handling.** Policy rule corpora may contain composite rules with OR-branch logic: a resource satisfies a composite rule if any one of its sub-rules passes. When a composite rule passes via one branch, failures in alternative branches MUST NOT be reported as violations. An implementation MUST suppress sub-rule failures when the parent composite rule has passed. Failure to suppress these produces false violations that prevent convergence in iterative CREATE loops.

**Tier 2 — Heuristic fallback.** For resource types without policy rule coverage, GRIP falls back to direct property comparison against session-level security constraints. This tier applies constraint patterns — "must be true", "must not be empty", "must be at least N" — against actual property values. Results from this tier that cannot be corroborated by any rule are returned as UNVERIFIED, ensuring that absence of coverage is never mistaken for compliance.

### 2.4 Iterative CREATE Loop

For CREATE operations, the GRIP loop supports iteration. If VERIFY returns VIOLATED, the implementation MAY feed the violation evidence back to the hypothesis source (e.g., the LLM) and request a revised hypothesis. The revised hypothesis re-enters the loop at HYPOTHESIZE and proceeds through GROUND and VERIFY again.

```
HYPOTHESIZE → GROUND → VERIFY → VIOLATED?
     ↑                              │
     └──── revise with evidence ────┘
```

The implementation MUST enforce a maximum iteration count to prevent infinite loops. A reasonable default is 5 iterations. If the maximum is reached without achieving VERIFIED, the last result is returned with all remaining violations.

Each iteration carries the full conversation history: the previous hypothesis, the violations found, and the specific remediation instructions. This enables the hypothesis source to make targeted corrections rather than regenerating from scratch.

---

## 3. Protocol Primitives

GRIP defines four primitives. Every conformant implementation MUST provide grip.schema and grip.verify. Conformance levels (Section 4) define which additional primitives are required.

### 3.1 grip.schema

Fetches the resource type schema and optionally the live state for a specific resource instance.

**Purpose.** Provide the ground truth that anchors hypothesis evaluation. The schema constrains which properties are valid; the live state provides their actual values.

**Request.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "resourceId": "my-bucket",
  "region": "us-east-1"
}
```

**Response.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "resourceId": "my-bucket",
  "schema": {
    "properties": { },
    "required": ["BucketName"],
    "readOnlyProperties": [
      "/properties/Arn",
      "/properties/DomainName",
      "/properties/DualStackDomainName",
      "/properties/RegionalDomainName",
      "/properties/WebsiteURL"
    ]
  },
  "liveState": {
    "BucketName": "my-bucket",
    "PublicAccessBlockConfiguration": {
      "BlockPublicAcls": false,
      "BlockPublicPolicy": false,
      "IgnorePublicAcls": false,
      "RestrictPublicBuckets": false
    }
  },
  "relationships": [
    {
      "propertyPath": "/properties/ReplicationConfiguration",
      "referencedResourceType": "AWS::S3::Bucket"
    }
  ],
  "propertyCount": 28,
  "requiredProperties": ["BucketName"],
  "readOnlyProperties": ["/properties/Arn", "/properties/DomainName"],
  "fetchedAt": "2026-03-30T00:00:00Z"
}
```

**Required response fields for schema validation.** Implementations MUST include `requiredProperties` (array of property names required for CREATE) and `readOnlyProperties` (array of JSON pointer paths to properties computed by the provider). These fields are consumed by Tier 0 schema validation in grip.verify.

**Error conditions.**
- `SchemaNotAvailable`: The resource type is not recognized by the provider. This is a grounding failure — the loop MUST NOT proceed.
- `ResourceNotFound`: The resource identifier does not correspond to an existing resource.
- `AuthenticationRequired`: The session lacks credentials for the target provider.

**Provider interface contract.** A compliant provider implementation MUST return the complete resource schema and current state for any resource type it supports. It MUST return a `SchemaNotAvailable` error for unsupported resource types. It MUST return a `ResourceNotFound` error for valid resource types with non-existent identifiers, rather than returning empty state.

### 3.2 grip.security

Loads security rules applicable to a resource type, optionally filtered by compliance framework.

**Purpose.** Deliver security constraints as session context. Rules are loaded once and applied to all evaluations within the session scope.

**Request.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "frameworks": ["cis-aws-benchmark-level-1"]
}
```

**Response.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "frameworks": ["cis-aws-benchmark-level-1"],
  "ruleCount": 5,
  "rules": "... rule content in provider-native format ..."
}
```

**Error conditions.**
- `FrameworkNotAvailable`: The specified compliance framework is not recognized.
- `NoRulesAvailable`: No rules exist for the specified resource type (optionally filtered by framework).

**Provider interface contract.** A compliant implementation MUST support loading rules by resource type. Framework filtering is OPTIONAL but RECOMMENDED. The rule format is provider-native — GRIP does not prescribe a universal rule language. The AWS reference implementation uses CFN Guard DSL; other providers may use OPA/Rego, Sentinel, or equivalent DSLs.

### 3.3 grip.verify

Evaluates a hypothesis against live state using the three-tier verification strategy. This is the verification gate.

**Purpose.** Provide the mandatory corroboration step that separates GRIP from unverified LLM evaluation. Every finding must pass through this gate.

**Request.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "operation": "CREATE",
  "proposedProperties": {
    "BucketName": "my-bucket",
    "PublicAccessBlockConfiguration": {
      "BlockPublicAcls": true
    }
  },
  "resourceSchema": { },
  "requiredProperties": ["BucketName"],
  "readOnlyProperties": ["/properties/Arn"],
  "rules": "... loaded rule content ...",
  "liveState": null
}
```

For DETECT operations, `liveState` contains the fetched resource state and `proposedProperties` is omitted.

**Response.**

```json
{
  "verdict": "VIOLATED",
  "evidence": [
    {
      "source": "grip.schema",
      "ruleId": "SCHEMA_UNKNOWN_PROPERTY",
      "verdict": "VIOLATED",
      "propertyPath": "/properties/LogginConfiguration",
      "actualValue": {},
      "expectedConstraint": "Remove or rename this property. Did you mean: LoggingConfiguration?",
      "severity": "HIGH"
    },
    {
      "source": "grip.guard/registry",
      "ruleId": "S3_BUCKET_LOGGING_ENABLED",
      "verdict": "VIOLATED",
      "propertyPath": "LoggingConfiguration",
      "actualValue": "NOT SET",
      "expectedConstraint": "Set LoggingConfiguration to enable S3 bucket logging.",
      "severity": "HIGH",
      "remediation": "Set the S3 Bucket property LoggingConfiguration to start logging."
    }
  ],
  "rulesEvaluated": 9,
  "violationsFound": 2,
  "schemaViolations": 1,
  "policyViolations": 1
}
```

**Verdict values.**
- `VERIFIED`: All evaluated rules across all tiers pass. The resource is compliant.
- `VIOLATED`: One or more rules failed. Evidence is provided for each violation.
- `UNVERIFIED`: No rules are available for this resource type. The hypothesis can be neither confirmed nor denied.

**Evidence source values.** Each evidence item carries a `source` field identifying which verification tier produced it:

| Source | Tier | Description |
|---|---|---|
| `grip.schema` | 0 | Schema validation — structural check against live schema |
| `grip.guard/registry` | 1 | Policy rules — provider's rule registry |
| `grip.guard` | 1 | Policy rules — local/custom rules |
| `grip.security` | 2 | Heuristic fallback — session-level constraint comparison |

**Schema validation rule IDs.** Tier 0 emits evidence with the following rule IDs:

| Rule ID | Description |
|---|---|
| `SCHEMA_UNKNOWN_PROPERTY` | Property name not found in schema. Fuzzy matches included in remediation. |
| `SCHEMA_READ_ONLY_PROPERTY` | Property is read-only (computed by provider). Cannot be set in CREATE. |
| `SCHEMA_REQUIRED_PROPERTY` | Required property missing from proposal. |
| `SCHEMA_TYPE_MISMATCH` | Value type does not match schema definition. |

**Error conditions.**
- `EvaluationError`: The rule engine encountered an error during evaluation.
- `GroundingFailed`: The resource type does not exist in the schema registry. The verification gate MUST NOT evaluate rules against an ungrounded hypothesis.

**Conformance requirement.** A compliant grip.verify implementation MUST be deterministic. The same input — same live state, same rules, same schema — MUST produce the same output on every invocation. Probabilistic backends (LLM-based evaluation, embedding similarity search, stochastic sampling) do not satisfy this requirement and MUST NOT be used as the primary verification mechanism. An implementation MAY use LLM evaluation as a supplementary heuristic for resource types without deterministic rules, but such results MUST be reported as `UNVERIFIED`, not as `VERIFIED` or `VIOLATED`.

### 3.4 grip.graph

Traverses resource relationships to detect cross-resource vulnerabilities.

**Purpose.** Cloud infrastructure vulnerabilities are frequently cross-resource: a misconfiguration is insecure not because of a single attribute value but because of a path through the resource relationship graph. An EC2 instance with an overly permissive security group, an IAM role with wildcard permissions, and a public subnet form a lateral movement path that no single-resource check detects.

**Status.** Interface defined in v0.1. Reference implementation in development.

**Request.**

```json
{
  "resourceType": "AWS::EC2::Instance",
  "resourceId": "i-1234567890abcdef0",
  "traversalDepth": 2
}
```

**Response.**

```json
{
  "resourceType": "AWS::EC2::Instance",
  "resourceId": "i-1234567890abcdef0",
  "edges": [
    {
      "relationship": "SecurityGroup",
      "resourceType": "AWS::EC2::SecurityGroup",
      "resourceId": "sg-892adfec",
      "liveState": { }
    },
    {
      "relationship": "Subnet",
      "resourceType": "AWS::EC2::Subnet",
      "resourceId": "subnet-47b4cf2c",
      "liveState": { }
    }
  ]
}
```

**Provider interface contract.** A compliant implementation MUST follow schema-defined relationship edges to discover first-degree relationships. Traversal depth beyond first-degree is OPTIONAL. The implementation MUST fetch live state for each traversed resource, enabling cross-resource rule evaluation.

---

## 4. Conformance

### 4.1 Conformance Levels

GRIP defines three conformance levels. Each level is a strict superset of the previous level.

**GRIP-Core.** The implementation provides grip.schema and grip.verify. The verification gate includes at minimum Tier 0 (schema validation) and is deterministic. The ACT step is gated by the verification result. Grounding failures (resource type does not exist) are detected and reported. This is the minimum viable GRIP implementation.

**GRIP-Security.** GRIP-Core plus grip.security with at least one compliance framework supported. Rules can be loaded by resource type and filtered by framework. Tier 1 (policy rules) is active with a deterministic rule engine. This enables compliance-scoped evaluation.

**GRIP-Full.** GRIP-Security plus grip.graph relationship traversal. Cross-resource hypotheses can be formed and verified. This enables vulnerability path detection.

The reference implementation at https://github.com/brianterry/grip is GRIP-Security conformant as of v0.3.

### 4.2 Determinism Requirement

Any implementation claiming GRIP conformance at any level MUST use a deterministic verification backend for grip.verify. The same live state evaluated against the same rule set MUST produce the same verdict and evidence on every invocation, without exception.

Probabilistic backends — including LLM-based evaluation, embedding similarity scoring, and stochastic sampling — do not satisfy this requirement. An implementation that uses a probabilistic backend for grip.verify is not GRIP-conformant, regardless of how accurate the backend may be in practice. The precision bottleneck that GRIP addresses is fundamentally a determinism problem, not an accuracy problem. A 95%-accurate probabilistic evaluator still fails 5% of the time in ways that cannot be predicted or reproduced.

### 4.3 Composite Rule Correctness

Implementations that use policy rule engines with composite (OR-branch) rules MUST correctly suppress sub-rule failures when the composite parent rule has passed. A composite rule `R` that passes because sub-rule `R_a` passed MUST NOT emit a violation for sub-rule `R_b`'s failure. This is a correctness requirement, not an optimization — reporting these failures as violations produces false positives that cannot be resolved, breaking convergence in iterative CREATE loops.

---

## 5. Provider Interface

A GRIP provider backend implements the following abstract interface:

```
interface GRIPProvider {
    getSchema(resourceType: string, resourceId: string?) → SchemaResult
    getLiveState(resourceType: string, resourceId: string) → StateResult
    listSupportedTypes() → string[]
    getRequiredProperties(resourceType: string) → string[]
    getReadOnlyProperties(resourceType: string) → string[]
}
```

The `getRequiredProperties` and `getReadOnlyProperties` methods support Tier 0 schema validation. They return property names (or JSON pointer paths) identifying which properties are required for resource creation and which are computed by the provider.

The interface is provider-neutral. Example bindings:

| Method | AWS | Terraform | Azure |
|---|---|---|---|
| `getSchema` | `cloudformation.describe_type()` | `terraform providers schema -json` | Azure Resource Manager schema |
| `getLiveState` | `cloudcontrol.get_resource()` | `terraform show -json` | Azure Resource Graph query |
| `listSupportedTypes` | CloudFormation Registry enumeration | Provider schema type listing | ARM resource type catalog |
| `getRequiredProperties` | Schema `required` array | Provider schema `required` array | ARM schema `required` array |
| `getReadOnlyProperties` | Schema `readOnlyProperties` array | Schema `computed` attributes | ARM schema `readOnly` attributes |

A compliant implementation satisfies this interface via provider-native APIs. The protocol does not prescribe provider-specific API calls; it prescribes the interface contract that provider adapters must satisfy.

---

## 6. Service Wire Format

### 6.1 Overview

A GRIP verification service exposes the protocol over HTTP as a stateless JSON API. The service accepts IaC artifacts at the artifact level — the client sends a complete template or configuration, and the service internally decomposes it into resources, runs the full GRIP loop for each resource, and returns a flat list of standardized findings.

This artifact-level interface ensures that the client does not need to understand the IaC format. The service handles parsing, resource extraction, schema grounding, and verification. The client sends content; the client gets findings back.

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐
│ Orchestrator │  HTTP   │  GRIP Service    │  intern │  Verification    │
│ (LLM agent,  │ ──────► │  /v1/sessions    │ ──────► │  Backend         │
│  CLI, SDK)   │         │  /v1/verify      │         │  (Guard, Trivy,  │
│              │ ◄────── │  /v1/health      │ ◄────── │   Checkov, OPA)  │
│              │ findings│                  │ results │                  │
└──────────────┘         └──────────────────┘         └──────────────────┘
```

### 6.2 Endpoints

**`POST /v1/sessions`** — Open a verification session.

Request body: [session-request.json](schemas/service/session-request.json)

```json
{
  "protocolVersion": "0.3",
  "frameworks": ["hipaa-security", "pci-dss"]
}
```

Response body: [session-response.json](schemas/service/session-response.json)

```json
{
  "sessionId": "sess-a1b2c3d4",
  "protocolVersion": "0.3",
  "frameworksAvailable": ["hipaa-security", "pci-dss", "cis-aws-benchmark-level-1"],
  "frameworksActive": ["hipaa-security", "pci-dss"],
  "coverage": {
    "guard": { "resourceTypes": 67, "rules": 185 }
  }
}
```

**`POST /v1/verify`** — Verify an IaC artifact.

Request body: [verify-request.json](schemas/service/verify-request.json)

The request contains an `artifact` object with two fields: `format` (the IaC language) and `content` (the raw template/configuration text).

```json
{
  "sessionId": "sess-a1b2c3d4",
  "artifact": {
    "format": "cloudformation",
    "content": "AWSTemplateFormatVersion: '2010-09-09'\nResources:\n  DataBucket:\n    Type: AWS::S3::Bucket\n    Properties:\n      BucketName: my-data"
  }
}
```

Response body: [verify-response.json](schemas/service/verify-response.json)

```json
{
  "runId": "run-e5f6g7h8",
  "sessionId": "sess-a1b2c3d4",
  "outcome": "VIOLATED",
  "findings": [
    {
      "ruleId": "S3_BUCKET_LOGGING_ENABLED",
      "severity": "HIGH",
      "resource": "DataBucket",
      "title": "S3 Bucket Logging must be enabled",
      "remediation": "Set the S3 Bucket property LoggingConfiguration",
      "property": "LoggingConfiguration",
      "sourceLine": 0,
      "referenceUrl": ""
    }
  ],
  "stats": {
    "totalChecks": 9,
    "totalViolations": 1
  },
  "error": null
}
```

**`GET /v1/health`** — Health check.

Returns service status, protocol version, and supported artifact formats. No request body. Response format is not prescribed; implementations SHOULD include at minimum `status`, `protocolVersion`, and `artifactFormats`.

### 6.3 Artifact Format

The `artifact` object is the unit of input to the verify endpoint. It wraps raw IaC content with a format discriminator so the service can select the correct parser and verification backend.

| Format | Value | Example Content |
|---|---|---|
| AWS CloudFormation | `cloudformation` | YAML or JSON template |
| Terraform | `terraform` | HCL configuration |
| Kubernetes | `kubernetes` | YAML manifests |
| Bicep | `bicep` | Bicep template |
| Pulumi | `pulumi` | Pulumi program output |

A service MUST reject artifacts with unsupported formats with HTTP 400 and an explanatory error message. A service MAY support multiple formats.

### 6.4 Service Finding

The service finding (`ServiceFinding` in the JSON schema) is the universal unit of output. All verification backends normalize their results into this shape. The finding is flat — it includes the `resource` identifier because the service operates at the artifact level (multiple resources).

| Field | Type | Required | Description |
|---|---|---|---|
| `ruleId` | string | **yes** | Scanner-native rule identifier |
| `severity` | enum | **yes** | CRITICAL, HIGH, MEDIUM, LOW, INFO |
| `resource` | string | **yes** | Resource identifier as it appears in the artifact |
| `title` | string | no | Human-readable one-line summary |
| `remediation` | string | no | Actionable guidance on how to fix |
| `property` | string | no | Specific attribute or property path affected |
| `sourceLine` | integer | no | Line number in the artifact (0 if unavailable) |
| `referenceUrl` | string | no | URL to rule documentation or advisory |

**Relationship to primitive-level Finding.** The service finding extends the primitive-level finding (Section 3, [finding.json](schemas/finding.json)) with `resource`, `sourceLine`, and `referenceUrl`. The primitive-level finding operates within a single resource evaluation; the service finding operates across all resources in an artifact.

### 6.5 Backend Portability

The service wire format is intentionally backend-agnostic. The same request/response shapes work regardless of the underlying verification engine. This was validated by building two reference service implementations:

| | CloudFormation Service | Terraform Service |
|---|---|---|
| **Verification backend** | CloudFormation Guard + guardpy | Trivy |
| **Artifact format** | `cloudformation` | `terraform` |
| **Rule source** | AWS Guard Rules Registry | Aqua vulnerability database |
| **Response shape** | Identical | Identical |

An orchestrator that can call `POST /v1/verify` with an artifact and parse the `findings` array works with any GRIP service, regardless of the IaC format or verification backend. This enables a future SDK that is implementation-agnostic.

### 6.6 Conformance

A GRIP service implementation is wire-format conformant if:

1. `POST /v1/sessions` accepts a `SessionRequest` and returns a `SessionResponse` with a valid `sessionId`
2. `POST /v1/verify` accepts a `VerifyRequest` containing an `artifact` and returns a `VerifyResponse` with `outcome` and `findings`
3. Every finding in the `findings` array contains at minimum `ruleId`, `severity`, and `resource`
4. The `outcome` field is `VERIFIED` when `findings` is empty, `VIOLATED` when `findings` is non-empty, and `UNVERIFIED` when verification could not be performed
5. The service returns HTTP 400 for unsupported artifact formats and HTTP 404 for unknown session IDs

---

## 7. Security Considerations

GRIP operates in read-only mode for DETECT and artifact verification operations. A conformant DETECT implementation MUST NOT write to cloud resources. CREATE and REMEDIATE operations require explicit user authorization before any write operation executes.

Live state fetched by grip.schema may contain sensitive configuration details including encryption key identifiers, network configuration, and access control settings. Implementations MUST handle live state data with access controls appropriate to the sensitivity of the target environment. Live state SHOULD NOT be logged in plaintext in production environments.

Security rules loaded by grip.security are sourced from external repositories (e.g., the AWS Guard Rules Registry, Aqua Trivy policies). Implementations SHOULD verify rule integrity before loading — for example, by checking repository signatures or pinning to specific commit hashes. An attacker who can modify the rule set can cause GRIP to produce incorrect verdicts.

---

## 8. Versioning

GRIP versions follow semantic versioning. The v0.x series is pre-stable: primitive interfaces, request/response schemas, and conformance requirements may change between minor versions. Breaking changes will be documented in the CHANGELOG.

Version 1.0 will establish a stable interface contract. After v1.0, breaking changes require a major version increment.

### Changes from v0.3-draft to v0.3

- Added Section 6: Service Wire Format — defines the HTTP API contract (`/v1/sessions`, `/v1/verify`, `/v1/health`)
- Added `Artifact` abstraction: `{format, content}` wrapper for submitting IaC content regardless of language
- Added `ServiceFinding` schema: universal flat finding with `ruleId`, `severity`, `resource`, `title`, `remediation`, `property`, `sourceLine`, `referenceUrl`
- Added service-level JSON schemas: `service/session-request.json`, `service/session-response.json`, `service/verify-request.json`, `service/verify-response.json`
- Added wire-format conformance checklist (Section 6.6)
- Validated backend portability across CloudFormation (Guard) and Terraform (Trivy) reference implementations

### Changes from v0.1 to v0.3

- Added three-tier verification strategy (schema validation, policy rules, heuristic fallback)
- Added schema validation as Tier 0 with four structural checks
- Added template authoring construct recognition (intrinsic functions)
- Added grounding failure behavior (terminal when resource type does not exist)
- Added iterative CREATE loop specification
- Added composite rule correctness requirement (OR-branch suppression)
- Added `requiredProperties` and `readOnlyProperties` to grip.schema response
- Added `schemaViolations` and `policyViolations` counts to grip.verify response
- Added schema-specific evidence source (`grip.schema`) and rule IDs
- Updated GRIP-Core conformance to require Tier 0 schema validation
- Updated provider interface to include property metadata methods

---

## 9. References

1. AWS CloudFormation Guard. https://github.com/aws-cloudformation/cloudformation-guard
2. AWS Guard Rules Registry. https://github.com/aws-cloudformation/aws-guard-rules-registry
3. Trivy — Comprehensive Security Scanner. https://github.com/aquasecurity/trivy
4. Open Policy Agent (OPA). https://www.openpolicyagent.org/
5. HashiCorp Sentinel. https://www.hashicorp.com/sentinel
6. guardpy — Python-Rust FFI binding for CloudFormation Guard. https://github.com/brianterry/guardpy
7. GRIP Reference Implementation. https://github.com/brianterry/grip
8. Terry, B. "Benchmarking Foundation Models on Infrastructure-as-Code Generation Across Adversarial Cloud Security Scenarios." IEEE CARS 2025. [Citation pending publication]
