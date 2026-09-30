# Ledgr Data Model

## 1. Purpose

This document defines the canonical data model for Ledgr's execution-assurance pipeline.

```text
Authorization
     |
     v
Execution
     |
     v
Participant Evidence
     |
     v
Reconstruction
     |
     v
Expected Execution
     |
     v
Verification
     |
     v
Divergence
     |
     v
Execution Assurance Record
```

The model is designed around stable execution identity, provenance, deterministic reconstruction, explicit uncertainty, and integrity commitments.

---

## 2. Canonical Objects

| Object | Purpose |
|---|---|
| `ExecutionContext` | Authorized context for an execution |
| `ExecutionEvent` | Canonical representation of an observed event |
| `ParticipantAttestation` | Signed participant observation |
| `ReconstructedExecution` | Reconstructed execution state and history |
| `ExpectedExecutionModel` | Expected states, transitions, constraints and invariants |
| `VerificationResult` | Result of observed-vs-expected verification |
| `SharedExecutionState` | Cross-organizational execution state |
| `ExecutionAssuranceRecord` | Final provenance-linked assurance output |
| `SecurityContext` | Identity, binding, authorization and security metadata |

---

## 3. ExecutionContext

`ExecutionContext` establishes the identity and authorization context of an execution.

Conceptual fields:

```text
execution_id
intent_id
mandate_id
agent_id
participant_id
authorization_context
constraints
policy_version
created_at
status
```

Example:

```text
execution_id = EXE-7821
intent_id    = INT-551
mandate_id   = MAN-102
agent_id     = AGENT-07
```

The `execution_id` is the primary binding identifier for downstream execution evidence.

---

## 4. Execution Identity

Execution identity provides a stable relationship between all records belonging to one logical execution.

```text
Mandate
   |
   v
Intent
   |
   v
Execution
   |
   +----> Events
   |
   +----> Attestations
   |
   +----> Verification
   |
   +----> Divergence
   |
   +----> Assurance Record
```

Identifiers include:

```text
execution_id
intent_id
mandate_id
event_id
attestation_id
verification_id
divergence_id
record_id
participant_id
```

Execution binding helps prevent evidence or operations from being incorrectly associated with another execution.

---

## 5. ExecutionEvent

`ExecutionEvent` is the canonical representation of an execution observation.

Conceptual fields:

```text
event_id
execution_id
participant_id
event_type
event_time
observed_state
amount
currency
related_operation_id
source_reference
payload_hash
schema_version
```

Canonicalization removes participant-specific formatting differences before reconstruction.

---

## 6. ParticipantAttestation

`ParticipantAttestation` represents an authenticated participant observation.

```text
ExecutionEvent
      |
      v
Canonical Representation
      |
      v
Hash
      |
      v
Participant Signature
      |
      v
ParticipantAttestation
```

Conceptual fields:

```text
attestation_id
event_id
execution_id
participant_id
event_hash
signature
signing_key_reference
attestation_timestamp
schema_version
```

The attestation binds the signed observation to the relevant execution and participant.

---

## 7. Attestation Validation

Before an attestation is accepted:

```text
Signature
   |
   +--> Participant Identity
   |
   +--> Schema
   |
   +--> Execution Binding
   |
   +--> Timestamp
   |
   +--> Replay Conditions
```

Invalid attestations are not treated as trusted execution evidence.

---

## 8. ReconstructedExecution

`ReconstructedExecution` represents the execution reconstructed from validated participant observations.

```text
Participant Attestations
          |
          v
      Correlation
          |
          v
      Normalization
          |
          v
        Ordering
          |
          v
       Transition
          |
          v
ReconstructedExecution
```

Conceptual fields:

```text
execution_id
ordered_events
state_history
current_state
participant_observations
uncertainty
conflicts
reconstruction_version
```

Reconstructed states retain references to their underlying observations.

---

## 9. Execution State Vector

The reconstructed execution can be represented as:

```text
X = (
    execution_state,
    participant_states,
    transaction_state,
    amount_state,
    evidence_state,
    uncertainty_state
)
```

The exact representation may vary, but the state must be deterministic from canonical observations and applicable reconstruction rules.

---

## 10. Event-Sourcing Model

Participant observations are treated as events contributing to execution history.

```text
Event 1
   |
   v
State 1
   |
   +--> Event 2
            |
            v
          State 2
            |
            +--> Event 3
                     |
                     v
                   State 3
```

The event history is preserved rather than storing only the final state.

This supports:

- reconstruction
- provenance
- investigation
- divergence localization
- replay analysis

---

## 11. Event Ordering

Arrival order is not necessarily execution order.

The model distinguishes:

```text
event_time
arrival_time
execution_sequence
```

The reconstruction process determines semantic execution order using available evidence and execution rules.

---

## 12. ExpectedExecutionModel

`ExpectedExecutionModel` describes what should happen according to the authorized mandate and applicable rules.

```text
E = (S, T, C, I)
```

Where:

- `S` = valid states
- `T` = valid transitions
- `C` = constraints
- `I` = invariants

Conceptual fields:

```text
model_id
execution_id
policy_version
valid_states
valid_transitions
constraints
invariants
model_version
```

The model is versioned so verification can identify the exact rules applied.

---

## 13. Constraints

Constraints represent requirements that must hold for an execution.

Examples:

```text
Maximum Amount
Allowed Merchant
Allowed Payment Method
Execution Validity
Required Authorization
Required Participant Conditions
```

A constraint evaluation can produce:

```text
TRUE
FALSE
UNKNOWN
```

`UNKNOWN` is used when evidence is insufficient to establish whether a condition was satisfied.

---

## 14. Invariants

Invariants represent conditions that should remain valid throughout execution.

Examples can include:

```text
Execution remains bound to its authorization
Execution identity remains stable
Consumed execution permits are not reused
Required execution conditions remain satisfied
```

Invariant violations become inputs to verification and divergence analysis.

---

## 15. VerificationResult

`VerificationResult` represents comparison of reconstructed execution against the expected execution model.

Conceptual fields:

```text
verification_id
execution_id
model_id
verification_status
constraint_results
transition_results
invariant_results
conflicts
uncertainties
divergence_id
verified_at
verification_version
```

Possible overall states:

```text
PASS
VIOLATION
NOT_OBSERVED / UNKNOWN
CONFLICTING
```

The result retains references to evidence and expected-state rules used during evaluation.

---

## 16. First Observable Divergence

A divergence identifies the earliest evidence-backed transition where observed execution can no longer be explained by the expected execution model.

Conceptual fields:

```text
divergence_id
execution_id
event_id
state_before
observed_transition
expected_transition
constraint_id
reason
evidence_refs
status
```

A divergence may initially be candidate or unresolved when later evidence could change the interpretation.

Missing evidence does not automatically create a divergence.

---

## 17. Conflict Representation

Conflicting authenticated observations remain represented.

Example:

```text
Observation A
PSP -> ACCEPTED ₹14,800

Observation B
Merchant -> ORDER ₹16,200
```

Conceptual conflict data:

```text
conflict_id
execution_id
observation_refs
participants
conflict_type
resolution_status
```

The system does not silently overwrite one authenticated observation with another.

---

## 18. SharedExecutionState

`SharedExecutionState` represents limited cross-organizational state committed to Drunix.

It can include:

```text
execution_id
execution_status
permit_status
execution_state
attestation_references
verification_reference
divergence_reference
evidence_commitments
assurance_reference
state_version
updated_at
```

The shared representation remains intentionally small.

Detailed raw participant evidence does not need to be placed in shared state.

---

## 19. Shared State Keys

Suggested logical keys:

```text
exec:{execution_id}
permit:{intent_id}
att:{attestation_id}
event:{event_id}
verify:{verification_id}
div:{divergence_id}
ear:{record_id}
```

These provide deterministic addressing for execution-related state.

---

## 20. Shared vs Private Data

### Shared State

Suitable for:

- execution identity
- execution state
- execution permits
- attestation references
- verification references
- divergence references
- evidence commitments
- assurance commitments

### Private / Off-chain Evidence

Suitable for:

- detailed participant records
- sensitive business information
- raw evidence
- restricted information
- data not required by every participant

The model separates:

```text
Shared Facts / Commitments
```

from:

```text
Detailed Evidence
```

---

## 21. State Versioning

Shared execution state is versioned.

Conceptually:

```text
state_version
expected_version
```

A transaction expecting version `N` can update the state only if the committed state is still compatible.

```text
Read version N
      |
      v
Prepare update
      |
      v
Validate expected_version
      |
      +----> Match ----> Commit
      |
      +----> Stale ----> MVCC Conflict
```

This is particularly important for concurrent execution-permit consumption.

---

## 22. Execution Permit State

An execution permit can be represented as versioned shared state.

```text
permit_id
execution_id
intent_id
status
consumed_by
consumed_at
state_version
```

Possible permit states include:

```text
AVAILABLE
CONSUMED
```

The implementation may add states required by the execution lifecycle.

---

## 23. ExecutionAssuranceRecord

The `ExecutionAssuranceRecord` is the final structured assurance output.

It can contain:

```text
record_id
execution_id
authorized_context
reconstructed_execution
expected_execution_model_reference
verification_result
first_observable_divergence
evidence_references
provenance
security_context
assurance_hash
drunix_references
record_version
created_at
```

Conceptually:

```text
Execution Context
        +
Reconstructed Execution
        +
Expected Conditions
        +
Verification
        +
Divergence
        +
Evidence
        +
Provenance
        +
Integrity Commitment
        +
Drunix References
```

---

## 24. Assurance Record Integrity

The canonical assurance record is serialized deterministically before hashing.

```text
Canonical EAR
      |
      v
Deterministic Serialization
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

The resulting hash becomes the integrity commitment for the assurance record.

---

## 25. Versioned Assurance

The assurance record should identify versions of the components used to produce it.

Relevant version references include:

```text
schema_version
reconstruction_version
policy_version
verification_version
record_version
```

This allows an assurance result to be interpreted together with the rules and data-model versions under which it was produced.

---

## 26. SecurityContext

`SecurityContext` captures security information required to validate execution evidence.

Conceptual fields:

```text
participant_id
identity_reference
key_reference
signature_algorithm
authorization_reference
execution_binding
replay_reference
access_context
```

SecurityContext supports:

- participant authentication
- signature verification
- execution binding
- replay protection
- access control

---

## 27. Provenance Model

Important derived objects maintain references to the objects from which they were derived.

```text
ExecutionEvent
      |
      v
ParticipantAttestation
      |
      v
ReconstructedExecution
      |
      v
VerificationResult
      |
      v
Divergence
      |
      v
ExecutionAssuranceRecord
```

This creates a traceable chain from participant evidence to final assurance.

---

## 28. Data Lineage

The lineage of a final conclusion can be represented as:

```text
EAR
 |
 +--> Verification Result
 |       |
 |       +--> Expected Execution Model
 |       |
 |       +--> Reconstructed Execution
 |                 |
 |                 +--> Attestation
 |                         |
 |                         +--> Execution Event
 |
 +--> Divergence
 |
 +--> Evidence References
 |
 +--> Drunix References
```

The objective is to make important assurance conclusions auditable and reproducible from their underlying evidence.

---

## 29. Chaincode Read / Write Model

The shared state layer should expose only operations required for cross-organizational consistency.

```text
CreateExecution
    WRITE -> execution state

ConsumeExecutionPermit
    READ  -> permit state
    WRITE -> consumed permit

RegisterAttestation
    WRITE -> attestation reference

AdvanceExecutionState
    READ  -> current execution state
    WRITE -> next state

RecordVerification
    WRITE -> verification reference

RecordDivergence
    WRITE -> divergence reference

CommitAssurance
    WRITE -> assurance commitment
```

The chaincode boundary remains intentionally small and deterministic.

---

## 30. Canonical Identifier Relationships

```text
mandate_id
    |
    v
intent_id
    |
    v
execution_id
    |
    +---- event_id
    |
    +---- attestation_id
    |
    +---- verification_id
    |
    +---- divergence_id
    |
    +---- record_id
```

This allows every assurance artifact to be traced back to the execution context.

---

## 31. Determinism Requirements

The following components should be deterministic for the same canonical inputs:

- event normalization
- event ordering
- execution reconstruction
- expected-state construction
- constraint evaluation
- transition evaluation
- invariant evaluation
- divergence localization
- assurance-record canonicalization

Determinism allows verification to be reproduced from the same evidence and policy versions.

---

## 32. Data Model Principle

The Ledgr data model is based on five principles:

1. **Stable execution identity**
2. **Signed and traceable evidence**
3. **Deterministic reconstruction**
4. **Explicit uncertainty and conflict**
5. **Provenance-linked assurance**

The final data model connects participant observations to a verifiable execution conclusion without requiring all participant data to be placed into shared state.
