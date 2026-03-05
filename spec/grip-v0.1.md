# GRIP Protocol Specification v0.1

**Grounded Resource Intelligence Protocol**

| Field | Value |
|---|---|
| Status | Draft — Pre-stable |
| Version | 0.1 |
| Date | 2026-03-05 |
| Author | Brian Terry — University of North Dakota / Amazon Web Services |
| Reference Implementation | https://github.com/brianterry/grip |

---

## Abstract

GRIP (Grounded Resource Intelligence Protocol) specifies an interaction protocol for LLM-based agents operating on cloud infrastructure resources. The protocol enforces a four-step evaluation loop — HYPOTHESIZE, GROUND, VERIFY, ACT — in which no security finding reaches the ACT step without passing through a deterministic verification gate backed by live schema state.

GRIP addresses a documented precision bottleneck in LLM-based infrastructure security analysis: non-reasoning models plateau at 60–77% precision when evaluating security misconfigurations, a failure mode this specification terms *phantom concordance*. The protocol resolves this by supplying externally the grounding that reasoning-capable models generate internally through chain-of-thought traces.

---

## 1. Introduction

### 1.1 Background and Motivation

Foundation models applied to cloud infrastructure security analysis exhibit a persistent precision ceiling. In adversarial benchmark evaluation across 90 scenarios and eight foundation models (Terry et al., "Benchmarking Foundation Models on Infrastructure-as-Code Generation Across Adversarial Cloud Security Scenarios," IEEE CARS 2025), non-reasoning models — including Claude 3.5 Sonnet, GPT-4o, and Amazon Nova Pro — plateau at 60–77% precision on misconfiguration detection regardless of model size, prompt engineering, or few-shot examples. The models generate plausible-sounding evaluations that happen to be wrong at a rate that makes them unreliable for production security analysis.

The root cause is a failure mode this specification terms *phantom concordance*: the model confidently asserts that a resource property satisfies or violates a security constraint when no ground-truth verification has occurred. The model is pattern-matching against its training distribution rather than evaluating against the actual resource state. This is a structural limitation of autoregressive generation, not a capability gap that will close with scale. A model that has never seen the live state of a specific S3 bucket cannot determine whether that bucket's `PublicAccessBlockConfiguration.BlockPublicAcls` is `true` or `false` — it can only generate a plausible answer.

One architectural escape exists within the model itself: reasoning-capable models. DeepSeek-R1 achieves 89% precision on the same benchmark by generating explicit chain-of-thought traces that function as self-verification steps. The model reasons through each property check rather than pattern-matching to a conclusion. However, this solution is architecture-dependent — it requires a model trained with reinforcement learning on reasoning traces and adds significant latency and cost per evaluation.

GRIP provides the architectural alternative: supply the grounding externally, at the protocol level, so that any model — reasoning or not — achieves verified evaluation. The verification gate is a protocol requirement, not a model capability. A finding reaches the surface only if a deterministic rule evaluation confirms it against the live resource state. The precision bottleneck is resolved by construction, not by model improvement.

### 1.2 Design Goals

GRIP is built on four design principles.

**Grounding over generation.** Live schema state constrains evaluation before any finding is surfaced. The protocol fetches the resource's authoritative schema definition and current property values from the cloud provider. No property may be evaluated that does not exist in the schema. No value may be asserted that contradicts the live state. The model's role is hypothesis formation; the protocol's role is ground-truth validation.

**Verification before commitment.** The ACT step is gated by deterministic rule evaluation, not by LLM confidence. A finding is surfaced only when the verification gate returns VIOLATED with supporting evidence. Findings that cannot be verified return UNVERIFIED and are reported as such — they are not silently dropped, and they are not silently passed. The verification gate is mandatory; implementations that skip it are not conformant.

**Security as context, not lookup.** Security rules are loaded as session context via the grip.security primitive, available to all four loop steps from session initialization. The model reasons with security constraints already in context rather than discovering them during interaction. This eliminates a class of failures where a model fails to check a relevant constraint because it was never retrieved.

**Provider agnosticism.** The protocol primitives are cloud-provider-neutral. Each primitive defines an interface contract that provider-specific implementations satisfy. The AWS reference implementation uses CloudFormation Registry for schema, Cloud Control API for live state, and CFN Guard for rule evaluation. A compliant Azure implementation would use Azure Resource Manager schema, ARM API live state, and an equivalent rule engine. The protocol does not assume any provider-specific API shape.

### 1.3 Relationship to Existing Protocols

**Model Context Protocol (MCP).** MCP is a general-purpose protocol for connecting LLMs to external tools and data sources. GRIP is domain-specific: it specifies a session lifecycle, a mandatory loop structure, and a verification gate — concepts MCP does not define. GRIP may coexist with MCP in a broader agent system; MCP handles general tool access while GRIP handles cloud infrastructure security operations. GRIP does not extend MCP; the two protocols have different scopes and different architectural commitments.

**OpenAPI / tool-use specifications.** Tool-use specifications define individual function call interfaces. GRIP specifies session-level semantics: a session is initialized with region, account, and compliance framework context; a loop is executed with mandatory sequencing; results carry a full evidence chain. These are session lifecycle concerns, not individual call concerns.

**AWS Config Rules / CloudFormation Guard.** These are rule evaluation engines — they evaluate rules against resource configurations and return pass/fail results. GRIP uses them as backends for its verification primitive. CFN Guard is the rule engine; the AWS Guard Rules Registry is the rule corpus; GRIP is the protocol that sequences hypothesis formation, schema grounding, rule evaluation, and finding surfacing into a verified pipeline. Guard rules are deterministic and repeatable; GRIP makes their use mandatory rather than optional.

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
│  Form a          Fetch      Run        Surface               │
│  security        live       det.       verified              │
│  hypothesis      schema +   rules      findings              │
│                  state      only       only                  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Verification Gate                        │    │
│  │  Finding passes ACT only if VERIFY returns VERIFIED   │    │
│  │  or VIOLATED with evidence                            │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**HYPOTHESIZE.** The agent forms a falsifiable security hypothesis about a specific resource. A hypothesis names the resource type (e.g., `AWS::S3::Bucket`), the operation type (`DETECT`, `CREATE`, or `REMEDIATE`), the resource identifier, and a human-readable description of the claim being tested. Hypotheses are not findings — they are claims pending verification. The hypothesis is a structured object, not free text.

**GROUND.** The protocol fetches the resource's live schema and current state from the cloud provider via the grip.schema primitive. The schema defines which properties exist for the resource type, their types, required/optional/read-only designations, and relationship edges to other resource types. The live state provides the actual current property values for the specific resource instance. Grounding is blocking — the loop does not proceed to VERIFY until grounding completes. A hypothesis that references a property not present in the schema is rejected at this step.

**VERIFY.** The grounded hypothesis is evaluated against the active security rule set using the grip.verify primitive. The reference implementation uses CFN Guard rules from the AWS Guard Rules Registry — deterministic DSL evaluation that returns PASS or FAIL per rule without probabilistic inference. A finding is emitted when the verification gate identifies a rule violation against the live state. Findings that cannot be evaluated due to missing rules return UNVERIFIED. The verification gate MUST be deterministic: the same input MUST produce the same output on every invocation.

**ACT.** Verified findings are surfaced with full evidence: the violated rule identifier, the property path, the current value, the expected constraint, severity, remediation recommendation, and any related rule identifiers (for deduplicated findings). For CREATE operations, the ACT step blocks execution if the verification gate returns VIOLATED — a security-violating configuration is not created. For DETECT operations, the ACT step surfaces all verified findings. For REMEDIATE operations, the ACT step produces a corrected configuration that has itself been re-verified.

### 2.2 Session Lifecycle

A GRIP session proceeds through four phases.

**Initialization.** The client specifies the target cloud provider, region, account identifier, and active compliance frameworks. The runtime confirms API access and loads the security rule set. If compliance frameworks are specified, rules are filtered to only those referenced by the specified frameworks. If no frameworks are specified, all available rules for each resource type are loaded.

**Registry loading.** The runtime indexes available security rules by resource type. For the AWS reference implementation, this means scanning the AWS Guard Rules Registry, parsing resource type annotations from each rule file, and building a lookup table. Framework mappings are loaded from the registry's compiled rule set JSON files.

**Loop execution.** One or more resources are evaluated within a session. Each resource evaluation executes the full HYPOTHESIZE → GROUND → VERIFY → ACT loop. Session context (provider credentials, compliance frameworks, loaded rules) is shared across all loop executions within a session.

**Termination.** The session releases any cached state. No persistent side effects are created by a DETECT session; CREATE and REMEDIATE sessions may have created or modified resources as authorized by the user.

---

## 3. Protocol Primitives

GRIP defines four primitives. Every conformant implementation MUST provide grip.schema and grip.verify. Conformance levels (Section 4) define which additional primitives are required.

### 3.1 grip.schema

Fetches the resource type schema and live state for a specific resource instance.

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
    "required": [ ],
    "readOnlyProperties": [ ],
    "primaryIdentifier": ["/properties/BucketName"]
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
  "fetchedAt": "2026-03-05T00:00:00Z"
}
```

**Error conditions.**
- `SchemaNotAvailable`: The resource type is not recognized by the provider.
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

Evaluates a hypothesis against live state using loaded security rules. This is the verification gate.

**Purpose.** Provide the mandatory corroboration step that separates GRIP from unverified LLM evaluation. Every finding must pass through this gate.

**Request.**

```json
{
  "resourceType": "AWS::S3::Bucket",
  "resourceId": "my-bucket",
  "liveState": {
    "PublicAccessBlockConfiguration": {
      "BlockPublicAcls": false
    }
  },
  "rules": "... loaded rule content ...",
  "hypothesis": "PublicAccessBlockConfiguration.BlockPublicAcls should be true"
}
```

**Response.**

```json
{
  "verdict": "VIOLATED",
  "evidence": [
    {
      "ruleId": "S3_BUCKET_LEVEL_PUBLIC_ACCESS_PROHIBITED",
      "verdict": "VIOLATED",
      "propertyPath": "PublicAccessBlockConfiguration.BlockPublicAcls",
      "actualValue": false,
      "expectedConstraint": "BlockPublicAcls == true",
      "severity": "HIGH",
      "remediation": "Set BlockPublicAcls to true in PublicAccessBlockConfiguration.",
      "source": "Guard/Registry",
      "relatedRuleIds": [
        "S3_BUCKET_PUBLIC_WRITE_PROHIBITED",
        "S3_BUCKET_PUBLIC_READ_PROHIBITED"
      ]
    }
  ],
  "rulesEvaluated": 8,
  "violationsFound": 1
}
```

**Verdict values.**
- `VERIFIED`: All evaluated rules pass. The resource is compliant with the active rule set.
- `VIOLATED`: One or more rules failed. Evidence is provided for each violation.
- `UNVERIFIED`: No rules are available for this resource type. The hypothesis can be neither confirmed nor denied.

**Error conditions.**
- `EvaluationError`: The rule engine encountered an error during evaluation (e.g., malformed rule syntax).

**Conformance requirement.** A compliant grip.verify implementation MUST be deterministic. The same input — same live state, same rules — MUST produce the same output on every invocation. Probabilistic backends (LLM-based evaluation, embedding similarity search, stochastic sampling) do not satisfy this requirement and MUST NOT be used as the primary verification mechanism. An implementation MAY use LLM evaluation as a supplementary heuristic for resource types without deterministic rules, but such results MUST be reported as `UNVERIFIED`, not as `VERIFIED` or `VIOLATED`.

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
    },
    {
      "relationship": "IamInstanceProfile",
      "resourceType": "AWS::IAM::InstanceProfile",
      "resourceId": "my-instance-profile",
      "liveState": { }
    }
  ]
}
```

**Provider interface contract.** A compliant implementation MUST follow schema-defined `$ref` edges to discover first-degree relationships. Traversal depth beyond first-degree is OPTIONAL. The implementation MUST fetch live state for each traversed resource, enabling cross-resource rule evaluation.

---

## 4. Conformance

### 4.1 Conformance Levels

GRIP defines three conformance levels. Each level is a strict superset of the previous level.

**GRIP-Core.** The implementation provides grip.schema and grip.verify. The verification gate is deterministic. The ACT step is gated by the verification result. This is the minimum viable GRIP implementation.

**GRIP-Security.** GRIP-Core plus grip.security with at least one compliance framework supported. Rules can be loaded by resource type and filtered by framework. This enables compliance-scoped evaluation.

**GRIP-Full.** GRIP-Security plus grip.graph relationship traversal. Cross-resource hypotheses can be formed and verified. This enables vulnerability path detection.

The reference implementation at https://github.com/brianterry/grip is GRIP-Security conformant as of v0.3.

### 4.2 Determinism Requirement

Any implementation claiming GRIP conformance at any level MUST use a deterministic verification backend for grip.verify. The same live state evaluated against the same rule set MUST produce the same verdict and evidence on every invocation, without exception.

Probabilistic backends — including LLM-based evaluation, embedding similarity scoring, and stochastic sampling — do not satisfy this requirement. An implementation that uses a probabilistic backend for grip.verify is not GRIP-conformant, regardless of how accurate the backend may be in practice. The precision bottleneck that GRIP addresses is fundamentally a determinism problem, not an accuracy problem. A 95%-accurate probabilistic evaluator still fails 5% of the time in ways that cannot be predicted or reproduced.

---

## 5. Provider Interface

A GRIP provider backend implements the following abstract interface:

```
interface GRIPProvider {
    getSchema(resourceType: string, resourceId: string) → SchemaResult
    getLiveState(resourceType: string, resourceId: string) → StateResult
    listSupportedTypes() → string[]
}
```

The AWS reference implementation satisfies this interface via:
- `CloudFormation.DescribeType` for resource type schema retrieval
- `CloudControl.GetResource` for live resource state retrieval
- CloudFormation Registry enumeration for supported type listing

A compliant Azure implementation would satisfy the same interface via Azure Resource Manager schema definitions and ARM API resource state queries. A compliant GCP implementation would use Cloud Asset Inventory schema and live state APIs. The protocol does not prescribe provider-specific API calls; it prescribes the interface contract that provider adapters must satisfy.

---

## 6. Security Considerations

GRIP operates in read-only mode for DETECT operations. A conformant DETECT implementation MUST NOT write to cloud resources. CREATE and REMEDIATE operations require explicit user authorization before any write operation executes.

Live state fetched by grip.schema may contain sensitive configuration details including encryption key identifiers, network configuration, and access control settings. Implementations MUST handle live state data with access controls appropriate to the sensitivity of the target environment. Live state SHOULD NOT be logged in plaintext in production environments.

Security rules loaded by grip.security are sourced from external repositories (e.g., the AWS Guard Rules Registry). Implementations SHOULD verify rule integrity before loading — for example, by checking repository signatures or pinning to specific commit hashes. An attacker who can modify the rule set can cause GRIP to produce incorrect verdicts.

---

## 7. Versioning

GRIP versions follow semantic versioning. The v0.x series is pre-stable: primitive interfaces, request/response schemas, and conformance requirements may change between minor versions. Breaking changes will be documented in the CHANGELOG.

Version 1.0 will establish a stable interface contract. After v1.0, breaking changes require a major version increment.

---

## 8. References

1. AWS CloudFormation Guard. https://github.com/aws-cloudformation/cloudformation-guard
2. AWS Guard Rules Registry. https://github.com/aws-cloudformation/aws-guard-rules-registry
3. AWS Cloud Control API. https://docs.aws.amazon.com/cloudcontrolapi/
4. guardpy — Python-Rust FFI binding for CloudFormation Guard. https://github.com/brianterry/guardpy
5. GRIP Reference Implementation. https://github.com/brianterry/grip
6. Terry, B. "Benchmarking Foundation Models on Infrastructure-as-Code Generation Across Adversarial Cloud Security Scenarios." IEEE CARS 2025. [Citation pending publication]
