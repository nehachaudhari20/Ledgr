# Ledgr Threat Model

## 1. Purpose

Ledgr is an execution-assurance layer for UAP-style agentic payments. It does not replace payment authorization, participant identity, delegation controls, or the underlying payment rails.

The threat model focuses on whether the execution record presented to Ledgr can be trusted enough to:

- reconstruct what happened,
- compare observed execution against the authorized expected state,
- identify the first observable divergence,
- preserve evidence and provenance,
- prevent duplicate or unauthorized execution state transitions,
- and produce an auditable Execution Assurance Record (EAR).

The design assumes that multiple organizations participate in an execution. Each participant may control its own systems and detailed operational records. Ledgr therefore treats shared state as a limited coordination and assurance layer rather than as a replacement for participant systems of record.

---

## 2. Security Objectives

Ledgr's primary security objectives are:

1. **Authenticity**  
   Evidence submitted by a participant must be attributable to an authenticated participant identity.

2. **Integrity**  
   Evidence, execution state, verification results, and assurance commitments must be tamper-evident.

3. **Execution binding**  
   An observation must be bound to the correct execution, intent, participant, and relevant operation.

4. **Replay resistance**  
   Previously accepted attestations or execution operations must not be reusable as new logical operations.

5. **Deterministic shared state**  
   Critical state transitions must be deterministic and validated through the Drunix transactional model.

6. **Concurrency safety**  
   Concurrent attempts to consume the same execution permit must not both become valid committed state.

7. **Evidence provenance**  
   An investigator must be able to trace an assurance conclusion back to the evidence and state used to derive it.

8. **Privacy minimization**  
   Detailed business records and sensitive information should remain in the systems that own them unless disclosure is required.

9. **Controlled disclosure**  
   Shared records should expose only the information required for coordination, verification, and audit.

10. **Explicit uncertainty**  
    Missing evidence must not automatically be treated as proof of a violation.

---

## 3. Assets

### 3.1 Execution Context

The execution context describes the authorized execution and its binding information.

Representative fields include:

- `execution_id`
- `intent_id`
- mandate or authorization reference
- participant identities
- expected constraints
- execution status
- timestamps
- policy/rule version
- permit state

Compromise of this asset can cause an otherwise valid execution to be evaluated against the wrong mandate.

### 3.2 Execution Permit

The permit represents the ability to perform a particular logical execution.

Security requirements include:

- unique binding to the execution/intent,
- explicit lifecycle state,
- one-time or otherwise controlled consumption semantics,
- transactional protection against concurrent consumption.

### 3.3 Participant Attestations

An attestation is a signed participant observation.

It can represent facts such as:

- payment requested,
- payment accepted,
- debit initiated,
- debit completed,
- fulfilment initiated,
- fulfilment completed,
- rejection,
- timeout,
- reversal.

The signature authenticates the submitting participant and protects the signed representation from modification. It does not by itself prove that the participant's underlying system was truthful.

### 3.4 Reconstructed Execution

This is Ledgr's derived representation of the observed execution timeline and state transitions.

It must remain distinguishable from raw participant evidence because reconstruction is an interpretation of evidence rather than an independent source of truth.

### 3.5 Expected Execution Model

The expected model captures what the execution was authorized to do.

It includes:

- mandate constraints,
- permitted transitions,
- amount limits,
- ordering constraints,
- participant obligations,
- policy version,
- and relevant invariants.

### 3.6 Verification Result

The verification result records the comparison between expected and observed execution.

It should preserve:

- verification identifier,
- policy/rule version,
- inputs or evidence references,
- constraint results,
- transition results,
- confidence/status where applicable,
- divergence reference,
- and verification timestamp.

### 3.7 Execution Assurance Record

The EAR is the durable assurance output.

It should contain or reference:

- execution identity,
- verification result,
- first observable divergence,
- evidence references,
- policy version,
- provenance,
- and an integrity commitment such as an assurance hash.

### 3.8 Shared Ledger State

Drunix stores selected cross-organizational facts and commitments, including:

- execution state,
- execution permit state,
- attestation commitments/references,
- verification records,
- divergence records,
- assurance commitments.

Raw logs, unnecessary PII, and complete private business records should not automatically be placed on shared state.

### 3.9 Cryptographic Material

The system depends on:

- participant signing keys,
- certificate/identity material,
- hashes,
- key identifiers,
- and associated lifecycle metadata.

Key compromise can undermine attribution and evidence integrity, so key lifecycle management is part of the security boundary.

---

## 4. Trust Boundaries

Ledgr crosses several trust boundaries.

### Boundary A — Agent / UAP Environment → Ledgr

The agent or UAP environment supplies execution context and participant observations.

Ledgr must not assume that an observation is trustworthy merely because it arrives through an API. Authentication, signature verification, schema validation, execution binding, and replay checks are required.

### Boundary B — Participant Systems → Attestation Gateway

Banks, PSPs, merchants, fulfilment systems, and other participants may be independently controlled.

A participant is trusted to authenticate its own submissions, but Ledgr should preserve the distinction between:

- authenticated observation,
- verified cryptographic integrity,
- and factual correctness of the underlying event.

### Boundary C — Attestation Gateway → Drunix

The gateway translates accepted observations into transactions against shared state.

The gateway must not be able to bypass the chaincode invariants by directly mutating shared state.

### Boundary D — Ledgr Services → Drunix

Reconstruction and verification services consume shared state and evidence references.

Derived conclusions must be traceable to their inputs and policy version.

### Boundary E — Shared State → Private Participant Systems

Detailed evidence may remain off-chain/private. Shared records can contain commitments or references that allow later verification without exposing all underlying data.

### Boundary F — Investigation / Dashboard → Assurance Data

Investigators and dashboard users should receive only the records permitted by their role and disclosure policy.

---

## 5. Threat Actors

The threat model considers the following actors.

### 5.1 Malicious or Compromised Participant

A participant may submit:

- forged observations,
- altered amounts,
- false timestamps,
- duplicate events,
- events for another execution,
- or conflicting observations.

The system should detect cryptographic, binding, schema, replay, and consistency violations where the available evidence permits detection.

### 5.2 Compromised Participant Key

An attacker possessing a participant's signing key can create apparently authentic attestations.

This is a critical limitation: cryptographic verification proves control of the signing key, not that the participant's internal system produced a truthful business event.

The design therefore needs key lifecycle controls and must preserve participant attribution without treating signatures as universal proof of real-world truth.

### 5.3 Replay Attacker

An attacker reuses a valid previously submitted attestation or transaction.

Examples include:

- resubmitting a debit event,
- reusing an old permit,
- replaying an attestation from another execution,
- replaying an API request after a timeout.

Replay protection requires unique identifiers, execution binding, lifecycle state, and idempotent operation handling.

### 5.4 Malicious Agent or Agent Runtime

An agent may attempt to:

- execute an amount outside the mandate,
- retry an operation,
- initiate conflicting actions,
- or cause a sequence of operations inconsistent with the expected model.

Ledgr is designed to observe and assure the execution rather than inspect the agent's chain-of-thought.

### 5.5 Compromised Ledgr Service

A compromised reconstruction, expected-state, or verification service may attempt to:

- alter derived results,
- suppress evidence,
- misidentify divergence,
- or produce an assurance record inconsistent with the evidence.

The design therefore separates evidence from derived state and commits important outputs to shared state.

### 5.6 Malicious Client / API Caller

An unauthorized caller may attempt to:

- submit attestations,
- query another execution,
- create duplicate executions,
- or invoke protected operations.

Authentication and authorization must be enforced at the API/service boundary.

### 5.7 Colluding Participants

Two or more participants may submit mutually consistent but false observations.

Ledgr cannot guarantee truth when all relevant independent evidence sources are compromised or colluding. The system should preserve provenance and identify the evidence basis of conclusions rather than claim certainty beyond available evidence.

### 5.8 Network / Infrastructure Attacker

An infrastructure attacker may attempt:

- message interception,
- traffic replay,
- service impersonation,
- denial of service,
- or manipulation of non-authoritative transport data.

Transport security, authenticated identities, and ledger-level integrity controls reduce the impact, but availability attacks remain an operational concern.

---

## 6. Threats and Mitigations

| Threat | Example | Primary control |
|---|---|---|
| Forged attestation | Fake bank debit event | Digital signature verification |
| Modified attestation | Amount changed after signing | Signature/hash verification |
| Cross-execution replay | Valid event reused for another execution | Execution binding |
| Same-execution replay | Same event submitted repeatedly | Unique attestation ID + replay state |
| Duplicate permit consumption | Two retries execute one mandate | Drunix transaction + MVCC validation |
| Stale state update | Old state overwrites a newer state | Expected state version + MVCC |
| Unauthorized API action | Caller submits another participant's event | Authentication + authorization |
| Schema abuse | Malformed event causes ambiguous interpretation | Canonical schema validation |
| Evidence suppression | Important observation omitted | Provenance references + reconciliation |
| Conflicting observations | PSP and bank report different amounts | Preserve both authenticated observations |
| False divergence | Missing event treated as violation | Explicit `UNKNOWN/NOT_OBSERVED` state |
| Policy mismatch | Verification uses wrong rules | Policy version recorded with verification |
| Tampered assurance output | EAR changed after verification | Canonicalization + SHA-256 commitment |
| Private-data exposure | Raw PII placed in shared ledger | Data minimization/private storage |
| Key compromise | Attacker signs false events | Key lifecycle, rotation/revocation |
| Service compromise | Verification result manipulated | Evidence-linked verification + shared commitment |
| DoS | Attestation endpoint flooded | Rate limits, queues, isolation |
| Collusion | Multiple participants submit coordinated false facts | Cross-source comparison + provenance |
| Ordering ambiguity | Events arrive out of order | Event timestamps/sequence metadata + deterministic reconstruction |
| Late evidence | Valid evidence arrives after initial verification | Reverification/versioned assurance state |

---

## 7. Cryptographic Threat Model

### 7.1 Signature Verification

Each participant attestation should be represented canonically before signing.

Conceptually:

```text
canonical_event
      ↓
hash / canonical byte representation
      ↓
participant signature
      ↓
attestation submission
      ↓
identity + signature verification
```

Verification should establish:

1. signer identity is recognized,
2. signature is valid,
3. signed bytes match the submitted canonical event,
4. event is bound to the expected execution,
5. event identifier has not already been consumed,
6. event timestamp/sequence is acceptable under the configured policy.

### 7.2 Hash Commitments

Hashing is used for integrity commitments.

For example:

```text
canonical(EAR)
      ↓
SHA-256
      ↓
assurance_hash
      ↓
CommitAssurance
```

A hash does not prove that the underlying record was factually correct. It proves that the committed canonical representation can later be checked for modification.

### 7.3 Key Lifecycle

The implementation should account for:

- key generation,
- secure storage,
- participant identity binding,
- rotation,
- expiration,
- revocation,
- compromised-key handling,
- and auditability of key identifiers.

The precise certificate authority or key-management implementation is a deployment decision and is not fixed by the current design.

---

## 8. Replay and Duplicate Execution Threats

Replay is especially important because agentic payment workflows may retry operations after timeouts or ambiguous responses.

### 8.1 Duplicate API Request

The same logical operation should use a stable operation identifier/idempotency key.

A retry with the same identifier should resolve to the existing logical operation rather than create a second operation.

### 8.2 Duplicate Attestation

An attestation identifier should be unique.

If the same attestation is submitted again:

```text
attestation_id already registered
            ↓
do not create a second logical observation
```

### 8.3 Cross-Execution Replay

An event from `EXEC-A` must not be accepted as an event for `EXEC-B`.

The signed/bound representation should include the execution identifier or equivalent binding context.

### 8.4 Duplicate Permit Consumption

The critical concurrency case is:

```text
Client A ── ConsumeExecutionPermit ──┐
                                      ├── same permit
Client B ── ConsumeExecutionPermit ──┘
```

Both proposals may initially be valid against the same state. After ordering, validation checks the state version.

One transaction can commit against the expected version; the competing transaction becomes invalid because the state has changed.

The invalid transaction must not result in a second valid permit consumption.

---

## 9. Evidence Integrity and Provenance

Evidence should be handled as a chain of references rather than as an untraceable collection of derived conclusions.

A useful relationship is:

```text
Participant Attestation
        │
        ▼
Evidence Reference
        │
        ▼
Reconstructed Execution
        │
        ▼
Expected Execution Model
        │
        ▼
Verification Result
        │
        ▼
Divergence
        │
        ▼
Execution Assurance Record
        │
        ▼
Assurance Hash / Drunix Commitment
```

Each derived stage should preserve enough metadata to answer:

- Which execution was evaluated?
- Which evidence was used?
- Which participant supplied it?
- Which policy version was used?
- Which state/version was evaluated?
- Which verification produced the conclusion?
- Which divergence was identified?
- Which assurance record was committed?

---

## 10. Conflicting Observations

Conflicting authenticated observations are not automatically a cryptographic failure.

For example:

```text
PSP:  payment accepted = ₹14,800

Bank: debit completed = ₹16,200
```

Both records can be authentic signatures from their respective participants while describing incompatible observations.

Ledgr should:

1. retain both observations,
2. preserve participant provenance,
3. reconstruct the conflict,
4. compare the observations with the expected model,
5. identify the earliest transition that becomes inconsistent,
6. classify uncertainty where evidence is insufficient,
7. include the conflicting evidence in the assurance record.

The system should not silently overwrite one observation with another.

---

## 11. Missing and Late Evidence

Absence of an observation is not necessarily proof that an event did not occur.

Therefore:

```text
No evidence observed
        ≠
Evidence disproved
```

The reconstruction layer should distinguish states such as:

- `OBSERVED`
- `NOT_OBSERVED`
- `UNKNOWN`
- `CONFLICTING`
- `INVALID`
- `REJECTED`

A late attestation may cause a previously incomplete execution to become verifiable or may change the classification of a previously unresolved case.

Verification should therefore retain the evidence/version basis used for each assurance result.

---

## 12. First Observable Divergence

The first observable divergence is the earliest evidence-backed execution transition for which the observed execution can no longer be explained by:

1. the authorized expected-state model,
2. valid execution transitions,
3. and applicable invariants.

This definition is deliberately evidence-based.

A missing event should not automatically become the first divergence.

Example:

```text
Expected:
REQUEST ₹15,000
        ↓
PSP_ACCEPT ₹15,000
        ↓
BANK_DEBIT ₹15,000

Observed:
REQUEST ₹15,000
        ↓
PSP_ACCEPT ₹15,000
        ↓
BANK_DEBIT ₹16,200
```

The bank debit transition is the first observable point at which the observed execution violates the amount constraint.

The resulting divergence should reference the evidence that established the mismatch.

---

## 13. Policy and Rule Integrity

Expected-state verification is only meaningful if the correct policy is used.

Each verification should therefore identify:

- policy/rule version,
- effective configuration,
- expected-state model version,
- verification timestamp,
- and relevant execution context.

Changing a rule after an execution should not silently rewrite the historical basis of an earlier verification.

Policy changes should result in a new versioned model.

---

## 14. Privacy Threats

A shared multi-organization execution-assurance layer creates privacy risks if too much business information is replicated.

### Data minimization principle

Shared state should contain only what is required for:

- execution coordination,
- authorization/permit state,
- attestation commitments or references,
- verification references,
- divergence records,
- assurance commitments,
- and auditability.

Detailed records can remain:

- in participant systems,
- private data stores,
- encrypted evidence stores,
- or other controlled off-chain systems.

### Sensitive information

The implementation should avoid unnecessarily placing:

- payment credentials,
- full personal information,
- raw customer documents,
- internal participant logs,
- or unrelated business metadata

into broadly shared ledger state.

---

## 15. Access Control

Access should be role-based and least-privilege.

Potential roles include:

- network/assurance operator,
- participant,
- investigator/dispute operator,
- compliance/regulatory observer,
- service administrator.

A participant should not automatically receive unrestricted access to another participant's private evidence.

Authorization should cover both:

- transaction submission,
- and evidence/query access.

---

## 16. Service and API Threats

Ledgr services should protect API boundaries against:

- unauthorized calls,
- malformed payloads,
- oversized payloads,
- replayed requests,
- duplicate requests,
- injection through structured fields,
- resource exhaustion,
- and enumeration of execution identifiers.

Recommended controls include:

- authenticated service identities,
- authorization checks,
- strict schemas,
- request size limits,
- rate limiting,
- idempotency keys,
- structured logging,
- correlation IDs,
- and audit trails.

The exact API framework and deployment topology are implementation choices.

---

## 17. Availability and Denial of Service

Integrity and assurance are primary objectives, but availability is also relevant.

Potential DoS targets include:

- attestation ingestion,
- reconstruction,
- verification,
- dashboard/API queries,
- and Drunix transaction submission.

The architecture should support:

- asynchronous ingestion,
- queues where appropriate,
- partitioning by `execution_id`,
- backpressure,
- retries with idempotency,
- service isolation,
- and bounded resource consumption.

A DoS condition should not cause the system to silently fabricate a verification result.

---

## 18. Threats to the Verification Pipeline

The verification pipeline is:

```text
Collect
  ↓
Reconstruct
  ↓
Expected State
  ↓
Verify
  ↓
Divergence
  ↓
Evidence / Assurance Record
```

Potential attacks at each stage include:

### Collect

- forged events,
- replay,
- cross-execution injection,
- malformed observations.

Controls:

- signatures,
- identity validation,
- schema validation,
- binding,
- replay checks.

### Reconstruct

- event omission,
- timestamp manipulation,
- ordering ambiguity,
- duplicate interpretation.

Controls:

- canonical event model,
- provenance,
- deterministic reconstruction,
- explicit event identifiers,
- conflict preservation.

### Expected State

- wrong mandate,
- wrong policy version,
- altered constraints.

Controls:

- immutable/versioned authorization context,
- policy version references,
- execution binding.

### Verify

- incorrect rules,
- nondeterministic evaluation,
- hidden evidence changes.

Controls:

- deterministic rule evaluation,
- versioned policies,
- evidence references,
- verification metadata.

### Divergence

- false first-divergence selection,
- treating missing evidence as a violation,
- ignoring conflicts.

Controls:

- evidence-backed definition,
- explicit uncertainty,
- deterministic localization logic,
- retained conflicting evidence.

### Assurance Record

- modified result,
- missing evidence references,
- incorrect commitment.

Controls:

- canonicalization,
- cryptographic hash,
- Drunix commitment,
- immutable/versioned records.

---

## 19. Insider Threats

An authorized operator or service administrator may have excessive access.

The design should reduce insider impact through:

- least privilege,
- role separation,
- auditable operations,
- immutable/shared commitments,
- private evidence boundaries,
- and separation between raw evidence and derived conclusions.

No single application component should be treated as the sole source of truth for the complete execution history.

---

## 20. Threats to Drunix State

Critical chaincode operations should enforce explicit invariants.

Representative operations:

```text
CreateExecution
ConsumeExecutionPermit
RegisterAttestation
AdvanceExecutionState
RecordVerification
RecordDivergence
CommitAssurance
```

Important protections include:

- authenticated transaction submission,
- endorsement,
- deterministic chaincode execution,
- ordering,
- validation,
- MVCC/state-version checks,
- authorization,
- and explicit lifecycle validation.

The application layer should not assume that a successful proposal equals a committed valid state transition. Final validity is established through the transaction lifecycle and commit/validation result.

---

## 21. Security Invariants

The prototype should test at least these invariants.

### Invariant 1 — Execution Binding

An attestation for one execution cannot become evidence for another execution.

### Invariant 2 — Signature Integrity

A modified signed event must fail verification.

### Invariant 3 — Replay Resistance

A previously accepted attestation or operation cannot create a second logical execution effect.

### Invariant 4 — Permit Uniqueness

Two concurrent consumers cannot both successfully consume the same execution permit.

### Invariant 5 — State Version Correctness

A stale state update must not overwrite a newer committed state.

### Invariant 6 — Evidence Provenance

Every verification conclusion must reference the evidence and policy version used to derive it.

### Invariant 7 — Assurance Integrity

A committed assurance hash must correspond to the canonical assurance record.

### Invariant 8 — Privacy Boundary

Private participant details must not be copied into shared state unless explicitly required by the design.

### Invariant 9 — Conflict Preservation

Conflicting authenticated observations must remain distinguishable rather than being silently overwritten.

### Invariant 10 — Missing Evidence Semantics

Missing evidence must not automatically be classified as an execution violation.

---

## 22. Security Testing Scenarios

The implementation should include controlled adversarial scenarios.

### Scenario A — Tampered Attestation

1. Create a valid signed event.
2. Change the amount after signing.
3. Submit the altered event.
4. Verify that signature validation fails.

### Scenario B — Cross-Execution Replay

1. Create an attestation for `EXEC-A`.
2. Attempt to submit it for `EXEC-B`.
3. Verify that execution binding rejects it.

### Scenario C — Same-Event Replay

1. Submit a valid attestation.
2. Submit the identical attestation again.
3. Verify that the second submission does not create a second logical observation.

### Scenario D — Concurrent Permit Consumption

1. Create one execution permit.
2. Submit two concurrent consumption transactions.
3. Verify that only one becomes valid committed state.
4. Verify that the competing transaction is invalidated by state validation/MVCC.

### Scenario E — Conflicting Observations

1. Submit two valid participant observations describing incompatible amounts.
2. Reconstruct the execution.
3. Verify that both observations remain available.
4. Verify that the resulting assurance record references the conflict.

### Scenario F — Missing Evidence

1. Omit an expected participant observation.
2. Run verification.
3. Verify that the missing observation becomes an uncertainty/incomplete-evidence condition rather than an automatically fabricated divergence.

### Scenario G — Policy Version Mismatch

1. Verify an execution with policy version `P1`.
2. Introduce `P2`.
3. Re-run verification explicitly under `P2`.
4. Verify that the historical `P1` basis remains identifiable.

### Scenario H — Tampered Assurance Record

1. Generate an EAR.
2. Compute and commit its assurance hash.
3. Modify the canonical EAR representation.
4. Recompute the hash.
5. Verify that the new hash differs from the committed value.

---

## 23. Residual Risks and Limitations

The design cannot guarantee truth in every real-world situation.

Important residual risks include:

### Compromised participant key

A stolen key can produce cryptographically valid observations until the key is revoked or otherwise contained.

### Compromised participant system

A participant may truthfully sign an event generated by a compromised internal system.

### Collusion

Multiple participants may coordinate false observations.

### Incomplete evidence

If the necessary independent observations are unavailable, Ledgr may only be able to report `UNKNOWN`, `INCOMPLETE`, or `UNRESOLVED`.

### Off-chain evidence availability

A commitment can prove integrity of a referenced record, but if the underlying private evidence is unavailable, investigation may remain incomplete.

### Availability attacks

Drunix, APIs, evidence stores, or participant systems may become unavailable.

### Time synchronization

Cross-system timestamps may differ. Timestamp interpretation should therefore be defined carefully rather than assuming perfect synchronized clocks.

### Policy errors

A perfectly executed verification can still produce an incorrect conclusion if the expected-state model or policy itself is wrong.

These are limitations of the assurance model and should be measured or documented rather than hidden.

---

## 24. Security Design Principles

Ledgr follows these principles:

1. **Authenticate before trusting.**
2. **Bind every observation to an execution.**
3. **Treat signatures as evidence of authorship, not universal proof of truth.**
4. **Preserve conflicting evidence instead of overwriting it.**
5. **Represent missing evidence explicitly.**
6. **Use deterministic shared-state transitions for critical invariants.**
7. **Use MVCC/state versions to protect concurrent state transitions.**
8. **Keep detailed private data outside shared state when possible.**
9. **Version policies and verification inputs.**
10. **Make every assurance conclusion traceable to evidence.**
11. **Commit integrity evidence, not unnecessary sensitive data.**
12. **Never claim an implementation control exists until it has actually been implemented and tested.**

---

## 25. Security Acceptance Criteria for the Prototype

The prototype security demonstration should show, at minimum:

- a valid signed attestation being accepted,
- a tampered attestation being rejected,
- a cross-execution replay being rejected,
- duplicate submission being handled idempotently,
- concurrent permit consumption producing one valid committed result and one MVCC conflict,
- conflicting participant observations being retained,
- missing evidence being represented explicitly,
- a policy version being associated with verification,
- a first observable divergence being linked to evidence,
- and an assurance record whose hash is committed to Drunix.

These tests should produce reproducible logs and machine-readable results where practical.

The threat model is considered an implementation guide and evaluation baseline, not a claim that every control is already implemented.
