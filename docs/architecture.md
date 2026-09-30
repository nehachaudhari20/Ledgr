# Ledgr Architecture

## 1. System Definition

Ledgr is a network-level execution-assurance layer for UAP-style agentic payments.

It operates downstream of authorization.

Ledgr assumes that identity, delegation, and payment authorization have already been established. Its responsibility begins with the execution itself.

The system ingests cryptographically verifiable participant observations, reconstructs execution state, derives the expected execution state from the mandate and applicable rules, verifies observed execution against expected execution, identifies the first observable divergence, and produces a provenance-linked Execution Assurance Record.

Drunix provides the shared multi-organization transactional state layer for critical execution state and assurance commitments.

---

## 2. Architectural Objective

The architecture is designed to answer three questions:

```text
1. Was the execution authorized?
2. What actually happened?
3. What should have happened?
```

The resulting assurance flow is:

```text
Authorized Context
        |
        v
Attested Observations
        |
        v
Reconstructed Execution
        |
        v
Expected Execution Model
        |
        v
Verification
        |
        v
First Observable Divergence
        |
        v
Shared Execution State
        |
        v
Execution Assurance Record
```

---

## 3. Eight-Layer Architecture

| Layer | Responsibility | Primary Output |
|---|---|---|
| L1 | Authorization & Execution Context | `ExecutionContext` |
| L2 | Evidence & Attestation | Verified Participant Attestations |
| L3 | Execution-State Reconstruction | `ReconstructedExecution` |
| L4 | Expected-State & Policy Model | `ExpectedExecutionModel` |
| L5 | Verification & Divergence | `VerificationResult` + `D*` |
| L6 | Drunix Shared Execution | Shared State + Commitments |
| L7 | Assurance & Evidence Output | `ExecutionAssuranceRecord` |
| L8 | Security, Privacy & Scalability | `SecurityContext` + Controls |

---

# 4. Layer 1 — Authorization & Execution Context

The first layer establishes the context within which an execution is evaluated.

Ledgr does not replace authorization. Instead, it consumes the authorization context required for downstream assurance.

The execution context identifies:

- mandate
- intent
- agent
- execution
- applicable constraints
- execution binding

Conceptually:

```text
MANDATE
   |
   v
INTENT
   |
   v
EXECUTION
   |
   v
OBSERVATIONS
```

Example identifiers:

```text
execution_id = EXE-7821
intent_id    = INT-551
mandate_id   = MAN-102
agent_id     = AGENT-07
```

The execution context becomes the binding context for subsequent evidence and verification.

---

# 5. Layer 2 — Evidence & Attestation

Participant systems generate observations about the execution.

Potential observation sources include:

- Agent / agent gateway
- PSP
- Bank
- Merchant / fulfilment system
- Authorization or UAP layer
- Dispute / investigation systems

The evidence pipeline is:

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

The attestation gateway verifies:

- participant signature
- participant identity
- event schema
- execution binding
- timestamp
- replay conditions

Only validated attestations proceed into the reconstruction pipeline.

---

# 6. Layer 3 — Execution-State Reconstruction

Participant events can arrive asynchronously and may not arrive in semantic execution order.

Therefore:

```text
Arrival Order != Execution Order
```

Ledgr reconstructs the execution using:

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

### Correlation

Events are associated with the correct execution using execution identity and related identifiers.

### Normalization

Participant-specific event formats are converted into the canonical execution event model.

### Ordering

Events are ordered using execution semantics rather than blindly trusting arrival order.

### Transition

The ordered events are applied to the execution state model.

A conceptual state machine may contain:

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

Legitimate branches such as rejection, cancellation, timeout, or partial execution may also exist.

---

# 7. Layer 4 — Expected-State & Policy Model

The expected execution model represents what should happen according to the authorized mandate and applicable rules.

Conceptually:

```text
E = (S, T, C, I)
```

Where:

- `S` = valid states
- `T` = valid transitions
- `C` = constraints
- `I` = invariants

### States

Represent valid execution states.

### Transitions

Represent valid movement between states.

### Constraints

Represent requirements derived from authorization and execution rules.

Examples:

```text
Maximum Amount
Allowed Merchant
Allowed Payment Method
Execution Validity
Required Authorization
```

### Invariants

Represent conditions that should remain true during execution.

The expected model is versioned so that verification can identify which rule or policy version was applied.

---

# 8. Layer 5 — Verification & Divergence

Verification compares the reconstructed observed execution against the expected execution model.

Conceptually:

```text
Observed Execution O
        |
        v
     VERIFY
        ^
        |
Expected Execution E
```

The observed execution can contain:

- reconstructed states
- ordered evidence
- uncertainty
- conflicting observations

The verification engine evaluates:

- state validity
- transition validity
- authorization constraints
- execution invariants
- evidence completeness
- conflicting observations

Possible outcomes include:

```text
PASS
VIOLATION
NOT_OBSERVED / UNKNOWN
CONFLICTING
```

Missing evidence is not automatically treated as a violation.

---

# 9. First Observable Divergence

The first observable divergence `D*` is the earliest evidence-backed execution transition for which the observed execution can no longer be explained by the authorized expected-state model and valid execution constraints.

Conceptually:

```text
Observed Execution
        |
        v
Transition 1  -> Valid
        |
        v
Transition 2  -> Valid
        |
        v
Transition 3  -> Divergence
        |
        v
D*
```

The system should identify the earliest observable divergence rather than simply reporting the final failed state.

For example:

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

If the mandate permits a maximum of ₹15,000, the `BANK_DEBITED` transition becomes the first observable divergence.

---

# 10. Missing Evidence vs Divergence

Ledgr explicitly distinguishes missing evidence from an observed violation.

Example:

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
     X
MERCHANT_CONFIRMED
```

If merchant confirmation has not been observed, the system should represent the condition as:

```text
NOT_OBSERVED / UNKNOWN
```

rather than automatically claiming:

```text
MERCHANT_FAILURE
```

This distinction is important because the absence of an observation does not necessarily prove that the corresponding event did not occur.

---

# 11. Conflicting Observations

Multiple participants may provide authenticated but conflicting observations.

Example:

```text
PSP       -> ACCEPTED ₹14,800
Bank      -> DEBITED ₹14,800
Merchant  -> ORDER ₹16,200
```

The architecture does not silently overwrite one observation with another.

Instead:

```text
Authenticated Observations
          |
          v
      Retain Conflict
          |
          v
   Verification Analysis
          |
          v
   Conflict / Divergence
```

The evidence remains available for investigation and provenance.

---

# 12. Layer 6 — Drunix Shared Execution

Drunix acts as the shared transactional state layer between participating organizations.

The purpose is not to store every raw participant log.

Instead, Drunix stores limited cross-organizational facts and commitments required for execution assurance.

Examples include:

- execution state
- execution permits
- attestation registration
- execution state transitions
- verification references
- evidence commitments
- assurance commitments

Conceptually:

```text
Participant Systems
       |
       | Signed Observations
       v
Attestation Gateway
       |
       v
Ledgr Assurance Services
       |
       +-----------------------+
       |                       |
       v                       v
Private / Off-chain       Drunix Shared State
Evidence
       |                       |
       +-----------+-----------+
                   |
                   v
          Assurance Record
```

---

# 13. Drunix Transaction Boundary

The core transactional operations include:

```text
CreateExecution
ConsumeExecutionPermit
RegisterAttestation
AdvanceExecutionState
RecordVerification
RecordDivergence
CommitAssurance
```

The chaincode boundary is intentionally small and deterministic.

Ledgr's application services perform processing such as:

- evidence validation
- reconstruction
- expected-state derivation
- verification
- assurance record generation

Drunix handles the shared transactional facts and commitments that require multi-organization consistency.

---

# 14. Execution Permit

The execution permit provides a shared transactional control point for an execution.

A simplified flow is:

```text
CreateExecution
      |
      v
Execution Permit
      |
      v
ConsumeExecutionPermit
      |
      v
Execution State
```

This becomes particularly important for duplicate or concurrent execution attempts.

---

# 15. Concurrent Retry & MVCC

Consider two concurrent attempts to consume the same execution permit.

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

One transaction can commit while the other becomes invalid because its read state is stale.

This provides a transactional mechanism for evaluating concurrent duplicate execution attempts.

---

# 16. Layer 7 — Assurance & Evidence Output

After reconstruction and verification, Ledgr produces an Execution Assurance Record.

The EAR can contain:

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

The assurance record provides a structured representation of the execution conclusion and the evidence supporting it.

---

# 17. Assurance Integrity

The canonical assurance record is hashed before its commitment is stored.

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

This provides an integrity reference that can later be used to verify that the assurance record has not been modified.

---

# 18. Layer 8 — Security, Privacy & Scalability

The architecture incorporates security and operational controls across the complete pipeline.

### Cryptographic Controls

- participant signatures
- cryptographic hashes
- integrity commitments
- key lifecycle

### Execution Security

- execution binding
- participant identity
- replay detection
- cross-execution replay protection

### Transaction Security

- authenticated transactions
- endorsement
- deterministic chaincode
- ordering
- validation
- MVCC

### Access Control

- role-based access control
- least privilege
- private data
- selective disclosure
- data minimization

### Scalability

The architecture separates:

```text
Runtime Execution Processing
```

from:

```text
Investigation / Forensic Analysis
```

It also supports asynchronous evidence ingestion and partitioning around execution identity.

---

# 19. Component Architecture

The major components are:

```text
+-------------------------+
| Participant Systems     |
| Agent / PSP / Bank /    |
| Merchant                |
+------------+------------+
             |
             | Signed Events
             v
+-------------------------+
| Attestation Gateway     |
+------------+------------+
             |
             v
+-------------------------+
| Reconstruction Service  |
+------------+------------+
             |
             +----------------------+
             |                      |
             v                      v
+----------------------+   +----------------------+
| Expected-State       |   | Evidence Store       |
| / Policy Engine      |   | Private / Off-chain  |
+----------+-----------+   +----------------------+
           |
           v
+-------------------------+
| Verification Service    |
+------------+------------+
             |
             v
+-------------------------+
| Assurance API           |
+------------+------------+
             |
       +-----+-----+
       |           |
       v           v
   Dashboard     Drunix
                 Shared State
```

---

# 20. Service Responsibilities

## Attestation Gateway

Responsible for:

- receiving participant attestations
- validating signatures
- validating participant identity
- validating schemas
- checking execution binding
- checking replay conditions

## Reconstruction Service

Responsible for:

- event correlation
- normalization
- semantic ordering
- execution-state reconstruction

## Expected-State Service

Responsible for:

- loading mandate constraints
- applying execution rules
- constructing expected states
- applying policy versions
- evaluating invariants

## Verification Service

Responsible for:

- comparing observed and expected execution
- detecting constraint violations
- detecting invalid transitions
- retaining conflicting observations
- identifying first observable divergence

## Assurance API

Responsible for exposing:

- execution state
- evidence
- verification
- divergence
- assurance records

---

# 21. Data Flow

The complete data flow is:

```text
Participant Observation
          |
          v
     Attestation
          |
          v
   Authentication &
   Validation
          |
          v
   Event Correlation
          |
          v
      Normalize
          |
          v
        Order
          |
          v
   Reconstruct State
          |
          +-------------------+
          |                   |
          v                   v
 Expected-State          Evidence
    Model                  Context
          |                   |
          +---------+---------+
                    |
                    v
               Verification
                    |
                    v
        First Observable Divergence
                    |
                    v
        Execution Assurance Record
                    |
                    v
             SHA-256 Hash
                    |
                    v
          Drunix Commitment
```

---

# 22. Shared vs Private Data

The architecture intentionally separates shared state from detailed evidence.

### Shared / Drunix

Suitable for:

- execution identifiers
- execution state
- permits
- attestation registration
- verification references
- divergence references
- evidence commitments
- assurance commitments

### Private / Off-chain

Suitable for:

- detailed participant records
- sensitive business data
- raw evidence
- data requiring restricted access
- information not required by every participant

The shared layer should contain only the information required for cross-organizational execution assurance.

---

# 23. API-Level Architecture

The conceptual API is execution-centric.

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

The API separates:

```text
Ingestion
   |
   v
Reconstruction
   |
   v
Verification
   |
   v
Assurance
```

---

# 24. Attestation Processing Sequence

```text
1. Participant creates canonical event
2. Event is hashed
3. Participant signs the hash
4. Attestation is submitted
5. Gateway verifies signature
6. Gateway verifies identity
7. Gateway validates schema
8. Gateway verifies execution binding
9. Gateway checks replay conditions
10. RegisterAttestation transaction is submitted
11. Transaction is endorsed
12. Transaction is ordered
13. Transaction is validated
14. MVCC rules are applied
15. State is committed
```

---

# 25. Verification Processing Sequence

```text
1. Load execution context
2. Load authorized evidence
3. Fetch relevant participant attestations
4. Reconstruct observed execution
5. Construct expected execution model
6. Evaluate constraints
7. Evaluate transitions
8. Evaluate invariants
9. Identify conflicts
10. Determine first observable divergence
11. Generate verification result
12. Generate Execution Assurance Record
13. Hash the canonical record
14. Commit assurance reference to Drunix
```

---

# 26. Retry Semantics

Retries should preserve logical operation identity.

A repeated request should use the same:

```text
operation_id
idempotency_key
```

when representing the same logical operation.

The objective is:

```text
Retry
  |
  v
Same Logical Operation
  |
  v
No Duplicate Logical Execution
```

This is particularly important for payment execution and execution-permit consumption.

---

# 27. Technology Boundaries

The design establishes the following technology boundaries:

| Component | Preferred Boundary |
|---|---|
| Drunix / Chaincode | Go preferred |
| Assurance Services | Python / FastAPI or equivalent |
| Shared State | Drunix |
| Detailed Evidence | Private / Off-chain |
| Dashboard | React / TypeScript or equivalent |
| Cryptography | Standard public-key cryptography |

The exact implementation stack remains a design choice during prototyping.

---

# 28. Design Principles

### 1. Deterministic Core Logic

Core reconstruction, expected-state evaluation, and verification should produce reproducible results from the same inputs.

### 2. Provenance by Default

Important conclusions should retain references to the evidence that supports them.

### 3. Missing Evidence Is Not Automatically Failure

`UNKNOWN` and `NOT_OBSERVED` must remain distinct from `VIOLATION`.

### 4. Conflicting Evidence Is Retained

Authenticated conflicts should not be silently overwritten.

### 5. Execution Order Is Semantic

Event arrival order should not automatically define execution order.

### 6. Drunix Is Shared State

Drunix should store critical shared facts and commitments rather than every raw participant log.

### 7. Sensitive Data Remains Controlled

Detailed business and sensitive data should remain within appropriate private or off-chain boundaries.

---

# 29. Architecture Boundary

Ledgr's responsibility can be summarized as:

```text
Authorization Established
          |
          v
+-------------------------------+
|            LEDGR              |
|                               |
| Observe                       |
|    ↓                          |
| Normalize                     |
|    ↓                          |
| Reconstruct                   |
|    ↓                          |
| Expected State                |
|    ↓                          |
| Verify                        |
|    ↓                          |
| Divergence                    |
|    ↓                          |
| Assurance                     |
+-------------------------------+
          |
          v
Shared Assurance Commitment
          |
          v
        Drunix
```

Ledgr does not replace authorization, participant systems, or the underlying payment infrastructure.

It provides the execution-assurance layer that connects participant observations, reconstructed execution, expected execution, verification, divergence localization, evidence, provenance, and shared assurance commitments.
