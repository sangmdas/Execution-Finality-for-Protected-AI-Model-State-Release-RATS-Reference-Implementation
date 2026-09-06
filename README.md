# Execution-Finality-for-Protected-AI-Model-State-Release-RATS-Reference-Implementation
Low-overhead execution-finality reference implementation for AI/GPU environments: bounded non-bearer authority, atomic extraction-state control, Finality Sink enforcement, RATS attestation mapping, and performance-aware paths for multi-GPU, streaming, batching, confidential computing, DPU/SmartNIC, firmware, and silicon.
# Execution-Finality for Protected AI Model-State Release

### RATS-Oriented Runnable Reference Implementation for AI / GPU / Confidential-Computing Environments

> **Core architectural rule:**
> **Computation is not authority to release.**

This repository provides a runnable software reference implementation of an **execution-finality architecture for controlling the external release of protected AI model information**.

It implements the logical transition:

```text
Model Computation
        |
        v
Candidate Release
(non-effective externally)
        |
        v
Protected Validation
        |
        v
Protected Extraction State
        |
        v
Atomic Reservation / Consumption
        |
        v
Bounded Non-Bearer Release Authority
        |
        v
Finality Sink Verification
        |
     +--+--+
     |     |
   VALID  INVALID
     |     |
     v     v
 RELEASE  DENY
```

The implementation accompanies the architecture described in:

`draft-das-rats-openai-anthropic-extraction`

and is intended to make the architecture **executable, inspectable, testable, attackable, and replaceable with stronger enforcement components**.

It is particularly relevant to engineers working on:

* AI accelerators;
* confidential GPU infrastructure;
* remote attestation;
* RATS / EAT;
* GPU firmware;
* AI inference serving;
* multi-GPU model serving;
* DPU / SmartNIC enforcement;
* protected DMA and interconnects;
* model extraction resistance;
* privileged inference interfaces;
* sovereign AI infrastructure;
* hardware-rooted authorization enforcement.

---

# 1. Why This Repository Exists

Modern AI security mechanisms can establish several important properties.

Authentication can establish:

```text
Who is requesting the operation?
```

Remote attestation can help establish:

```text
What software or hardware environment is executing?
```

Confidential computing can help establish:

```text
Is protected computation occurring inside an expected environment?
```

Those questions are important, but a separate security question can remain:

```text
Is this specific model-derived artifact
authorized to become externally usable
at this particular moment,
for this particular destination,
under the current security state?
```

A computation can therefore be valid while its external release is not.

This repository implements that separation.

```text
successful computation
        !=
authority for external release
```

The software can compute a protected result while representing that result only as a **Candidate Release**.

The Candidate Release becomes externally usable through the governed path only after its release authority and protected state have been successfully verified at the **Finality Sink**.

---

# 2. What Problem Is Being Addressed?

Protecting a frontier or proprietary AI model is broader than protecting the model-weight file.

Depending on a deployment, sensitive or privileged model information may include:

* detailed probability distributions;
* logits;
* log probabilities;
* embeddings;
* intermediate representations;
* hidden states;
* activation data;
* diagnostic information;
* cached intermediate state;
* KV-related state;
* model metadata;
* privileged evaluation outputs;
* high-information inference artifacts;
* other information subject to a release policy stricter than ordinary model output.

Repeated access to sufficiently rich model information can potentially improve the economics or effectiveness of:

* model extraction;
* reconstruction;
* imitation;
* unauthorized knowledge transfer;
* teacher/student distillation;
* capability replication.

This project does **not** claim to prevent every possible form of model extraction or distillation.

Its narrower objective is to explore a technical mechanism for controlling **identified protected release paths before external effect occurs**.

---

# 3. Architecture

The reference implementation preserves the following logical security ordering:

```text
CandidateRelease
  attributes + payload
       |
       v
ReleasePolicy.allows()
       |
       v
SQLiteExtractionState.reserve()
       |
       v
ReleaseAuthorityIssuer.issue()
       |
       v
FinalitySink.verify()
       |
       v
protected state commit
       |
       v
return payload
```

These are **logical security stages**.

They are **not requirements for five separate network operations, five CPU/GPU transitions, five hardware stages, or five serial remote calls**.

A production implementation may:

* fuse operations;
* pipeline operations;
* batch operations;
* colocate components;
* cache protected information;
* shard extraction state;
* pre-provision bounded authority;
* delegate non-overlapping allowances;
* move enforcement into accelerator firmware;
* implement verification in DPU / SmartNIC hardware;
* enforce release at DMA or interconnect boundaries;
* implement protected state directly in silicon.

The requirement is preservation of the security invariants, not preservation of the Python function-call structure.

---

# 4. Candidate Release

Implemented primarily in:

```text
src/execution_finality/models.py
```

A `CandidateRelease` represents information that has been computed but has **not yet acquired authority for external effect**.

Its load-bearing `ReleaseAttributes` include:

```text
model_id
workload_id
principal_id
release_class
destination
quantity
policy_id
epoch
nonce
scope_key
```

Example:

```python
CandidateRelease(
    attributes=ReleaseAttributes(
        model_id="model-frontier-17",
        workload_id="workload-serving-04",
        principal_id="partner-eval",
        release_class="HIGH_RES_LOGPROBS",
        destination="tenant:acme",
        quantity=1,
        policy_id="policy-frontier-17",
        epoch=7,
        nonce="n-001",
        scope_key="model-frontier-17:partner-eval",
    ),
    payload=b"privileged-logprob-vector",
)
```

Construction of this object does **not** authorize release.

That distinction is intentional.

---

# 5. Candidate Release Granularity

The architecture does **not require one Candidate Release per generated token**.

A Candidate Release may represent, depending on deployment policy:

* one privileged API response;
* one bounded streaming segment;
* one tensor export;
* one activation object;
* one diagnostic object;
* one batch;
* one high-resolution log-probability response;
* one bounded stream;
* another explicitly controlled release unit.

This is important for high-throughput inference.

Internal token generation does not automatically equal external effect.

---

# 6. Protected Validation

Implemented primarily in:

```text
src/execution_finality/policy.py
```

The reference `ReleasePolicy` evaluates whether the proposed release is permitted for the configured:

* model;
* requester/principal;
* release class;
* destination;
* policy identifier;
* security epoch.

The reference implementation deliberately uses explicit checks instead of an implicit default-allow path.

A policy failure results in denial before extraction-state authority is reserved.

---

# 7. Protected Extraction State

Implemented primarily in:

```text
src/execution_finality/state.py
```

The reference implementation uses **SQLite** to demonstrate:

* protected release budgets;
* extraction-state accounting;
* atomic reservations;
* consumption state;
* concurrency behavior;
* replay-related state;
* process-restart persistence;
* security epochs.

SQLite was selected because it is:

* widely understood;
* easy to inspect;
* transactionally mature;
* available through the Python standard library;
* suitable for demonstrating state-machine behavior.

### SQLite is not the proposed production AI hot-path mechanism.

It is an auditable reference backend.

A production deployment may replace it with:

* accelerator-local protected memory;
* confidential-computing state;
* protected firmware state;
* hardware monotonic state;
* sealed epoch state;
* secure persistent memory;
* delegated bounded allowances;
* sharded protected state;
* DPU-resident state;
* another anti-rollback or authority-consumption mechanism.

---

# 8. Atomic Authority Consumption

One security requirement is preventing concurrent consumers from independently spending the same remaining authority.

The reference implementation uses transactional reservation semantics.

A dedicated concurrency test creates multiple concurrent callers competing for the last available release unit.

Expected result:

```text
8 concurrent callers
        |
        v
1 remaining authority unit
        |
        v
exactly 1 successful reservation
```

The other callers are denied.

Atomicity therefore applies to the authority that could otherwise be double-spent.

It does **not** imply that all AI inference requests globally must share one serialized counter.

---

# 9. No Mandatory Global Counter

A hyperscale deployment does not need:

```text
GPU 1 ----\
GPU 2 -----\
GPU 3 ------> ONE GLOBAL COUNTER
GPU N -----/
```

Production state can instead be partitioned by:

* model;
* tenant;
* principal;
* destination;
* release class;
* epoch;
* accelerator;
* region;
* delegated allowance;
* another enforceable security scope.

For example:

```text
Global permitted budget
        |
        +---- bounded allowance A ---> GPU group A
        |
        +---- bounded allowance B ---> GPU group B
        |
        +---- bounded allowance C ---> GPU group C
```

The required invariant is:

> Two independent enforcement domains must not be able to recreate or spend the same bounded authority independently.

The exact distributed protocol is intentionally outside the scope of this Python reference implementation.

---

# 10. Bounded Non-Bearer Release Authority

Implemented primarily in:

```text
src/execution_finality/authority.py
```

The architecture does not use a generic transferable bearer credential as the final authority for arbitrary protected release.

The reference `ReleaseAuthority` is bound to:

```text
authority_id
reservation_id
candidate_digest
model_id
release_class
destination
epoch
scope_key
quantity
```

and authenticated using the reference cryptographic mechanism.

Possession of the authority alone is therefore insufficient for arbitrary reuse.

For example, authority issued for:

```text
Model A
Release Class: HIGH_RES_LOGPROBS
Destination: tenant:acme
Epoch: 7
Candidate Digest: X
```

cannot simply be copied and applied to:

```text
Model B
Destination: tenant:other
Epoch: 8
Candidate Digest: Y
```

The sink verifies the bound context.

---

# 11. Cryptographic Binding

Implemented primarily in:

```text
src/execution_finality/crypto.py
```

The reference implementation uses:

```text
SHA-256
HMAC-SHA-256
canonical JSON serialization
constant-time MAC comparison
```

The purpose is to demonstrate the required **binding semantics**.

HMAC-SHA-256 is not proposed as the only possible production construction.

Production profiles could potentially use:

* device-local protected MAC keys;
* asymmetric signatures;
* hardware-derived keys;
* TEE-bound keys;
* COSE structures;
* secure capability handles;
* accelerator firmware keys;
* cryptographically authenticated protected state;
* another implementation-specific mechanism.

No public-key signature is inherently required for every token or release.

---

# 12. Finality Sink

Implemented primarily in:

```text
src/execution_finality/sink.py
```

The **Finality Sink** is the first governed boundary at which the Candidate Release payload becomes externally usable through the reference path.

Before release, the sink verifies applicable properties including:

* authority authentication;
* Candidate Release digest;
* model binding;
* release-class binding;
* destination binding;
* security epoch;
* scope;
* quantity;
* reservation identity;
* reservation status;
* current protected state.

Only after successful verification is the release allowed through the governed path.

Conceptually:

```text
Candidate Release
       |
       v
Finality Sink
       |
       +--- valid authority/state ---> release
       |
       +--- invalid/stale/replayed ---> deny
```

---

# 13. Why the Sink Rechecks Current Epoch State

A security epoch can change after authority preparation.

For example:

```text
Authority prepared at epoch 7
          |
          v
security event / policy transition
          |
          v
protected state becomes epoch 8
```

An authority previously valid under epoch 7 should not automatically survive that transition.

The Finality Sink therefore rechecks the **current protected extraction-state epoch at release time**.

A regression test verifies:

```text
old authority + new protected epoch = DENY
```

This prevents prepared authority from silently surviving an applicable freshness/revocation transition.

---

# 14. RATS Relationship

Implemented illustratively in:

```text
src/execution_finality/rats.py
```

The repository deliberately separates two concepts.

### A. Attestation information

Evidence that the expected release-control mechanism exists and is operating in an expected state.

### B. Release Authority

The bounded act-specific authority governing a particular Candidate Release.

These should not be conflated.

```text
RATS / Evidence
      |
      v
"Is the expected enforcement mechanism present?"

Release Authority
      |
      v
"May this particular protected release occur?"
```

---

# 15. Illustrative RATS-Style Claims

The repository demonstrates semantic fields corresponding to:

```text
release_control_profile
release_control_enabled
release_control_measurement
model_identity
release_policy_id
security_epoch
extraction_state_id
extraction_state_commitment
rollback_protection
finality_sink_id
finality_sink_class
destination_binding_supported
authority_consumption_mode
alternate_egress_control
release_control_assurance
```

These are **illustrative repository fields**.

They are not claimed to be:

* registered EAT claim numbers;
* registered CWT claim numbers;
* IANA assignments;
* an adopted RATS profile;
* IETF-standardized semantics.

---

# 16. Conservative Assurance Reporting

The software demo intentionally reports:

```text
release_control_assurance = SOFTWARE_REFERENCE

rollback_protection = false

alternate_egress_control = false

finality_sink_class = SOFTWARE_API_EGRESS
```

This is deliberate.

A Python process cannot truthfully prove that all of the following are subordinate to its Finality Sink:

* DMA;
* PCIe;
* peer-GPU transfer;
* debugger access;
* firmware diagnostics;
* telemetry;
* shared host memory;
* accelerator interconnects;
* privileged host access;
* hardware side channels.

The implementation therefore demonstrates **release-control semantics**, while refusing to claim hardware properties it has not established.

---

# 17. Assurance Progression

The repository documents the following qualitative progression:

| Assurance class      | Example implementation                  | Hardware rollback claim |         Alternate-egress claim |
| -------------------- | --------------------------------------- | ----------------------: | -----------------------------: |
| `SOFTWARE_REFERENCE` | Python / SQLite / API sink              |                      No |                             No |
| `SOFTWARE_GATEWAY`   | Hardened serving gateway                |    Deployment-dependent |                   Generally no |
| `ATTESTED_TEE`       | Measured TEE / confidential VM          |                Possible | Partial / deployment-dependent |
| `PROTECTED_FIRMWARE` | Accelerator firmware + protected egress |                Possible |             Potentially strong |
| `SILICON_BACKED`     | Dedicated state + egress primitives     |                Possible |          Potentially strongest |

These names are repository conventions, not IETF registry values.

---

# 18. AI Performance: What This Architecture Does NOT Require

A major purpose of this reference implementation is to make clear that execution finality does **not inherently require**:

```text
remote attestation RTT per generated token

remote policy decision per decode step

global serialized extraction counter

NVRAM write per token

TPM write per token

public-key signature per token

CPU <-> GPU transition per token

Finality decision for every speculative token

Finality decision for every attention operation

Finality decision for every tensor transfer

Finality decision for every MoE expert invocation
```

Those costs are not architectural requirements.

---

# 19. Slow Path vs Fast Path

The architecture separates slower trust establishment from the protected release fast path.

## Possible slow-path operations

```text
platform attestation
model registration
policy provisioning
key establishment
security-epoch initialization
accelerator assignment
extraction-budget delegation
periodic re-attestation
```

These operations need not execute for every token.

## Possible local fast path

```text
Candidate Release classification
        |
        v
local protected-state lookup
        |
        v
bounded authority consumption
        |
        v
local Finality Sink verification
        |
        v
release / deny
```

A production implementation may keep this path local to:

* accelerator firmware;
* TEE;
* protected host-device boundary;
* DPU;
* SmartNIC;
* DMA controller;
* memory controller;
* interconnect controller;
* dedicated silicon.

---

# 20. Streaming Inference

Execution finality does not inherently disable streaming.

A bounded stream can itself be an authorized release class.

For example:

```text
Authorized stream
model = A
destination = tenant:acme
class = privileged-stream
epoch = 7
maximum quantity = Q
```

The stream can then consume the bounded allowance incrementally.

The existence of an active stream does not imply unlimited authority to release unrelated information.

---

# 21. Continuous Batching and Microbatching

The accelerator compute scheduling unit and release-authority unit are separate concepts.

Multiple inference requests can share:

```text
one continuous compute batch
```

while maintaining:

```text
separate principals
separate destinations
separate extraction state
separate bounded release authority
```

Therefore Candidate Release enforcement does not inherently require disabling:

* continuous batching;
* microbatching;
* request coalescing.

---

# 22. Speculative Decoding

Speculative decoding may generate internal draft tokens that never become externally visible.

Those speculative internal computations do not automatically become Candidate Releases.

Conceptually:

```text
draft tokens
   |
   +---- rejected --> remain internal
   |
   +---- accepted --> may contribute to external response
```

Execution finality applies at the applicable externally effective release boundary rather than automatically to every internal speculative operation.

---

# 23. Tensor / Pipeline / Expert Parallelism

Internal transfer among GPUs inside the same protected execution domain does not automatically constitute external effect.

This can include:

* tensor parallelism;
* pipeline parallelism;
* Mixture-of-Experts routing;
* chiplet communication;
* protected interconnect traffic;
* internal activation exchange.

The relevant question is not:

```text
Did information move?
```

but:

```text
Did protected information cross into a less-trusted
or independently authorized domain where it becomes externally usable?
```

That transition may become the relevant Finality boundary.

---

# 24. Disaggregated Prefill / Decode / KV Architectures

Modern inference systems may separate:

```text
prefill
decode
KV/cache services
routing
accelerator workers
network egress
```

Execution finality does not automatically classify every transfer between those components as an external release.

The determination depends on the deployment's trust-domain boundary.

Where those components belong to the same protected domain:

```text
internal transfer != external release
```

Where protected information crosses into another less-trusted or independently authorized domain:

```text
transfer may become Candidate Release
```

---

# 25. No Mandatory Extra Copy

A Finality Sink is a **logical security role**.

It does not necessarily require protected data to be copied through a new software buffer.

Release enforcement could potentially occur at an existing:

* accelerator egress;
* memory-controller boundary;
* DMA boundary;
* protected API boundary;
* interconnect boundary;
* DPU;
* SmartNIC;
* network egress path.

Therefore:

```text
Finality Sink != mandatory additional data copy
```

---

# 26. No Mandatory Persistent Write Per Token

Rollback resistance is a security property.

It is not a requirement to perform:

```text
NVRAM write
TPM increment
database fsync
remote ledger transaction
```

for every generated token.

A production profile may use:

* protected epochs;
* sealed checkpoints;
* accelerator-local allowances;
* firmware-resident protected state;
* delegated bounded authority;
* secure persistent memory;
* another anti-rollback construction.

The reference SQLite backend prioritizes inspectability, not production accelerator throughput.

---

# 27. Legacy and Incremental Deployment

The architecture can be introduced in stages.

### Stage 1 — Software reference / gateway

```text
Existing inference service
        |
        v
software release-control layer
```

Provides useful semantics, but weak assurance against privileged bypass.

### Stage 2 — TEE / confidential computing

```text
Attested protected environment
        |
        v
validation + protected state + sink
```

Reduces reliance on ordinary host software.

### Stage 3 — Accelerator / protected firmware

```text
GPU/NPU firmware
        |
        v
protected state + egress control
```

Moves enforcement closer to physical data egress.

### Stage 4 — Silicon-backed enforcement

Potential dedicated primitives for:

* protected state;
* epoch state;
* atomic consumption;
* model-bound state;
* release verification;
* egress enforcement;
* attestation Evidence.

---

# 28. Existing APIs Need Not Be Renamed

The architecture does not require an application-level API to change from:

```text
ReturnLogProbs()
```

to a completely new application API.

An implementation can internally transform:

```text
ReturnLogProbs()
       |
       v
Candidate Release
       |
       v
protected validation
       |
       v
Finality decision
       |
       v
existing API response
```

This permits incremental deployment behind existing serving interfaces.

---

# 29. Programming Language

The reference implementation is written in:

```text
Python >= 3.11
```

`pyproject.toml` currently declares:

```toml
requires-python = ">=3.11"
```

The implementation deliberately uses **no third-party runtime dependencies**.

Important standard-library modules include:

```text
sqlite3
hashlib
hmac
json
dataclasses
threading
uuid
pathlib
tempfile
statistics
time
```

Python was selected for:

* readability;
* inspectability;
* portability;
* ease of adversarial review;
* easy replacement of individual components.

Python is **not proposed as the production accelerator hot-path language**.

---

# 30. Implementation Provenance

This reference implementation was produced by translating the execution-finality architecture in the accompanying source Internet-Draft into executable software components.

The implementation was **AI-assisted using OpenAI ChatGPT, GPT-5.6 Sol**.

AI assistance was used for:

* architecture-to-code translation;
* module decomposition;
* state-machine implementation;
* adversarial test design;
* documentation;
* packaging;
* security review of the reference behavior.

The implementation was then executed and tested.

It was not obtained from proprietary model-serving source code.

It was **not derived from**:

* OpenAI internal source code;
* Anthropic internal source code;
* NVIDIA firmware;
* NVIDIA confidential-computing implementation code;
* cloud-provider proprietary code;
* leaked or confidential materials;
* reverse engineering of commercial AI infrastructure.

Named companies are used only as technical deployment examples.

---

# 31. Validation Environment

The packaged reference implementation was validated in a virtualized Linux environment.

Observed environment:

```text
Operating System:
Debian GNU/Linux 13 (trixie)

Kernel:
Linux 6.18.35

Architecture:
x86_64

Virtualization:
KVM

Python:
Python 3.13.5

Visible CPUs:
5 virtual CPUs

Reported CPU:
Intel Xeon Platinum 8573C
```

Because the environment is virtualized, the reported CPU should **not** be interpreted as a dedicated hardware benchmark configuration.

---

# 32. GPU / Accelerator Validation

No NVIDIA GPU interface was available in the validation environment.

`nvidia-smi` was not present.

Therefore this repository has **not been performance-validated on**:

* NVIDIA H100;
* NVIDIA H200;
* NVIDIA B100;
* NVIDIA B200;
* NVIDIA GB200;
* NVIDIA Blackwell;
* future NVIDIA Rubin hardware;
* AMD Instinct;
* Google TPU;
* Intel Gaudi;
* AWS Trainium;
* AWS Inferentia;
* another AI accelerator.

No GPU throughput or latency claim should be inferred from this software reference implementation.

---

# 33. Confidential-Computing Validation

The current implementation was not validated as a production enforcement mechanism inside:

* NVIDIA Confidential Computing;
* Intel TDX;
* Intel SGX;
* AMD SEV-SNP;
* Arm CCA;
* confidential GPU composite environments;
* a hardware-backed HSM;
* production TPM monotonic state;
* accelerator secure firmware.

Accordingly, the default assurance values remain deliberately conservative.

---

# 34. Quick Start

Requires:

```text
Python 3.11+
```

No third-party runtime packages are required.

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it:

```bash
. .venv/bin/activate
```

Install the repository:

```bash
python -m pip install -e .
```

Run the demonstration:

```bash
execution-finality-demo
```

Run the complete test suite:

```bash
python -m unittest discover -s tests -v
```

---

# 35. Running Without Installation

The repository can also be run directly:

```bash
PYTHONPATH=src python -m execution_finality.demo
```

Tests:

```bash
PYTHONPATH=src python -m unittest discover -s tests -v
```

Benchmark:

```bash
PYTHONPATH=src python -m execution_finality.benchmark
```

---

# 36. Demo Behavior

The demonstration provisions:

```text
Model:
model-frontier-17

Release class:
HIGH_RES_LOGPROBS

Principal:
partner-eval

Destination:
tenant:acme

Epoch:
7

Budget:
2
```

It then demonstrates:

```text
Release 1 -> ALLOW
Release 2 -> ALLOW
Release 3 -> DENY: budget exhausted

wrong destination -> DENY

stale epoch -> DENY

replayed authority -> DENY
```

It also emits an illustrative RATS-style claim set representing the software-reference assurance level.

---

# 37. Test Coverage

The current repository contains **14 executable tests**.

The validated test suite covers:

### 1. Authorized path

```text
test_allow_path_consumes_budget
```

Verifies successful release and budget consumption.

### 2. Concurrent final-budget consumption

```text
test_only_one_thread_gets_final_budget_unit
```

Multiple concurrent callers compete for the final authority unit.

Only one succeeds.

### 3. Conservative assurance claims

```text
test_software_reference_does_not_overclaim_hardware_assurance
```

Ensures the Python reference implementation does not falsely advertise hardware rollback or alternate-egress enforcement.

### 4. Attribute tampering

```text
test_attribute_tamper_after_authority_issue_denied
```

Changing bound attributes after authority issuance causes denial.

### 5. Security-epoch transition

```text
test_authority_invalid_after_state_epoch_transition
```

Authority prepared under an earlier epoch becomes unusable after protected state advances.

### 6. Budget exhaustion

```text
test_exhaustion_fails_closed
```

No remaining authority results in no governed release.

### 7. Payload modification

```text
test_payload_tamper_after_authority_issue_denied
```

Modifying payload data after authority issuance invalidates the Candidate Release commitment.

### 8. Process restart persistence

```text
test_process_restart_does_not_reset_sqlite_budget
```

Ordinary process restart does not recreate already-consumed SQLite state.

### 9. Quantity enforcement

```text
test_quantity_is_enforced_against_budget
```

A request cannot consume more bounded authority than is available.

### 10. Replay resistance

```text
test_replay_of_consumed_authority_denied
```

Consumed Release Authority cannot simply be replayed.

### 11. Stale epoch

```text
test_stale_epoch_denied
```

Candidate Releases under an unacceptable epoch are denied.

### 12. MAC tampering

```text
test_tampered_authority_mac_denied
```

Modification of authenticated Release Authority data causes verification failure.

### 13. Destination substitution

```text
test_wrong_destination_denied_before_reservation
```

Authority cannot be obtained for an unauthorized destination.

### 14. Recovery semantics

```text
test_cancel_no_effect_restores_capacity_only_while_reserved
```

Capacity can be restored only while the implementation remains in a state establishing that the associated external effect did not occur.

---

# 38. Current Test Result

The test suite was executed with:

```bash
PYTHONPATH=src python -m unittest discover -s tests -v
```

Observed result:

```text
Ran 14 tests

OK
```

This demonstrates that the current test cases pass.

It does **not** constitute:

* formal verification;
* independent security audit;
* production certification;
* cryptographic proof;
* hardware validation;
* IETF approval.

---

# 39. CI

The repository contains:

```text
.github/workflows/ci.yml
```

The CI configuration tests the implementation against:

```text
Python 3.11
Python 3.12
Python 3.13
```

using GitHub's Ubuntu environment.

CI is intended to verify software portability and regression behavior.

It is not GPU or confidential-computing certification.

---

# 40. Repository Structure

```text
execution-finality-rats-reference/
|
+-- .github/
|   +-- workflows/
|       +-- ci.yml
|
+-- docs/
|   +-- ARCHITECTURE.md
|   +-- ASSURANCE_LEVELS.md
|   +-- ENGINEERING_FAQ.md
|   +-- IMPLEMENTATION_GAPS.md
|   +-- LEGACY_DEPLOYMENT.md
|   +-- PERFORMANCE.md
|   +-- RATS_MAPPING.md
|   +-- REFERENCES.md
|   +-- THREAT_MODEL.md
|   |
|   +-- reference/
|       +-- draft-das-rats-openai-anthropic-extraction-01.xml
|
+-- scripts/
|   +-- run_demo.sh
|   +-- run_tests.sh
|
+-- src/
|   +-- execution_finality/
|       +-- __init__.py
|       +-- authority.py
|       +-- benchmark.py
|       +-- crypto.py
|       +-- demo.py
|       +-- engine.py
|       +-- models.py
|       +-- policy.py
|       +-- rats.py
|       +-- sink.py
|       +-- state.py
|
+-- tests/
|   +-- common.py
|   +-- test_concurrency.py
|   +-- test_rats_claims.py
|   +-- test_security.py
|   +-- test_state_recovery.py
|
+-- CONTRIBUTING.md
+-- GITHUB_UPLOAD.md
+-- Makefile
+-- NOTICE.md
+-- README.md
+-- SECURITY.md
+-- SHA256SUMS
+-- pyproject.toml
```

---

# 41. Mapping to the Architectural Security Properties

| Property                                  | Reference behavior                                                                     |
| ----------------------------------------- | -------------------------------------------------------------------------------------- |
| **R1 — Computation / Release Separation** | A `CandidateRelease` may exist without becoming externally usable                      |
| **R2 — Release-Specific Binding**         | Authority binds Candidate digest, model, class, destination, epoch, scope and quantity |
| **R3 — Rollback Resistance**              | Partial software demonstration only; hardware/VM rollback protection is not claimed    |
| **R4 — Atomic Reservation / Consumption** | Transactional state prevents concurrent double use of final authority                  |
| **R5 — Replay Resistance**                | Consumed authority/reservations become terminal                                        |
| **R6 — Alternate-Path Closure**           | Not claimed by the Python implementation                                               |
| **R7 — Fail-Closed Release**              | Failed verification causes no payload return through the governed sink                 |
| **R8 — Attestable Enforcement State**     | Illustrative claim mapping only; no real hardware EAT Attester                         |

---

# 42. Threat Model

## Demonstrated by the software reference

The repository directly tests or models:

* replay;
* Candidate payload modification;
* Candidate attribute modification;
* wrong destination;
* stale epoch;
* security-epoch change after authority preparation;
* extraction-budget exhaustion;
* quantity overflow;
* concurrent double consumption;
* ordinary process restart;
* release-authority authentication failure.

## Architectural threats documented but not solved by Python alone

* VM snapshot rollback;
* filesystem/disk snapshot rollback;
* privileged-host compromise;
* malicious accelerator driver;
* malicious firmware;
* DMA bypass;
* PCIe bypass;
* debugger bypass;
* diagnostic interface bypass;
* telemetry bypass;
* peer-GPU transfer;
* shared-memory bypass;
* interconnect bypass;
* side channels;
* covert channels;
* attestation-key compromise;
* Verifier compromise;
* distributed state desynchronization.

---

# 43. Critical Limitation: Alternate Egress

The Python Finality Sink controls only code paths that invoke it.

For example:

```text
protected payload
     |
     +---- FinalitySink ----> governed release
     |
     +---- hypothetical DMA bypass
     |
     +---- hypothetical debug interface
     |
     +---- hypothetical peer-GPU path
```

The reference code does not physically prevent those hypothetical alternate paths.

Therefore:

```text
alternate_egress_control = false
```

is intentional.

A stronger deployment requires a credible closure argument establishing that all equivalent protected-to-unprotected release paths:

```text
either traverse the Finality Sink
OR
remain technically subordinate to the same release decision
```

---

# 44. Critical Limitation: Hardware Rollback

SQLite survives normal process restart.

That does not mean it prevents an attacker with sufficient privilege from restoring:

```text
old VM snapshot
old disk image
old filesystem snapshot
old trusted-machine state
```

Therefore SQLite persistence must **not** be described as hardware anti-rollback protection.

A production design requires stronger protected freshness/epoch state.

---

# 45. Critical Limitation: Exactly-Once External Effect

The reference sink uses conservative:

```text
consume-before-return
```

behavior.

This provides an at-most-once property for the governed in-process return path.

However:

```text
authority committed
       |
       v
process crashes
       |
       v
caller may never receive payload
```

In that case, budget may remain consumed even though delivery was unsuccessful.

That is an availability tradeoff.

The implementation deliberately prefers this outcome over silently recreating authority when the external-effect status is uncertain.

Production profiles should specify:

* reservation;
* prepare;
* commit;
* acknowledgement;
* timeout;
* poison state;
* idempotency;
* recovery.

---

# 46. Critical Limitation: Distributed Systems

The reference implementation uses one SQLite state domain.

It does not solve synchronization for:

```text
thousands of GPUs
multiple datacenters
multiple clouds
multiple administrative domains
```

A production deployment may require:

* delegated non-overlapping allowances;
* sharded extraction state;
* protected coordinator services;
* distributed epoch mechanisms;
* authority reconciliation;
* revocation handling.

The security invariant remains:

```text
the same bounded authority
must not be independently recreated
in two enforcement domains
```

---

# 47. Critical Limitation: Destination Identity

The demonstration uses simple strings such as:

```text
tenant:acme
```

A production profile must define what a cryptographically meaningful destination actually represents.

Possibilities include:

* workload identity;
* tenant identity;
* service identity;
* public key;
* protected TLS/channel binding;
* attested receiving environment;
* device identity;
* protected security-domain identifier.

A mutable hostname or IP address alone may not provide sufficient binding for all threat models.

---

# 48. Critical Limitation: RATS Claims Are Illustrative

`rats.py` does **not** produce a complete production EAT.

The implementation does not currently:

* generate real hardware Evidence;
* use a hardware Attesting Environment;
* encode registered EAT claims;
* produce a production CWT;
* wrap Evidence in COSE;
* communicate with a production RATS Verifier;
* process manufacturer Endorsements;
* validate Reference Values;
* implement complete freshness semantics.

It demonstrates the **interoperability information that may need to be represented**, not a finalized RATS protocol.

---

# 49. Side Channels and Covert Channels

Execution finality controls identified governed release paths.

It does not claim universal elimination of:

* timing channels;
* cache channels;
* power channels;
* electromagnetic channels;
* covert communication;
* malicious firmware;
* implementation vulnerabilities;
* information-equivalent uncontrolled interfaces.

A deployment's security analysis must consider these separately.

---

# 50. Ordinary Output Is Outside the Strongest Claim

This project does not claim that a provider can prevent all learning from ordinary outputs that it intentionally exposes.

If ordinary response text is legitimately available, a recipient may potentially use that information for training or analysis.

The stronger execution-finality mechanism is aimed at **protected release classes whose policy is more restrictive than the ordinary interface**.

---

# 51. Benchmarking

Run the included reference microbenchmark:

```bash
PYTHONPATH=src python -m execution_finality.benchmark
```

or after installation:

```bash
execution-finality-benchmark
```

The benchmark measures only the local:

```text
Python
+
SQLite
+
software reference path
```

It does **not** measure:

* NVIDIA GPU latency;
* CUDA overhead;
* GPU utilization;
* confidential-GPU transitions;
* TEE transitions;
* DPU throughput;
* SmartNIC throughput;
* protected DMA;
* HBM-controller enforcement;
* accelerator firmware;
* production distributed inference.

Do not cite the benchmark as evidence of production AI latency.

---

# 52. What a Serious Performance Evaluation Should Measure

A future hardware or production implementation should report at least:

* Candidate Release granularity;
* assurance class;
* added release latency;
* protected-release throughput;
* tokens/sec impact;
* GPU utilization;
* accelerator idle time;
* state contention;
* concurrency scaling;
* tail latency;
* continuous-batching behavior;
* streaming behavior;
* multi-GPU scaling;
* failure-path cost;
* state persistence cost;
* epoch-transition cost.

Results from different assurance levels should not be compared without stating those differences.

For example:

```text
software gateway latency
        !=
silicon-backed egress latency
```

and:

```text
per-response finality
        !=
per-stream-chunk finality
```

---

# 53. Standards References

The architectural work builds on or relates to the following IETF technologies and work areas:

### RFC 9334

**Remote ATtestation procedureS (RATS) Architecture**

Relevant to:

* Attester;
* Evidence;
* Verifier;
* Relying Party;
* Attestation Results;
* Reference Values;
* Endorsements.

### RFC 9711

**The Entity Attestation Token (EAT)**

Relevant to representation of attested claims.

### RFC 10013

**Entity Attestation Token (EAT) Measured Component**

Relevant to measured-component representation.

### Epoch Markers

Relevant to freshness and shared security epochs.

### Trustworthy Device Assignment

Relevant to trustworthy accelerator/device assignment.

### Confidential CPU / GPU Attestation Work

Relevant to composite confidential-computing environments.

### Attested Inference Receipt

Relevant to evidence describing completed inference.

### Application-Layer Action Evidence

Relevant to evidence concerning actions and their associated authority.

This repository specifically explores the distinction between:

```text
evidence that something occurred
```

and:

```text
a technical precondition that must succeed
before protected external effect can occur
```

See:

```text
docs/REFERENCES.md
```

for the versions retained with the accompanying source draft.

---

# 54. Source Internet-Draft

The supplied source document is retained under:

```text
docs/reference/draft-das-rats-openai-anthropic-extraction-01.xml
```

The reference implementation should be read together with that architecture.

The repository does not silently redefine the source architecture.

---

# 55. Related Documentation

For focused review, see:

```text
docs/ARCHITECTURE.md
```

Architecture-to-code mapping.

```text
docs/PERFORMANCE.md
```

AI throughput and hot-path considerations.

```text
docs/ENGINEERING_FAQ.md
```

Non-repetitive engineering questions.

```text
docs/ASSURANCE_LEVELS.md
```

Software-to-hardware assurance progression.

```text
docs/IMPLEMENTATION_GAPS.md
```

Production problems intentionally left open.

```text
docs/LEGACY_DEPLOYMENT.md
```

Incremental deployment strategy.

```text
docs/RATS_MAPPING.md
```

Illustrative RATS/EAT relationship.

```text
docs/THREAT_MODEL.md
```

Threats covered and not covered.

```text
SECURITY.md
```

Security reporting and project security notes.

---

# 56. Standards Status Disclaimer

This repository accompanies an individual Internet-Draft / technical architecture.

It does not imply:

* IETF adoption;
* RATS Working Group adoption;
* working-group consensus;
* IETF endorsement;
* Internet Standard status;
* registered claim semantics;
* IANA assignment.

The implementation exists to support technical evaluation and discussion.

---

# 57. Vendor Disclaimer

References to organizations or technologies such as:

* OpenAI;
* Anthropic;
* NVIDIA;
* GPU infrastructure;
* confidential computing;
* cloud AI infrastructure

are used as technical examples.

No statement in this repository should be interpreted as asserting that any named company:

* uses this architecture;
* has evaluated it;
* endorses it;
* plans to implement it;
* contributed proprietary code;
* is affiliated with the repository author.

The implementation is vendor-neutral.

---

# 58. AI-Generated / AI-Assisted Code Disclaimer

This repository contains AI-assisted software.

AI-generated or AI-assisted security code can contain errors even when tests pass.

The implementation should therefore be:

* independently reviewed;
* fuzzed;
* attacked;
* profiled;
* formally analyzed where appropriate;
* replaced with hardware-backed components before stronger assurance claims are made.

The presence of tests should not be mistaken for proof of security.

---

# 59. Security Audit Disclaimer

This repository has not undergone an independent professional security audit.

It has not been:

* formally verified;
* FIPS certified;
* Common Criteria evaluated;
* production-certified;
* GPU-vendor certified;
* cloud-provider certified;
* RATS interoperability certified.

Security researchers are encouraged to scrutinize:

* replay handling;
* TOCTOU behavior;
* reservation state;
* state rollback;
* destination binding;
* epoch handling;
* concurrent consumption;
* crash recovery;
* authority widening;
* alternate-egress assumptions;
* cryptographic canonicalization;
* Finality Sink placement.

---

# 60. IPR and Licensing

Software copyright licensing and patent licensing are separate matters.

This repository does not itself define patent licensing terms.

The associated Internet-Draft handles applicable IPR disclosure through the relevant IETF process.

Unless a separate applicable license expressly states otherwise:

> **No patent license should be inferred merely from publication, inspection, execution, cloning, or availability of this reference implementation.**

See:

```text
NOTICE.md
```

for repository notices.

---

# 61. Contributing

Critical engineering review is welcome.

Useful contributions include:

* new replay attacks;
* concurrency attacks;
* state rollback tests;
* crash-recovery tests;
* destination-binding alternatives;
* real EAT/COSE integration experiments;
* TEE-backed state implementations;
* GPU/firmware experiments;
* DPU/SmartNIC enforcement;
* accelerator-local state;
* protected DMA experiments;
* performance profiling;
* fuzzing;
* formal state-machine analysis.

See:

```text
CONTRIBUTING.md
```

---

# 62. Security Reporting

Security-sensitive findings should follow the guidance in:

```text
SECURITY.md
```

Please distinguish between:

1. a bug in this Python reference implementation;
2. an architectural issue;
3. an implementation-specific hardware concern;
4. a limitation already explicitly documented.

---

# 63. Intended Future Implementations

The software reference defines replacement boundaries for progressively stronger experimentation.

```text
Python reference
      |
      v
hardened software gateway
      |
      v
attested TEE / confidential VM
      |
      v
accelerator protected firmware
      |
      v
DPU / SmartNIC / protected DMA
      |
      v
silicon-backed Finality Sink
```

Potential future experiments include:

* real EAT generation;
* COSE-protected Evidence;
* RATS Verifier integration;
* Intel TDX;
* AMD SEV-SNP;
* Arm CCA;
* confidential GPU attestation;
* accelerator-local protected state;
* hardware-derived Release Authority;
* DPU/SmartNIC Finality Sink;
* protected DMA;
* multi-GPU bounded delegation;
* distributed extraction budgets;
* hardware security epochs;
* protected streaming release;
* accelerator-level performance testing.

---

# 64. Summary

This repository turns the execution-finality concept into an executable security state machine.

It demonstrates:

```text
Candidate Release
+
explicit release policy
+
bounded extraction state
+
atomic reservation
+
cryptographic Candidate binding
+
bounded non-bearer authority
+
replay resistance
+
security-epoch revalidation
+
independent Finality Sink verification
+
fail-closed governed release
```

It does **not** claim to demonstrate:

```text
hardware-enforced alternate-egress closure
hardware anti-rollback
confidential GPU enforcement
production RATS Evidence
hyperscale distributed state
production GPU performance
formal verification
vendor adoption
IETF adoption
```

That distinction is deliberate.

The purpose of the repository is not to hide implementation gaps behind architectural terminology.

The purpose is to provide a concrete implementation that engineers can:

```text
RUN
INSPECT
BENCHMARK
ATTACK
CRITICIZE
REPLACE
AND HARDEN
```

while preserving the central rule:

> ## Computation is not authority to release.

A protected AI system may compute information successfully while still withholding authority for that information to become externally effective.
