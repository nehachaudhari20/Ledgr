# LEDGR
## Agentic Payment Execution Assurance

Ledgr is an execution-assurance layer for UAP-style agentic payments.

It is designed for payment executions that cross multiple independently controlled organizations, where authorization may already be established but the network still needs to answer:

- Was the execution authorized?
- What actually happened across participants?
- What should have happened according to the mandate and execution rules?
- Where did the first observable divergence occur?
- Can the resulting conclusion be supported by verifiable evidence and provenance?

Ledgr reconstructs payment execution from cryptographically verifiable participant observations, derives an expected execution model from the authorized mandate and rules, verifies observed execution against that model, identifies the first observable divergence, and produces a provenance-linked **Execution Assurance Record (EAR)**.

Drunix provides the shared multi-organization transactional state layer for critical execution state, execution permits, evidence commitments, and assurance commitments.

---

## 1. Problem

Agentic payment execution can span multiple independently controlled systems:

```text
Agent
  |
  v
Payment / PSP
  |
  v
Bank
  |
  v
Merchant / Fulfilment
```

Each participant may maintain its own databases, logs, events, and internal records.

When an execution is disputed, failed, duplicated, partially completed, or inconsistent across participants, reconstructing what actually happened becomes difficult.

Ledgr addresses this execution-assurance problem by creating a verifiable execution view across participating organizations.

The system separates:

```text
AUTHORIZED
     |
     v
WHAT HAPPENED
     |
     v
WHAT SHOULD HAVE HAPPENED
     |
     v
FIRST OBSERVABLE DIVERGENCE
     |
     v
EXECUTION ASSURANCE RECORD
```

---

## 2. What Ledgr Does

Ledgr operates downstream of authorization.

It assumes that identity, delegation, and payment authorization have already been established.

Its responsibility begins with the execution itself.

Ledgr:

1. Creates an execution context.
2. Receives participant observations.
3. Verifies signed attestations.
4. Reconstructs the execution state.
5. Derives the expected execution state from the mandate and rules.
6. Verifies observed execution against expected execution.
7. Identifies the first observable divergence.
8. Commits critical execution and assurance information to Drunix.
9. Produces a provenance-linked Execution Assurance Record.

---

## 3. Core Questions

### Was it authorized?

The execution context captures the mandate, intent, agent identity, constraints, and execution binding.

### What happened?

Participant attestations are collected and reconstructed into an ordered execution history.

### What should have happened?

An expected execution model is derived from the authorized mandate, constraints, transitions, and invariants.

Verification compares:

```text
Observed Execution
        |
        v
     VERIFY
        ^
        |
Expected Execution
```

Possible outcomes include:

- `PASS`
- `VIOLATION`
- `NOT_OBSERVED / UNKNOWN`
- `CONFLICTING`

---

## 4. Eight-Layer Architecture

| Layer | Responsibility | Main Output |
|---|---|---|
| L1 | Authorization & Execution Context | ExecutionContext |
| L2 | Evidence & Attestation | Verified Participant Attestations |
| L3 | Execution-State Reconstruction | ReconstructedExecution |
| L4 | Expected-State & Policy Model | ExpectedExecutionModel |
| L5 | Verification & Divergence | VerificationResult + D* |
| L6 | Drunix Shared Execution | Shared State + Commitments |
| L7 | Assurance & Evidence Output | Execution Assurance Record |
| L8 | Security, Privacy & Scalability | Security Context + Controls |

### End-to-End Flow

```text
Authorized Context
        |
        v
Attested Observations
        |
        v
Reconstructed State
        |
        v
Expected State
        |
        v
Verification
        |
        v
First Observable Divergence
        |
        v
Drunix Shared Execution State
        |
        v
Execution Assurance Record
```

---

## 5. First Observable Divergence

A central concept in Ledgr is the **first observable divergence**.

It is the earliest evidence-backed execution transition for which the observed execution can no longer be explained by:

- the authorized expected-state model
- valid execution transitions
- applicable constraints
- execution invariants

Missing evidence is **not automatically treated as a divergence**.

For example:

```text
AUTHORIZED
     |
     v
REQUESTED
     |
     v
ACCEPTED
     |
     v
FUNDS_DEBITED
     |
     v
MERCHANT_CONFIRMED
     |
     v
COMPLETED
```

If merchant confirmation has not been observed, Ledgr represents that as missing or unknown evidence rather than automatically claiming that the merchant failed.

If two authenticated participants provide conflicting observations, Ledgr retains the conflict instead of silently selecting one participant as the truth.

---

## 6. Evidence & Attestation

Potential participants include:

- Agent
- Payment Service Provider
- Bank
- Merchant / Fulfilment system
- Network / Assurance Coordinator
- Investigator / Dispute Operations

The attestation flow is:

```text
Participant Event
       |
       v
Canonical Event
       |
       v
Hash
       |
       v
Participant Signature
       |
       v
Attestation
       |
       v
Attestation Gateway
```

The gateway validates:

- signature
- participant identity
- schema
- execution binding
- timestamp
- replay protection

Only validated observations enter the assurance pipeline.

---

## 7. Execution Reconstruction

Participant events may arrive asynchronously and do not necessarily arrive in semantic execution order.

Ledgr therefore separates:

```text
Arrival Order
```

from:

```text
Execution Order
```

Reconstruction performs:

```text
CORRELATE
    |
    v
NORMALIZE
    |
    v
ORDER
    |
    v
TRANSITION
```

A conceptual execution state machine can contain:

```text
AUTHORIZED
     |
     v
REQUESTED
     |
     v
ACCEPTED
     |
     v
PROCESSING
     |
     v
FUNDS_DEBITED
     |
     v
MERCHANT_CONFIRMED
     |
     v
COMPLETED
```

The model can also represent legitimate rejection, cancellation, timeout, partial execution, and other branches.

---

## 8. Expected Execution Model

Ledgr derives an expected execution model from the authorized mandate and applicable rules.

Conceptually:

```text
E = (S, T, C, I)
```

Where:

- `S` = valid states
- `T` = valid transitions
- `C` = constraints
- `I` = invariants

The expected model can represent constraints such as:

- Maximum amount
- Allowed merchant
- Allowed payment method
- Execution validity
- Required authorization
- Required participant sequence

Verification compares:

```text
Observed Execution O
        |
        v
     VERIFY
        ^
        |
Expected Execution E
```

---

## 9. Example: Amount Divergence

Suppose the authorized mandate specifies:

```text
Maximum Amount = ₹15,000
```

Execution:

```text
Agent requests       ₹14,800
PSP accepts          ₹14,800
Bank debits          ₹16,200
```

The reconstructed execution becomes:

```text
AUTHORIZED
     |
     v
REQUESTED ₹14,800
     |
     v
ACCEPTED ₹14,800
     |
     v
BANK_DEBITED ₹16,200
```

The first observable divergence is:

```text
BANK_DEBITED
```

because the observed amount violates the authorized maximum.

The assurance record can contain:

- authorized execution context
- participant evidence
- reconstructed execution
- expected execution constraints
- verification result
- first divergence
- evidence references
- provenance information
- integrity commitment
- Drunix references

---

## 10. Why Drunix

Ledgr uses Drunix as a shared transactional state layer across participating organizations.

Drunix is **not** intended to become a raw-log warehouse.

It stores limited cross-organizational facts and commitments required for execution assurance.

Examples include:

- execution state
- execution permits
- attestation registration
- state transitions
- verification references
- evidence commitments
- assurance commitments

Detailed business records and sensitive data remain in participant systems or private/off-chain storage where appropriate.

```text
Participant Systems
       |
       | Signed Observations
       v
Ledgr Assurance
       |
       +--------------------+
       |                    |
       v                    v
Private / Off-chain     Drunix
Evidence                Shared State
       |                    |
       +---------+----------+
                 |
                 v
       Execution Assurance
```

---

## 11. Drunix Transaction Model

The design includes explicit transactional operations:

```text
CreateExecution
ConsumeExecutionPermit
RegisterAttestation
AdvanceExecutionState
RecordVerification
RecordDivergence
CommitAssurance
```

The shared state is intentionally small and explicit.

A key design objective is deterministic chaincode execution and protection against concurrent state changes.

Duplicate execution attempts can result in competing transactions attempting to consume the same execution permit. Transaction validation and MVCC can expose the stale concurrent update.

---

## 12. Execution Identity

A single execution is bound to a stable execution identity.

Conceptual identifiers include:

```text
execution_id
intent_id
mandate_id
participant_id
attestation_id
event_id
verification_id
divergence_id
record_id
```

Example:

```text
execution_id = EXE-7821
intent_id    = INT-551
mandate_id   = MAN-102
agent_id     = AGENT-07
```

This binding helps prevent evidence or operations from being replayed against another execution.

---

## 13. Provenance

Every important assurance conclusion should be traceable to its supporting evidence.

```text
Participant Observation
        |
        v
Attestation
        |
        v
Reconstructed Event
        |
        v
Verification Result
        |
        v
Divergence
        |
        v
Execution Assurance Record
```

The system should be able to answer:

- Where did this observation come from?
- Who signed it?
- Which execution does it belong to?
- Which verification result used it?
- Why was a divergence identified?
- Which evidence supports the conclusion?

---

## 14. Execution Assurance Record

The **Execution Assurance Record (EAR)** is the primary assurance output.

It can contain:

```text
Execution Context
        +
Reconstructed Execution
        +
Expected Conditions
        +
Verification Result
        +
First Observable Divergence
        +
Evidence References
        +
Provenance
        +
Integrity Commitment
        +
Drunix References
```

The canonical EAR is hashed using SHA-256 before the assurance commitment is recorded.

```text
Canonical EAR
     |
     v
  SHA-256
     |
     v
assurance_hash
     |
     v
CommitAssurance
```

---

## 15. Security Model

### Cryptographic Integrity

- participant signatures
- cryptographic hashes
- integrity commitments
- key lifecycle considerations

### Execution Binding

Evidence is bound to:

- execution identity
- participant identity
- operation
- relevant context

### Replay Protection

The system checks for:

- duplicate attestations
- replayed operations
- cross-execution replay

### Drunix Transaction Security

The design relies on:

- authenticated transactions
- endorsement
- deterministic chaincode
- ordering
- validation
- MVCC

### Access Control

The system applies:

- role-based access control
- least privilege
- private data where required
- selective disclosure
- data minimization

---

## 16. Failure & Threat Scenarios

Ledgr is designed to evaluate:

```text
Amount Divergence
Partial Execution
Duplicate Retry
Conflicting Observations
Late Evidence
Replay
Cross-Execution Replay
Tampered Evidence
Invalid Transition
Concurrent Execution
```

These scenarios can be represented separately for deterministic testing and benchmarking.

---

## 17. Ecosystem Participants

| Participant | Role |
|---|---|
| Network Operator / Assurance Coordinator | Coordinates execution assurance |
| Agent Provider | Produces agent-side execution observations |
| PSP | Provides payment processing observations |
| Bank | Provides account/debit observations |
| Merchant / Fulfilment | Provides fulfilment observations |
| Investigator / Dispute Operations | Uses assurance records for reconstruction |
| Regulator / Compliance | May consume appropriate assurance information |

The detailed commercial model, buyer, pricing, and willingness-to-pay assumptions require validation rather than being treated as established facts.

---

## 18. Technology Boundaries

### Drunix / Chaincode

Go is preferred for Drunix and chaincode components.

### Assurance Services

Python with FastAPI or an equivalent service framework can be used for:

- attestation gateway
- reconstruction
- expected-state evaluation
- verification
- assurance API

### Evidence

Evidence can be divided between:

```text
Drunix Shared State
        +
Private / Off-chain Evidence
```

### Frontend

A React-based dashboard can provide:

- execution timeline
- evidence view
- verification status
- divergence view
- assurance record
- provenance references

### Cryptography

Standard public-key cryptography and cryptographic hashing are used for participant signatures and integrity commitments.

The exact implementation stack remains a prototype design choice.

---

## 19. Conceptual API

```text
POST /executions
POST /executions/{id}/permit
POST /executions/{id}/attestations
POST /executions/{id}/reconstruct
POST /executions/{id}/verify

GET /executions/{id}
GET /executions/{id}/assurance
GET /executions/{id}/evidence
```

Typical attestation flow:

```text
Canonical Event
      |
      v
Sign Hash
      |
      v
POST Attestation
      |
      v
Verify Signature / Identity / Schema
      |
      v
Verify Execution Binding / Replay
      |
      v
RegisterAttestation
      |
      v
Endorsement
      |
      v
Ordering
      |
      v
Validation / MVCC
      |
      v
Commit
```

---

## 20. Duplicate Execution & MVCC

A duplicate execution scenario can involve two concurrent attempts to consume the same execution permit.

```text
                 Execution Permit
                       |
              +--------+--------+
              |                 |
              v                 v
          Attempt A         Attempt B
              |                 |
              v                 v
          Valid Read         Valid Read
              |                 |
              v                 v
           Commit          Stale State
                                |
                                v
                           MVCC Conflict
```

Only the valid committed state transition should consume the permit.

This scenario can be used to evaluate concurrent retry protection and MVCC correctness.

---

## 21. Evaluation

Ledgr should be evaluated using synthetic executions with known ground truth.

The benchmark can generate:

- signed participant events
- authorized execution contexts
- controlled failures
- conflicting observations
- replay attempts
- concurrent operations
- tampered evidence

### Correctness Metrics

- first-divergence accuracy
- constraint-violation detection
- conflict detection
- missing-evidence handling
- replay rejection
- execution-binding validation
- concurrent permit-consumption correctness

### Performance Metrics

- attestation verification latency
- reconstruction latency
- verification latency
- Drunix commit latency
- end-to-end assurance latency
- throughput
- p95 latency
- p99 latency

### Scale

```text
100 executions
1,000 executions
10,000 executions
Larger synthetic datasets
```

---

## 22. Industry Impact Measures

The system can be evaluated against operational outcomes such as:

- dispute reconstruction time
- first-divergence localization accuracy
- evidence completeness
- provenance completeness
- duplicate execution attempts prevented
- verification latency
- Drunix commit latency
- throughput
- p95 latency
- unresolved execution cases

---

## 23. Reproducible Demo

The minimum demonstration should execute one complete end-to-end scenario:

```text
1. Create mandate
2. Create execution
3. Consume execution permit
4. Generate participant observations
5. Submit signed attestations
6. Register attestations
7. Reconstruct execution
8. Derive expected state
9. Verify observed vs expected
10. Identify first divergence
11. Create Execution Assurance Record
12. Commit assurance hash to Drunix
13. Display evidence and execution timeline
```

### Example Mandate

```text
Book Bengaluru → Delhi
Maximum Amount = ₹15,000
```

### Execution

```text
Agent Request = ₹14,800
PSP Acceptance = ₹14,800
Bank Debit = ₹16,200
```

### Expected Result

```text
First Observable Divergence = BANK_DEBITED
Reason = AMOUNT_LIMIT violation
```

---

## 24. Repository Structure

```text
ledgr/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── data-model.md
│   ├── drunix-design.md
│   ├── threat-model.md
│   └── evaluation.md
│
├── drunix/
│   ├── network/
│   └── chaincode/
│
├── execution/
├── assurance/
├── configs/
├── scripts/
│
├── services/
│   ├── attestation-gateway/
│   ├── reconstruction/
│   ├── expected-state/
│   ├── verification/
│   └── assurance-api/
│
├── participants/
│   ├── agent-simulator/
│   ├── psp-simulator/
│   ├── bank-simulator/
│   └── merchant-simulator/
│
├── frontend/
├── benchmark/
│
└── scenarios/
    ├── amount_divergence.json
    ├── partial_execution.json
    ├── duplicate_retry.json
    └── conflicting_observations.json
```

---

## 25. Development Roadmap

### Phase 1 — Drunix Foundation

Set up the official Drunix test network and establish the basic transaction flow.

### Phase 2 — Minimal Chaincode

Implement:

```text
CreateExecution
ConsumeExecutionPermit
RegisterAttestation
AdvanceExecutionState
```

### Phase 3 — Participant Simulators

Create simulated Agent, PSP, Bank, and Merchant participants capable of generating signed attestations.

### Phase 4 — Deterministic Reconstruction

Implement:

```text
Correlation
Normalization
Ordering
State Reconstruction
```

### Phase 5 — Expected-State Engine

Implement:

```text
Authorization Constraints
Execution Rules
State Transitions
Invariants
Policy Versioning
```

### Phase 6 — Verification

Implement:

```text
Observed vs Expected
Constraint Verification
Transition Verification
Conflict Detection
First Observable Divergence
```

### Phase 7 — Assurance

Create the Execution Assurance Record and commit its integrity reference to Drunix.

### Phase 8 — Dashboard

Expose execution timeline, evidence, verification, divergence, provenance, and assurance record.

### Phase 9 — Benchmarking

Evaluate correctness, security, concurrency, throughput, latency, and scalability.

---

## 26. Current Implementation Status

The repository is being developed incrementally.

The documentation defines the target:

- architecture
- data model
- Drunix interaction model
- threat model
- evaluation methodology
- reproducible scenarios

Implementation status should only be marked complete after the corresponding functionality exists and has been validated.

The project distinguishes between:

```text
DESIGNED
IMPLEMENTED
TESTED
BENCHMARKED
```

Planned components are not presented as completed functionality.

---

## 27. Scope Boundaries

Ledgr does **not**:

- replace UAP authorization
- establish identity or delegation
- inspect private agent chain-of-thought
- determine legal liability
- replace participant systems
- act as a raw-log warehouse
- automatically treat missing evidence as proof of failure

Ledgr focuses on:

```text
Execution Assurance
Evidence
Reconstruction
Expected-State Verification
Divergence Localization
Provenance
Shared Execution Commitments
```

---

## 28. Core Design Principle

> **Do not merely record that a payment happened. Establish what was authorized, reconstruct what happened, determine what should have happened, identify where the execution first became inconsistent, and preserve the evidence required to support that conclusion.**

---

## Status

**Project:** Ledgr  
**Domain:** Agentic Payment Execution Assurance  
**Architecture:** Eight-layer execution-assurance architecture  
**Shared State Layer:** Drunix  
**Implementation:** In progress  
**Primary Focus:** Execution reconstruction, verification, divergence localization, provenance, and assurance
