# Ledgr Evaluation Plan

## 1. Purpose

This document defines the evaluation plan for Ledgr's execution-assurance prototype.

The evaluation is designed to determine whether the prototype can:

- reconstruct an agentic payment execution from participant observations,
- derive the expected execution state from the authorized mandate and rules,
- identify the first observable divergence,
- preserve evidence and provenance,
- reject replay and invalid execution bindings,
- enforce concurrent execution-permit safety through Drunix,
- detect conflicting observations,
- distinguish missing evidence from observed violations,
- and produce a tamper-evident Execution Assurance Record (EAR).

The evaluation should use controlled scenarios with known ground truth so that correctness can be measured rather than demonstrated only through screenshots.

---

## 2. Evaluation Principles

The evaluation follows these principles:

1. **Known ground truth**  
   Each synthetic execution has a predefined expected outcome.

2. **Controlled failures**  
   Failures are injected deliberately so the exact point of divergence is known.

3. **Reproducibility**  
   Scenario inputs, participant events, policy versions, and configuration should be versioned.

4. **Evidence-backed results**  
   A verification result must be traceable to the evidence used to derive it.

5. **Deterministic verification**  
   Re-running the same inputs and policy version should produce the same logical verification result.

6. **Security and correctness together**  
   The evaluation covers both execution-assurance accuracy and Drunix transactional/security invariants.

7. **Performance as a measured property**  
   Latency and throughput should be measured rather than claimed.

8. **Implementation honesty**  
   Results must distinguish implemented and measured behavior from planned capabilities.

---

## 3. Evaluation Architecture

The benchmark environment should contain:

```text
Scenario Generator
       │
       ▼
Participant Simulators
 ┌─────┼───────────┐
 │     │           │
Agent PSP        Bank
 │     │           │
 └─────┼───────────┘
       │
       ▼
Attestation Gateway
       │
       ▼
Drunix Shared State
       │
       ├───────────────┐
       ▼               ▼
Reconstruction     Evidence Store
       │
       ▼
Expected-State Engine
       │
       ▼
Verification
       │
       ▼
Divergence Localization
       │
       ▼
Execution Assurance Record
       │
       ▼
Drunix Assurance Commitment
       │
       ▼
Evaluation / Metrics
```

The same scenario should be able to exercise the complete vertical slice.

---

## 4. Minimum Valuable Evaluation

The minimum valuable vertical slice should execute one complete payment lifecycle through:

```text
Observe
  ↓
Reconstruct
  ↓
Expected State
  ↓
Verify
  ↓
First Divergence
  ↓
Evidence / Provenance
  ↓
Assurance Record
  ↓
Drunix Commitment
```

The first evaluation milestone should therefore prioritize one complete deterministic path before expanding the number of scenarios.

---

## 5. Ground-Truth Model

Every benchmark execution should have an explicit ground-truth specification.

A conceptual record can contain:

```json
{
  "execution_id": "EXE-7821",
  "expected_outcome": "DIVERGENCE",
  "expected_divergence_type": "AMOUNT_MISMATCH",
  "expected_first_divergence": "BANK_DEBITED",
  "expected_amount": 15000,
  "observed_amount": 16200,
  "expected_policy_version": "policy-v1"
}
```

The ground truth is not itself an input to the verification engine. It is used by the benchmark evaluator to compare the system's output with the known scenario outcome.

This prevents the evaluator from treating the system's own answer as its reference answer.

---

## 6. Scenario Matrix

The initial evaluation matrix should include at least the following cases.

| Scenario | Expected result | Main capability |
|---|---|---|
| Normal execution | Verified / no divergence | Baseline reconstruction |
| Amount divergence | Divergence at amount transition | Constraint verification |
| Invalid transition | Divergence | State-transition validation |
| Missing evidence | Unknown/incomplete | Uncertainty handling |
| Conflicting evidence | Conflict / divergence as applicable | Multi-source comparison |
| Late evidence | Reverification/update | Evidence lifecycle |
| Replay | Rejected | Replay protection |
| Cross-execution replay | Rejected | Execution binding |
| Concurrent retry | One valid, one conflict | MVCC correctness |
| Tampered evidence | Rejected | Cryptographic integrity |

Additional scenarios can be added after the baseline suite is stable.

---

# 7. Scenario 1 — Normal Execution

## Purpose

Establish the baseline behavior when all participant observations conform to the authorized execution model.

## Example

Mandate:

```text
Book Bengaluru → Delhi
Maximum amount: ₹15,000
```

Observed:

```text
REQUESTED       ₹14,800
PSP_ACCEPTED    ₹14,800
BANK_DEBITED    ₹14,800
```

## Expected result

- all signatures valid,
- execution binding valid,
- expected transitions satisfied,
- no first divergence,
- assurance record indicates a consistent execution,
- provenance references all relevant evidence.

## Metrics

- reconstruction latency,
- verification latency,
- end-to-end assurance latency,
- evidence completeness,
- Drunix commit latency.

---

# 8. Scenario 2 — Amount Divergence

## Purpose

Test whether the verifier identifies the first observable amount violation.

## Example

Expected:

```text
Maximum authorized amount = ₹15,000
```

Observed:

```text
REQUESTED       ₹14,800
PSP_ACCEPTED    ₹14,800
BANK_DEBITED    ₹16,200
```

## Expected result

The first observable divergence is:

```text
BANK_DEBITED
```

because the observed debit violates the authorized amount constraint.

The result should reference:

- bank attestation,
- expected amount constraint,
- policy version,
- execution identifier,
- and verification identifier.

## Metrics

- first-divergence accuracy,
- amount-constraint detection,
- evidence completeness,
- verification latency.

---

# 9. Scenario 3 — Invalid Transition

## Purpose

Test whether the state machine detects an execution transition that is not permitted by the expected model.

## Example

Expected:

```text
REQUESTED
  ↓
PSP_ACCEPTED
  ↓
BANK_DEBITED
```

Observed:

```text
REQUESTED
  ↓
BANK_DEBITED
```

## Expected result

The verifier should identify the earliest transition that violates the expected transition graph.

The result should not simply report that the final state is invalid. It should localize the earliest observable inconsistent transition supported by the available evidence.

---

# 10. Scenario 4 — Missing Evidence

## Purpose

Test how the system behaves when an expected participant observation is absent.

## Example

Observed:

```text
REQUESTED
PSP_ACCEPTED
```

No bank observation is available.

## Expected result

The system should not automatically conclude:

```text
BANK_DEBIT_FAILED
```

or:

```text
BANK_DEBIT_NEVER_OCCURRED
```

Instead, it should represent the missing observation as an incomplete/unknown condition.

The result should preserve:

- evidence actually observed,
- missing evidence,
- verification status,
- and the policy/model used.

## Metric

Measure correct classification of incomplete evidence versus false divergence detection.

---

# 11. Scenario 5 — Conflicting Participant Observations

## Purpose

Test whether authenticated but inconsistent observations are preserved and analyzed.

## Example

```text
PSP:
payment accepted = ₹14,800

Bank:
debit completed = ₹16,200
```

Both observations may have valid signatures.

## Expected result

The system should:

1. accept both cryptographically valid observations,
2. preserve their provenance,
3. identify the inconsistency,
4. compare the observations against the expected model,
5. determine the earliest evidence-backed divergence where possible,
6. retain the conflicting records in the assurance evidence set.

The benchmark should verify that one participant's observation does not silently overwrite another participant's observation.

---

# 12. Scenario 6 — Late Evidence

## Purpose

Test evidence arriving after an initial verification.

## Initial state

Only part of the expected execution has been observed.

The system may initially produce:

```text
INCOMPLETE / UNKNOWN
```

## Later event

A valid participant attestation arrives.

## Expected behavior

The system should be able to process the new evidence according to the implemented evidence lifecycle and, where supported, rerun verification.

The resulting record must make clear:

- which evidence version was evaluated,
- when the evidence arrived,
- which policy version was used,
- and whether the assurance result changed.

The evaluation should not silently rewrite the history of the earlier verification.

---

# 13. Scenario 7 — Replay

## Purpose

Verify that a previously accepted attestation cannot create a second logical observation.

## Procedure

1. Generate a valid signed attestation.
2. Submit it.
3. Confirm successful registration.
4. Submit the exact same attestation again.
5. Inspect shared state and verification inputs.

## Expected result

The second submission must not create an additional logical attestation effect.

The exact API response/status is an implementation detail, but the final logical state must remain idempotent.

## Metrics

- replay rejection rate,
- duplicate-state creation count,
- replay detection latency.

---

# 14. Scenario 8 — Cross-Execution Replay

## Purpose

Verify that an authentic event from one execution cannot be reused in another execution.

## Procedure

```text
Create EXEC-A
Create EXEC-B

Create valid attestation for EXEC-A

Attempt:
submit attestation as evidence for EXEC-B
```

## Expected result

The event must be rejected because its execution binding does not match the target execution.

This test is distinct from ordinary replay because the event itself may be cryptographically valid and previously unused in the target execution.

---

# 15. Scenario 9 — Concurrent Retry / MVCC

## Purpose

Test the core shared-state concurrency invariant.

## Setup

Create one execution permit:

```text
execution_id = EXE-7821
permit = available
```

Then issue two concurrent transactions:

```text
Client A → ConsumeExecutionPermit
Client B → ConsumeExecutionPermit
```

Both proposals can initially reference the same state version.

## Expected result

After ordering and validation:

```text
Transaction A → valid / committed
Transaction B → invalid due to stale state / MVCC conflict
```

or the reverse.

The benchmark must not assume which client wins.

## Acceptance condition

Exactly one valid committed logical permit consumption exists.

## Metrics

- successful consumption count,
- invalid transaction count,
- MVCC conflict count,
- transaction latency,
- end-to-end assurance latency.

---

# 16. Scenario 10 — Tampered Evidence

## Purpose

Test cryptographic integrity.

## Procedure

1. Create a canonical participant event.
2. Sign the event.
3. Modify one signed field, such as amount.
4. Submit the modified event.

## Expected result

Signature verification must fail.

The altered event must not become accepted evidence.

## Metrics

- tamper detection rate,
- false acceptance count,
- verification latency.

---

# 17. First-Divergence Accuracy

First-divergence localization is a primary correctness metric.

For every controlled failure, the benchmark knows the ground-truth first divergence.

Let:

```text
D_gt = ground-truth first divergence
D_pred = system first divergence
```

A scenario is correctly localized when:

```text
D_pred == D_gt
```

The benchmark should report:

```text
first_divergence_accuracy =
correctly localized scenarios / scenarios with known divergence
```

This should be calculated over scenarios where the ground-truth divergence is observable from the supplied evidence.

Cases where evidence is insufficient should not be incorrectly counted as localization failures if the expected benchmark outcome is explicitly `UNKNOWN` or `INCOMPLETE`.

---

# 18. Constraint-Violation Detection

The benchmark should measure whether the verifier detects controlled violations such as:

- amount exceeding mandate,
- invalid state transition,
- unauthorized operation,
- duplicate execution,
- invalid participant binding,
- and other explicitly implemented policy constraints.

For each scenario record:

```text
ground_truth_violation
detected_violation
```

The evaluation should report:

- true detections,
- missed violations,
- incorrect violation classifications,
- and unresolved/unknown cases.

The benchmark should avoid collapsing all failures into a single accuracy number because different failure classes have different meanings.

---

# 19. Conflict Detection

Conflicting observations should be evaluated separately from ordinary violations.

Measure:

- number of conflicting evidence pairs generated,
- number correctly detected,
- number silently overwritten,
- number retained with provenance,
- and number left unresolved.

A key acceptance condition is:

```text
conflicting authenticated observations
        ↓
both remain traceable
```

---

# 20. Evidence Completeness

Evidence completeness measures whether the assurance record contains the references necessary to reconstruct why the result was produced.

For a scenario with required evidence set:

```text
E_required
```

and assurance-referenced evidence:

```text
E_recorded
```

a simple benchmark measure is:

```text
evidence_completeness =
|E_recorded ∩ E_required| / |E_required|
```

The implementation may define a richer weighted measure later.

The evaluation should separately identify:

- required evidence,
- available evidence,
- used evidence,
- unavailable evidence,
- and evidence excluded by policy.

---

# 21. Replay and Binding Tests

The benchmark should report separate results for:

### Same-event replay

```text
valid event → submit → submit again
```

### Cross-execution replay

```text
event for EXEC-A → attempt under EXEC-B
```

### Duplicate API retry

```text
same operation_id → retry
```

### Duplicate permit consumption

```text
same permit → concurrent consumers
```

These are related but should remain distinct metrics because they exercise different controls.

---

# 22. Performance Metrics

The prototype should measure at least:

### Attestation verification latency

Time from attestation receipt to cryptographic/schema/binding validation result.

### Reconstruction latency

Time required to produce the reconstructed execution from available evidence.

### Expected-state evaluation latency

Time required to construct/evaluate the expected execution model.

### Verification latency

Time from verification start to verification result.

### Drunix commit latency

Time required for the relevant transaction to progress through submission and commit/validation result.

### End-to-end assurance latency

Conceptually:

```text
execution/evidence ingestion
        ↓
reconstruction
        ↓
expected-state evaluation
        ↓
verification
        ↓
divergence
        ↓
EAR
        ↓
Drunix assurance commitment
```

The exact start/end timestamps should be defined consistently in the benchmark implementation.

---

# 23. Throughput

Throughput should be measured using controlled synthetic workloads.

Potential workload dimensions include:

- attestations per second,
- executions per second,
- verification jobs per second,
- Drunix transactions per second,
- concurrent execution count.

The benchmark should record both:

```text
off-chain processing throughput
```

and:

```text
shared-state transaction throughput
```

because these may become different bottlenecks.

---

# 24. Latency Percentiles

Average latency alone can hide tail behavior.

The benchmark should report at least:

```text
p50
p95
p99
```

for major stages where enough samples exist.

Candidate measurements:

- attestation validation,
- reconstruction,
- verification,
- Drunix commit,
- end-to-end assurance.

---

# 25. Scale Evaluation

A staged scale test should be used.

Suggested execution counts:

```text
100
1,000
10,000
larger synthetic sets as infrastructure permits
```

For each scale level measure:

- total processing time,
- throughput,
- p95/p99 latency,
- memory/resource usage where available,
- Drunix transaction behavior,
- queue/backlog behavior,
- and failure/retry rates.

The goal is to identify the point at which the prototype becomes constrained, not to claim production-scale capacity from a small benchmark.

---

# 26. Concurrency Evaluation

Concurrency should be tested around shared execution state.

Important tests include:

1. two consumers for one permit,
2. repeated retries for one execution,
3. concurrent attestations from multiple participants,
4. concurrent verification requests,
5. concurrent updates to the same execution where supported.

The benchmark should verify that concurrent operations do not create impossible committed state.

---

# 27. Determinism Evaluation

The same scenario should be executed repeatedly using identical:

- execution context,
- evidence,
- policy version,
- ordering inputs,
- and configuration.

The logical result should remain stable.

At minimum compare:

```text
verification status
first divergence
violation set
evidence references
assurance hash
```

If timestamps or transaction identifiers are intentionally nondeterministic, those should be excluded from direct equality while remaining traceable.

---

# 28. Assurance-Hash Verification

For each generated EAR:

```text
canonical(EAR)
     ↓
SHA-256
     ↓
assurance_hash
```

The benchmark should verify that:

1. the stored hash matches the canonical EAR,
2. changing a committed field changes the calculated hash,
3. the assurance commitment points to the expected execution/record,
4. the record can be independently rehashed.

This test evaluates integrity commitment, not factual correctness of the underlying evidence.

---

# 29. Reproducibility

Every benchmark run should record enough metadata to reproduce the scenario.

Recommended metadata:

```json
{
  "run_id": "RUN-001",
  "scenario_id": "amount_divergence",
  "execution_id": "EXE-7821",
  "policy_version": "policy-v1",
  "dataset_version": "synthetic-v1",
  "code_version": "<git-commit>",
  "configuration_version": "<config-version>",
  "timestamp": "<run-time>"
}
```

The benchmark should retain scenario inputs and expected results alongside the measured outputs.

---

# 30. Benchmark Dataset

The initial benchmark should use synthetic executions rather than real customer payment data.

Each generated execution can contain:

- execution context,
- mandate,
- participant identities,
- expected transitions,
- signed participant events,
- controlled failure injection,
- ground-truth outcome,
- expected first divergence where applicable.

This allows repeatable testing without exposing real payment information.

---

# 31. Failure Injection

Controlled failure injection should cover at least:

- amount modification,
- invalid transition,
- duplicate event,
- cross-execution event,
- missing event,
- delayed event,
- conflicting participant amount,
- tampered signed payload,
- concurrent permit consumption,
- stale state update.

The benchmark should label injected failures explicitly so evaluation results are not ambiguous.

---

# 32. Evaluation Output

Each scenario run should produce a machine-readable result similar to:

```json
{
  "scenario_id": "amount_divergence",
  "execution_id": "EXE-7821",
  "expected": {
    "status": "DIVERGENCE",
    "first_divergence": "BANK_DEBITED"
  },
  "observed": {
    "status": "DIVERGENCE",
    "first_divergence": "BANK_DEBITED"
  },
  "checks": {
    "signature_valid": true,
    "execution_binding_valid": true,
    "replay_rejected": true,
    "evidence_complete": true
  },
  "metrics": {
    "reconstruction_ms": 0,
    "verification_ms": 0,
    "drunix_commit_ms": 0,
    "end_to_end_ms": 0
  }
}
```

The zero values above are placeholders for the benchmark implementation and must not be presented as measured performance.

---

# 33. Acceptance Criteria

The prototype evaluation should establish, with reproducible evidence, that:

### Correctness

- normal executions can be reconstructed,
- controlled amount divergence is detected,
- invalid transitions are detected,
- first observable divergence is localized where evidence permits,
- missing evidence is not automatically treated as failure,
- conflicting observations are retained and analyzed.

### Security

- tampered signatures are rejected,
- cross-execution replay is rejected,
- duplicate logical submissions are handled safely,
- execution permits cannot be consumed twice through concurrent valid commits.

### Provenance

- verification results reference their evidence,
- policy/rule versions are identifiable,
- divergence records are linked to the relevant evidence,
- assurance records can be independently integrity-checked.

### Shared-state correctness

- Drunix state transitions obey defined invariants,
- MVCC conflicts prevent invalid concurrent state,
- committed state can be inspected independently of the application UI.

### Performance

- latency and throughput are measured,
- p95/p99 behavior is reported where applicable,
- scale behavior is documented,
- bottlenecks are identified rather than hidden.

---

# 34. Evaluation Report Structure

The final benchmark report should contain:

```text
1. Executive summary
2. Environment and versions
3. Scenario definitions
4. Ground-truth definitions
5. Correctness results
6. Security results
7. First-divergence results
8. Evidence/provenance results
9. MVCC/concurrency results
10. Latency results
11. Throughput results
12. Scale results
13. Failure analysis
14. Reproducibility information
15. Limitations
16. Implementation status
```

Each result should identify whether it is:

- measured,
- derived from a controlled test,
- or planned for future implementation.

---

# 35. Recommended Initial Benchmark Sequence

The implementation can proceed through the evaluation suite in this order:

```text
1. Normal execution
        ↓
2. Amount divergence
        ↓
3. Invalid transition
        ↓
4. Missing evidence
        ↓
5. Conflicting observations
        ↓
6. Replay
        ↓
7. Cross-execution replay
        ↓
8. Concurrent permit consumption / MVCC
        ↓
9. Tampered evidence
        ↓
10. Late evidence
        ↓
11. Performance benchmark
        ↓
12. Scale benchmark
```

This ordering first establishes functional correctness, then security/concurrency behavior, and finally performance.

---

# 36. Demo Evaluation

The reproducible demo should use a concrete end-to-end scenario.

Example:

```text
Mandate:
Book Bengaluru → Delhi
Maximum ₹15,000
```

Execution:

```text
EXE-7821
```

Observed:

```text
Agent request      ₹14,800
PSP accepts        ₹14,800
Bank debits        ₹16,200
```

Expected demonstration:

```text
Observe
  ↓
Reconstruct
  ↓
Expected = max ₹15,000
  ↓
Verify
  ↓
First divergence = BANK_DEBITED
  ↓
Create EAR
  ↓
Commit assurance hash to Drunix
  ↓
Display timeline + evidence
```

A separate concurrency demonstration should show:

```text
INT-900
  ├── ConsumeExecutionPermit → valid
  └── ConsumeExecutionPermit → MVCC conflict
```

The demo should show actual shared-state/chaincode interactions rather than representing Drunix only through a dashboard screenshot.

---

# 37. What the Evaluation Does Not Claim

The evaluation does not establish:

- universal truth of participant observations,
- legal liability,
- production-scale capacity,
- complete fraud prevention,
- replacement of UAP authorization,
- replacement of participant systems of record,
- or correctness of an implementation feature that has not been built and tested.

The evaluation is intended to establish measurable execution-assurance behavior under controlled conditions.

---

# 38. Final Evaluation Checklist

Before considering the prototype evaluation complete, verify:

- [ ] Normal execution passes.
- [ ] Amount divergence is detected.
- [ ] Invalid transition is detected.
- [ ] Missing evidence is classified explicitly.
- [ ] Conflicting evidence is preserved.
- [ ] Replay is rejected or safely idempotent.
- [ ] Cross-execution replay is rejected.
- [ ] Concurrent permit consumption produces one valid committed result.
- [ ] MVCC conflict is observable.
- [ ] Tampered signed evidence is rejected.
- [ ] First-divergence accuracy is measured.
- [ ] Evidence completeness is measured.
- [ ] Policy version is recorded.
- [ ] Assurance hash is independently verified.
- [ ] Reconstruction latency is measured.
- [ ] Verification latency is measured.
- [ ] Drunix commit latency is measured.
- [ ] End-to-end latency is measured.
- [ ] Throughput is measured.
- [ ] p95/p99 latency is reported where applicable.
- [ ] Scale behavior is documented.
- [ ] Benchmark runs are reproducible from versioned inputs.
- [ ] Implementation status is clearly separated from planned work.
