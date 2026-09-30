# Ledgr Drunix Design

## 1. Purpose

Drunix provides the shared multi-organization transactional state layer for Ledgr.

Ledgr assumes that agent identity, delegation, and payment authorization have already been established. Drunix is used downstream to provide a shared mechanism for critical execution state and commitments across independently controlled participants.

The design does **not** use Drunix as a raw participant-log warehouse.

Instead, Drunix stores limited cross-organizational facts required for execution assurance.

---

## 2. Why a Shared Transactional Layer

Agentic payment execution can cross independently controlled organizations.

A single execution may involve:

```text
Agent
   |
   v
PSP
   |
   v
Bank
   |
   v
Merchant / Fulfilment
```

Each participant may maintain its own internal state.

Ledgr therefore needs a shared mechanism for selected facts such as:

- execution state
- execution permits
- attestation registration
- critical state transitions
- verification references
- evidence commitments
- assurance commitments

Detailed business data and raw evidence remain in participant systems or private/off-chain storage where appropriate.

---

## 3. Role of Drunix

Drunix acts as the shared execution-state and commitment layer.

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
        +----------------------+
        |                      |
        v                      v
Private / Off-chain       Drunix Shared State
Evidence
        |                      |
        +----------+-----------+
                   |
                   v
          Assurance Commitment
```

The Ledgr services perform the higher-level assurance processing.

Drunix provides the shared transactional boundary where selected facts must be consistently represented across organizations.

---

## 4. Prototype Organization Model

The prototype can represent multiple participating organizations such as:

```text
Organization 1 -> Agent / Agent Provider
Organization 2 -> PSP
Organization 3 -> Bank
Organization 4 -> Merchant / Fulfilment
```

The exact organization configuration belongs to the prototype network setup.

The important architectural boundary is that no single participant is assumed to own the complete execution state.

---

## 5. Shared Execution State

The shared state should remain intentionally small.

Conceptual execution state:

```text
execution_id
execution_status
execution_state
permit_status
state_version
created_at
updated_at
```

Additional references can include:

```text
attestation_references
verification_reference
divergence_reference
evidence_commitments
assurance_reference
```

The purpose is to make critical cross-organizational execution facts available through a consistent transactional state.

---

## 6. Shared vs Private Data

### Shared Through Drunix

Suitable shared information includes:

- execution identifiers
- execution state
- execution permits
- attestation registration
- verification references
- divergence references
- evidence commitments
- assurance commitments

### Private / Off-chain

Suitable private information includes:

- detailed participant records
- sensitive business data
- raw participant evidence
- restricted information
- information that does not need to be shared across organizations

The architecture therefore follows:

```text
Drunix
   |
   +--> Shared Facts
   |
   +--> State
   |
   +--> Commitments

Private / Off-chain
   |
   +--> Detailed Evidence
   |
   +--> Sensitive Business Data
```

---

## 7. Logical State Keys

Suggested logical keys are:

```text
exec:{execution_id}
permit:{intent_id}
att:{attestation_id}
event:{event_id}
verify:{verification_id}
div:{divergence_id}
ear:{record_id}
```

These keys provide deterministic addressing for execution-related state.

---

## 8. Core Chaincode Operations

The core transaction model includes:

```text
CreateExecution
ConsumeExecutionPermit
RegisterAttestation
AdvanceExecutionState
RecordVerification
RecordDivergence
CommitAssurance
```

The chaincode boundary should remain small and deterministic.

---

## 9. CreateExecution

`CreateExecution` establishes the shared execution state.

Conceptually:

```text
CreateExecution
      |
      v
Execution State
      |
      v
Execution Permit
```

The execution is bound to its execution identity and relevant intent/mandate context.

The resulting shared state becomes the reference point for later operations.

---

## 10. ConsumeExecutionPermit

`ConsumeExecutionPermit` provides a transactional control point for execution.

Conceptually:

```text
Execution Permit
      |
      v
ConsumeExecutionPermit
      |
      v
Consumed
```

The transaction reads the current permit state and writes the consumed state.

This operation is important for duplicate and concurrent execution scenarios.

---

## 11. Concurrent Permit Consumption

Two participants or processes may attempt to consume the same permit concurrently.

Conceptually:

```text
                 Execution Permit
                       |
              +--------+--------+
              |                 |
              v                 v
          Attempt A         Attempt B
              |                 |
              v                 v
          Read State          Read State
              |                 |
              v                 v
           Commit          Stale Version
                                |
                                v
                           MVCC Conflict
```

The shared transactional state and MVCC validation prevent both stale concurrent writes from being treated as valid updates to the same state.

This scenario should be explicitly demonstrated during implementation.

---

## 12. RegisterAttestation

`RegisterAttestation` records that a participant attestation has entered the shared execution-assurance state.

The detailed signed event does not need to become a raw shared log.

Instead, the shared state can contain an attestation reference and associated commitment.

Conceptually:

```text
Participant Event
      |
      v
Canonical Event
      |
      v
Hash + Signature
      |
      v
Attestation Gateway
      |
      v
RegisterAttestation
      |
      v
Shared Attestation Reference
```

The gateway remains responsible for validation before the transaction is submitted.

---

## 13. AdvanceExecutionState

`AdvanceExecutionState` records an allowed execution-state transition.

Conceptually:

```text
Current State
      |
      v
Validate Transition
      |
      v
AdvanceExecutionState
      |
      v
Next State
```

The transaction should validate the expected current state/version before writing the next state.

This creates a deterministic shared state transition boundary.

---

## 14. RecordVerification

`RecordVerification` stores a reference to a completed verification result.

The detailed verification computation remains within Ledgr's assurance services.

Drunix stores the cross-organizational reference required to establish that verification has been recorded.

Conceptually:

```text
Reconstructed Execution
        +
Expected Execution Model
        |
        v
Verification
        |
        v
RecordVerification
        |
        v
Shared Verification Reference
```

---

## 15. RecordDivergence

`RecordDivergence` records the shared reference to an identified divergence.

The divergence can reference:

```text
execution_id
divergence_id
event_id
verification_id
reason
evidence_reference
```

The detailed evidence remains available through the appropriate evidence store.

The shared record establishes the existence and identity of the assurance finding.

---

## 16. CommitAssurance

`CommitAssurance` records the integrity commitment for an Execution Assurance Record.

The process is:

```text
Execution Assurance Record
          |
          v
Canonical Serialization
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

The shared state therefore contains an integrity reference without requiring the complete assurance document to be stored in the shared ledger.

---

## 17. Assurance Commitment

The assurance commitment can conceptually contain:

```text
record_id
execution_id
assurance_hash
verification_reference
divergence_reference
created_at
record_version
```

The commitment provides a shared reference to the assurance result produced by Ledgr.

---

## 18. State Versioning

Shared state is versioned.

Conceptually:

```text
state_version
expected_version
```

A transaction reads a state version and proposes an update against that expected version.

```text
Read Version N
      |
      v
Prepare Update
      |
      v
Validate Expected Version
      |
      +----> Match ----> Commit
      |
      +----> Stale ----> MVCC Conflict
```

This mechanism is especially relevant to:

- execution permits
- execution state
- concurrent retries
- duplicate execution attempts

---

## 19. Transaction Lifecycle

A transaction follows the shared transactional lifecycle:

```text
Client Request
      |
      v
Proposal
      |
      v
Endorsement
      |
      v
Ordering
      |
      v
Validation
      |
      v
MVCC Check
      |
      v
Commit
```

Ledgr should expose this flow in the prototype rather than hiding the core Drunix interaction behind screenshots or a simulated ledger.

---

## 20. Endorsement and Validation

Critical shared-state operations are subject to the network's transaction validation and endorsement model.

The chaincode must remain deterministic so that participating organizations can validate the same state transition.

The architecture relies on:

- authenticated transactions
- endorsement
- deterministic chaincode
- ordering
- validation
- MVCC

---

## 21. Chaincode Boundary

Ledgr application services should perform logic that requires broader execution analysis.

Examples:

```text
Application / Assurance Services
    |
    +--> Evidence validation
    +--> Event normalization
    +--> Reconstruction
    +--> Expected-state derivation
    +--> Verification
    +--> Divergence localization
    +--> EAR generation
```

Drunix chaincode should focus on:

```text
Shared Transactional State
    |
    +--> Execution creation
    +--> Permit consumption
    +--> Attestation registration
    +--> State transition
    +--> Verification reference
    +--> Divergence reference
    +--> Assurance commitment
```

This keeps the shared transactional layer explicit and deterministic.

---

## 22. Hard Invariants

The shared state should enforce critical invariants.

Examples include:

### Execution Identity

An execution state transition must remain bound to the correct `execution_id`.

### Permit Consumption

A consumed execution permit must not be treated as simultaneously available for another logical execution.

### State Version

An update must operate against the expected state version.

### Attestation Binding

An attestation reference must remain associated with the correct execution.

### Assurance Binding

An assurance commitment must reference the correct execution and assurance record.

These invariants form part of the transactional boundary.

---

## 23. Duplicate Retry Example

Suppose an execution permit is available:

```text
permit_status = AVAILABLE
state_version = 7
```

Two concurrent attempts read version `7`.

```text
Attempt A
expected_version = 7

Attempt B
expected_version = 7
```

If Attempt A commits first:

```text
permit_status = CONSUMED
state_version = 8
```

Attempt B now operates against stale version `7`.

Its transaction should fail validation through the MVCC conflict rather than producing another valid consumption of the same shared state.

---

## 24. Evidence Commitment

Detailed evidence may remain private or off-chain.

A commitment can still be placed into shared state.

Conceptually:

```text
Detailed Evidence
       |
       v
Canonical Representation
       |
       v
Hash / Commitment
       |
       v
Drunix
```

This provides a shared integrity reference without requiring all detailed evidence to become shared state.

---

## 25. Privacy Boundary

The Drunix design follows a minimization principle.

Only information necessary for cross-organizational assurance should be shared.

```text
                 Cross-Organization
                       |
                       v
                  Drunix State
                       |
          +------------+------------+
          |                         |
          v                         v
     Commitments               References
          |
          v
Detailed evidence remains
in appropriate private/off-chain systems
```

This supports selective disclosure and controlled access to sensitive information.

---

## 26. Transaction Mapping

| Operation | Primary State | Purpose |
|---|---|---|
| `CreateExecution` | Execution | Create shared execution state |
| `ConsumeExecutionPermit` | Permit | Prevent duplicate logical execution |
| `RegisterAttestation` | Attestation reference | Register participant evidence |
| `AdvanceExecutionState` | Execution state | Record shared state transition |
| `RecordVerification` | Verification reference | Record verification result |
| `RecordDivergence` | Divergence reference | Record first observable divergence |
| `CommitAssurance` | Assurance commitment | Commit EAR integrity hash |

---

## 27. End-to-End Drunix Flow

A complete assurance scenario can use:

```text
CreateExecution
       |
       v
ConsumeExecutionPermit
       |
       v
RegisterAttestation
       |
       v
AdvanceExecutionState
       |
       v
RecordVerification
       |
       v
RecordDivergence
       |
       v
CommitAssurance
```

Not every execution must necessarily invoke every operation in every branch; the transaction sequence depends on the execution scenario.

---

## 28. Prototype Demonstration Requirements

The prototype should visibly demonstrate actual Drunix interaction.

At minimum, the demonstration should show:

1. Execution creation.
2. Execution permit creation/consumption.
3. Participant attestation registration.
4. Shared execution-state transition.
5. Concurrent permit-consumption attempt.
6. MVCC conflict for the stale concurrent transaction.
7. Verification reference.
8. Divergence reference.
9. Assurance hash commitment.

The objective is to demonstrate that Drunix is part of the actual execution-assurance flow rather than only an architectural label.

---

## 29. Relationship to Ledgr Services

The complete architecture is:

```text
Participant Systems
        |
        v
Attestation Gateway
        |
        v
Reconstruction
        |
        v
Expected-State Model
        |
        v
Verification
        |
        v
Divergence
        |
        v
Execution Assurance Record
        |
        v
SHA-256 Commitment
        |
        v
Drunix
```

Drunix provides shared transactional state and commitments.

Ledgr provides the execution-assurance processing around that shared state.

---

## 30. Design Principle

The Drunix layer should remain:

- explicit
- deterministic
- minimal
- transaction-oriented
- versioned
- provenance-aware

Drunix is the shared transactional state and commitment layer for Ledgr, not a replacement for participant databases and not a repository for every raw execution record.
