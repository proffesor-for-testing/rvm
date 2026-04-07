# SFDIPOT Product Factors Analysis: RVM Microhypervisor

**Framework**: James Bach's Heuristic Test Strategy Model (HTSM) -- Product Factors  
**Subject**: RVM (RuVix Virtual Machine) coherence-native microhypervisor  
**Codebase**: 14 crates, 32,543 LoC (Rust), 895 test functions, `no_std` bare-metal  
**Date**: 2026-04-07  
**Analyst**: QE Product Factors Assessor v3 (SFDIPOT)

---

## Executive Summary

RVM is a bare-metal Rust microhypervisor targeting AArch64 that introduces coherence domains as a first-class scheduling and isolation primitive. The system comprises 14 crates with clear layering: foundation types at the bottom, a HAL boundary, mid-tier subsystems (capabilities, witness, proof, partition, scheduler, memory, coherence), optional extensions (WASM, GPU), and a top-level kernel integration crate.

**Overall Maturity Assessment**: The type system and API contracts are well-designed. The `#![forbid(unsafe_code)]` policy in 13 of 14 crates (HAL excepted) is exemplary. The codebase has strong test coverage at the unit level (895 tests) with thorough boundary testing. However, significant gaps exist in cross-crate integration testing, real hardware validation, and adversarial security testing.

### Key Risk Summary

| Priority | Risk Area | SFDIPOT Factor |
|----------|-----------|----------------|
| P0 | Partition switch hot path untested on real AArch64 hardware | Platform, Time |
| P0 | Security gate P3 verification reconstructs minimal records, not full records | Function, Data |
| P0 | No integration test exercises full boot-to-scheduler-to-coherence pipeline | Structure, Operations |
| P1 | GPU subsystem has zero real hardware integration (all stubs) | Platform, Interfaces |
| P1 | Coherence graph hash collision in `id_to_node` lookup | Data, Function |
| P1 | Memory allocator `free_pages` trusts caller-supplied order parameter | Data, Function |
| P1 | Witness ring buffer overflow under sustained load (262K capacity, 100K ops/sec = 2.6s) | Time, Data |
| P2 | WASM module validation reuses `ProofInvalid` error for unrelated parsing failures | Data |
| P2 | Mincut Stoer-Wagner iteration budget is a proxy for wall-clock time budget | Time |
| P2 | No test for concurrent access to shared structures (`spin` locks under contention) | Time |

---

## 1. STRUCTURE: What the Product IS

### 1.1 Crate Dependency Architecture

The workspace defines a strict layered dependency graph:

```
Layer 0 (Foundation):   rvm-types (zero deps except bitflags)
Layer 1 (HAL):          rvm-hal -> rvm-types
Layer 2 (Security):     rvm-cap -> rvm-types, spin
                        rvm-witness -> rvm-types, spin, [hmac, sha2]
Layer 3 (Proof):        rvm-proof -> rvm-types, rvm-cap, rvm-witness, spin, [sha2, subtle, hmac, ed25519-dalek]
Layer 4 (Domain):       rvm-partition -> rvm-types, rvm-cap, rvm-witness, spin
                        rvm-sched -> rvm-types, rvm-partition, rvm-witness, spin
                        rvm-memory -> rvm-types
                        rvm-coherence -> rvm-types, rvm-partition, [rvm-sched]
Layer 5 (Extensions):   rvm-boot -> rvm-types, rvm-hal, rvm-partition, rvm-witness, rvm-sched, rvm-memory, [sha2, subtle]
                        rvm-wasm -> rvm-types, rvm-partition, rvm-cap, rvm-witness
                        rvm-security -> rvm-types, rvm-witness, [sha2, subtle]
                        rvm-gpu -> rvm-types, [cuda-rust-wasm]
Layer 6 (Integration):  rvm-kernel -> ALL above
```

**Findings:**

- **F-STRUCT-01**: Clean layering with no circular dependencies. `rvm-types` has zero internal crate dependencies (only `bitflags`), serving as a proper foundation.
- **F-STRUCT-02**: `rvm-memory` depends only on `rvm-types`, not on `rvm-coherence`. The coherence-driven tier management is in `rvm-memory` itself, which means the memory subsystem carries coherence knowledge without depending on the coherence crate. This is an intentional design (DC-1 fallback), but creates parallel representations of coherence thresholds.
- **F-STRUCT-03**: `rvm-security` does NOT depend on `rvm-cap` or `rvm-proof`. The security gate evaluates capability tokens passed by value (`CapToken`) and proof commitments passed by value (`WitnessHash`). This is by design (the security crate is a policy decision point, not a policy enforcement point), but it means the security gate cannot verify that a capability actually exists in the capability table.
- **F-STRUCT-04**: `rvm-gpu` is optional (`publish = false`) and depends only on `rvm-types` plus the external `cuda-rust-wasm` crate. It does not depend on `rvm-cap` or `rvm-witness`, meaning GPU operations bypass the capability and witness subsystems at the crate level. The kernel integration crate bridges this gap.
- **F-STRUCT-05**: The `rvm-kernel` crate re-exports all subsystem crates (`pub use rvm_boot as boot;` etc.) and provides a `Kernel` struct that owns all subsystem instances. This is the sole integration point.

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-STRUCT-01 | No integration test covers the full kernel subsystem wiring | High | Certain |
| R-STRUCT-02 | `rvm-security` cannot validate capability existence, only token contents | Medium | Possible |
| R-STRUCT-03 | `rvm-gpu` bypasses witness and capability at crate level | Medium | Possible |
| R-STRUCT-04 | Dual representations of coherence thresholds (rvm-types vs rvm-coherence) could diverge | Low | Unlikely |

### 1.2 Feature Gating Strategy

The workspace uses Cargo feature flags extensively:

- `default` / `std` / `alloc`: Every crate supports `no_std` by default, with `std` and `alloc` as opt-in features that propagate through the dependency chain.
- `crypto-sha256`: Gates SHA-256-based witness signing (HMAC). Default ON in rvm-witness, rvm-proof, rvm-boot, rvm-security.
- `ed25519`: Gates Ed25519 signing in rvm-proof (optional).
- `null-signer`: Gates a null signer for testing (no cryptographic verification).
- `gpu` / `webgpu` / `cuda` / `opencl` / `vulkan` / `wasm-simd`: GPU backend selection.
- `sched`: Enables scheduler integration in rvm-coherence.
- `ruvector`: Placeholder for future ruvector integration in rvm-coherence.
- `coherence` / `coherence-sched` / `wasm`: Feature flags in rvm-kernel for optional subsystems.

**Findings:**

- **F-STRUCT-06**: Feature propagation is correct. `std` in rvm-kernel enables `std` in all 12 dependencies. The `alloc` feature correctly flows through the chain.
- **F-STRUCT-07**: The `null-signer` feature is gated with `#[cfg(any(test, feature = "null-signer"))]` -- it cannot be accidentally compiled into release builds unless explicitly opted in. This is a sound security decision.
- **F-STRUCT-08**: The `strict-signing` feature in rvm-witness is a default feature but appears to be a marker flag (no `#[cfg(feature = "strict-signing")]` guards found in the crate). It may be dead configuration.

### 1.3 Code Quality Attributes

- `#![no_std]` enforced in all 14 crates.
- `#![forbid(unsafe_code)]` in 13 of 14 crates (rvm-hal uses `#![deny(unsafe_code)]` because hardware access requires it).
- `#![deny(missing_docs)]` enforced everywhere -- all public items are documented.
- `#![deny(clippy::all)]` plus `#![warn(clippy::pedantic)]` with targeted exceptions.
- `#[repr(C, align(64))]` on `WitnessRecord` ensures cache-line alignment across architectures.
- Compile-time size assertion: `assert!(core::mem::size_of::<WitnessRecord>() == 64)`.
- `#[must_use]` annotations on all pure functions returning values.

**Test Coverage Mapping (Structure):**

| Test Area | Coverage | Gap |
|-----------|----------|-----|
| Crate compilation (no_std) | High | No CI evidence of bare-metal compilation |
| Feature flag combinations | Low | No matrix testing of feature flag permutations |
| Dependency layering | Medium | Tests exist per-crate but not cross-crate |
| Code quality lints | High | All crates have clippy deny/warn |

---

## 2. FUNCTION: What the Product DOES

### 2.1 Core Functions

#### Partition Lifecycle Management

The partition subsystem (`rvm-partition`) manages the lifecycle of coherence domain containers:

- States: `Created -> Running -> Suspended -> Destroyed` (plus `Hibernated`)
- Valid transitions enforced by `lifecycle::valid_transition()`
- Split: Divides a partition along a mincut boundary using `scored_region_assignment()` which applies a simple heuristic (assign to the partition with closer coherence score)
- Merge: Requires both partitions to exceed `DEFAULT_MERGE_THRESHOLD` (0.70), be adjacent in the coherence graph, and have combined resources within limits

**Findings:**

- **F-FUNC-01**: The split algorithm (`scored_region_assignment`) uses a binary decision (7500 or 2500 weight) based on coherence distance. This is a coarse heuristic that does not consider region size, access patterns, or communication edges. Regions exactly equidistant default to "left" -- this introduces a systematic bias.
- **F-FUNC-02**: The `PartitionManager` has a fixed capacity of `MAX_PARTITIONS` (256). The `PartitionId::MAX_LOGICAL` allows 4096 logical partitions. There is a mismatch: the type system allows IDs up to 4096 but the manager only holds 256 slots.
- **F-FUNC-03**: Merge precondition checks evaluate coherence before adjacency before resources (short-circuit order). This is documented and tested (`test_full_coherence_checked_before_adjacency`).

#### Scheduling

The scheduler (`rvm-sched`) implements a 2-signal priority model:

```
priority = deadline_urgency + (cut_pressure_boost * weight)
```

- Binary max-heap per-CPU run queues (O(log n) enqueue/dequeue)
- Three modes: Reflex (hard real-time), Flow (throughput), Recovery (degraded)
- Degraded mode zeroes all cut pressure, falling back to deadline-only scheduling
- Epoch-based witness logging (DC-10: individual switches are NOT witnessed)

**Findings:**

- **F-FUNC-04**: The `compute_priority()` function adds `deadline_urgency` (u16, max 65535) and `cut_pressure` (u32, up to billions). The addition uses `saturating_add`, but the two signals have vastly different scales. A partition with high cut pressure will always dominate over deadline urgency. The documentation says "priority = deadline_urgency + cut_pressure_boost" but the implementation is `(deadline_urgency as u32).saturating_add(cut_pressure.as_fixed())`.
- **F-FUNC-05**: The scheduler does not implement work-stealing or load balancing across CPUs. Each CPU has an independent run queue. A partition enqueued on CPU 0 will never be picked up by CPU 1, even if CPU 1 is idle.
- **F-FUNC-06**: `MAX_RUN_QUEUE` is 32 entries per CPU. With up to 256 partitions and 8 CPUs, the maximum schedulable partitions are 256, which fits, but if partitions are unevenly distributed across CPUs, some CPUs could overflow while others are idle.

#### Memory Management

The memory subsystem (`rvm-memory`) provides:

- 4-tier model: Hot (SRAM/L1), Warm (DRAM), Dormant (compressed), Cold (persistent)
- Buddy allocator with power-of-two blocks (max order 10 = 4 MiB)
- Region manager with address translation (guest-physical to host-physical)
- Reconstruction pipeline for dormant state restoration
- Overlap detection for both guest-physical and host-physical spaces

**Findings:**

- **F-FUNC-07**: The buddy allocator's `free_pages` trusts the caller-supplied `order` parameter. Freeing with a different order than was allocated can corrupt the bitmap. There is a test comment documenting this: "We do not try to free with the wrong order because the buddy allocator's bitmap tracking would not match cleanly." This is a correctness gap.
- **F-FUNC-08**: Host-physical overlap detection (`regions_overlap_host`) is provided but there is no evidence of it being called during region creation in the kernel integration. This is a critical isolation check that may not be wired up.
- **F-FUNC-09**: The reconstruction pipeline uses LZ4 compression (`lz4_flex` in workspace dependencies). The pipeline correctly handles out-of-order witness deltas and verifies checksums.

#### Coherence Engine

The coherence engine (`rvm-coherence`) implements:

- Fixed-size coherence graph (32 nodes, configurable edges)
- Coherence scoring: ratio of internal edge weight to total edge weight
- Mincut computation: Stoer-Wagner heuristic with iteration budget
- Adaptive recomputation frequency based on CPU load
- EMA filter for smoothing coherence score updates
- Edge weight decay per epoch to prevent stale patterns

**Findings:**

- **F-FUNC-10**: The coherence graph's `id_to_node` lookup uses `partition_id % 256` as a hash. Two partition IDs that differ by a multiple of 256 will collide. The code has a linear-scan fallback, but the primary path assumes no collision. With 4096 possible logical partition IDs and 256 hash slots, collisions are guaranteed at scale.
- **F-FUNC-11**: The `phi_to_coherence_bp()` function is a stub: `if raw >= 10_000 { 10_000 } else { raw as u16 }`. The real implementation ("calibrated transfer function derived from IIT theory") is not implemented. This means the entire coherence-Phi pipeline is operating on a linear identity mapping.
- **F-FUNC-12**: Edge weight decay uses `old_weight * (10000 - decay_bp) / 10000` which truncates to zero for small weights. An edge with weight 1 and 10% decay becomes `1 * 9000 / 10000 = 0` and is pruned. This means low-weight edges are systematically eliminated.

### 2.2 Security Functions

#### Capability-Based Access Control

- 7 rights: READ, WRITE, GRANT, REVOKE, EXECUTE, PROVE, GRANT_ONCE
- 9 capability types: Partition, Region, CommEdge, Device, Scheduler, WitnessLog, Proof, Vcpu, Coherence
- Max delegation depth: 8
- Epoch-based staleness detection
- Derivation tree for revocation propagation

#### Proof-Gated Mutations

- 3-tier proof system: Hash (FNV-1a/SHA-256), Witness (chain verification), Zk (not implemented)
- P1 (< 1 us): Capability check
- P2 (< 100 us): Policy validation with constant-time comparison
- P3 (< 10 ms): Deep proof with witness chain verification

#### Security Gate

The unified security gate (`rvm-security::gate`) is the single entry point for all privileged operations:

1. P1 capability type and rights check
2. P2 proof commitment validation
3. P3 witness chain verification (optional)
4. Witness record emission (both allow and deny)

**Findings:**

- **F-FUNC-13**: ZK proof verification returns `Err(RvmError::Unsupported)`. The comment says "Silently accepting would be a security hole." This is correct defensive programming, but it means the P3 tier is partially non-functional.
- **F-FUNC-14**: The `SignedSecurityGate`'s `verify_chain_signatures` reconstructs a minimal `WitnessRecord` with only `prev_hash`, `record_hash`, and `sequence` set. All other fields are zeroed. If the signer's `sign()` method incorporates other fields (e.g., `action_kind`, `actor_partition_id`), the reconstructed signature will not match. The `DefaultSigner` (FNV-1a-based) does hash all record bytes, meaning this reconstruction is incorrect for any signer that reads beyond the three populated fields.
- **F-FUNC-15**: The security gate emits witness records for both allowed and denied operations. The `actor_partition_id` is hardcoded to 0 in both `emit_allowed` and `emit_rejection`. This means the witness trail does not record which partition attempted the operation.
- **F-FUNC-16**: Capability epoch staleness check in `rvm-security::evaluate()` compares `token.epoch() != epoch`, but the security gate (`gate.rs`) does NOT perform epoch staleness checks. The `PolicyRequest`-based path has this check, but the `GateRequest`-based path does not. This is a potential bypass.

### 2.3 GPU Functions

- Device discovery with GpuDeviceInfo, GpuCapabilities
- Per-partition GPU context with budget enforcement
- Kernel management (compiled compute kernels)
- Buffer management with typed GPU memory buffers
- Command queues with bounded submission
- Coherence acceleration (GPU-accelerated mincut and scoring)

**Findings:**

- **F-FUNC-17**: The GPU subsystem is entirely stub-based. No actual GPU hardware interaction exists. All tests use mock/simulated device info. The `cuda-rust-wasm` dependency is optional and provides transpilation, not runtime execution.
- **F-FUNC-18**: GPU budget enforcement is per-epoch, per-partition. However, there is no integration with the kernel's epoch tick. The GPU budget tracking exists in `rvm-gpu::budget` but the kernel's `tick()` method does not call into it.
- **F-FUNC-19**: The `SwitchContext` struct includes `gpu_queue_head` and `gpu_pt_base` fields gated on `#[cfg(feature = "gpu")]`. The partition switch code does not save/restore these fields (it is a stub), creating a future risk when GPU context switching is implemented.

**Test Coverage Mapping (Function):**

| Test Area | Coverage | Gap |
|-----------|----------|-----|
| Partition lifecycle transitions | High (22 tests) | No test for rapid create/destroy cycling |
| Split/merge preconditions | High (17 tests) | Split region assignment is simplistic |
| Scheduler priority ordering | High (10 tests) | No test for priority inversion scenarios |
| Scheduler mode transitions | Medium | No test for mode switch under load |
| Buddy allocator | High (18 tests) | Wrong-order free not prevented |
| Coherence scoring | Medium (4 tests) | Phi-to-coherence mapping is a stub |
| Mincut computation | Medium (6 tests) | No test for dense graph accuracy |
| Capability derivation | Medium (6 tests) | No adversarial capability forging test |
| Proof verification | Medium (12 tests) | ZK tier not implemented |
| Security gate | High (16 tests) | Epoch staleness not checked in gate path |
| GPU subsystem | Medium (56 tests) | All stubs, no hardware integration |
| Agent lifecycle | Medium (6 tests) | No concurrent agent interaction test |
| Migration protocol | Medium (7 tests) | No test for partial state transfer |

---

## 3. DATA: What the Product PROCESSES

### 3.1 Core Data Types

RVM defines 40+ types in `rvm-types`. Key structural properties:

| Type | Size | Alignment | Key Property |
|------|------|-----------|--------------|
| `WitnessRecord` | 64 bytes | 64 (cache line) | `#[repr(C, align(64))]`, compile-time size assertion |
| `PartitionId` | 4 bytes | 4 | Transparent newtype, lower 8 bits = VMID |
| `CapToken` | 16 bytes | 8 | Unforgeable token with truncated hash |
| `Capability` | ~40 bytes | 8 | Full kernel-internal representation |
| `CoherenceScore` | 2 bytes | 2 | Fixed-point u16, 0..10000 basis points |
| `CutPressure` | 4 bytes | 4 | Fixed-point u32 |
| `CommEdge` | 32 bytes | 8 | Weighted directed edge |
| `MemoryRegion` | ~48 bytes | 8 | Typed, tiered, owned |
| `PhysAddr` / `VirtAddr` / `GuestPhysAddr` | 8 bytes | 8 | Transparent newtypes |

**Findings:**

- **F-DATA-01**: All identifier types (`PartitionId`, `VcpuId`, `CapabilityId`, `OwnedRegionId`, `DeviceLeaseId`, `CommEdgeId`) are transparent newtypes with `#[repr(transparent)]`. This is correct for zero-cost abstraction and FFI safety.
- **F-DATA-02**: `CoherenceScore` uses `from_basis_points()` with clamping: values above 10,000 are silently clamped. This is defensive but could mask bugs where a caller passes an unclamped floating-point conversion.
- **F-DATA-03**: `WitnessRecord.reserved` is private and enforced to be zero. `WitnessRecord.pad` is also private. The `zeroed()` constructor ensures both are zero. There is no validation that these fields remain zero after construction (e.g., if records are read from a buffer).
- **F-DATA-04**: The `ActionKind` enum has 50+ variants spanning 11 subsystems. Hex encoding enables prefix-based filtering (0x0_ = partition, 0x1_ = capability, etc.). This is well-designed for audit queries.
- **F-DATA-05**: `CapToken.truncated_hash()` mixes `id`, `epoch`, and `rights` into a 32-bit hash using XOR. This is not cryptographically secure -- intentional collisions are trivial (`id = 0, epoch = 0, rights = 0` all hash to 0). The documentation says this is "for identification without leaking the full token contents", but the hash provides zero collision resistance.

### 3.2 Data Flow Between Crates

```
rvm-types   --> [type definitions flow down to all crates]
rvm-witness <-- [every crate that mutates state emits WitnessRecords]
rvm-cap     --> [CapToken passed by value through security gate]
rvm-proof   --> [Proof objects validated before state transitions]
rvm-coherence --> [CoherenceScore and CutPressure fed to scheduler]
rvm-kernel  --> [bridges all data flows via Kernel struct]
```

**Findings:**

- **F-DATA-06**: WitnessRecords flow unidirectionally into the WitnessLog. The log is append-only with a ring buffer. When the ring wraps, old records are silently overwritten. There is no drain mechanism visible in the crate API -- the `WITNESS_RING_CAPACITY` comment mentions "overflow drain" but no drain function exists.
- **F-DATA-07**: Coherence scores flow from the coherence engine to the scheduler via the kernel's `sync_partition_scores()` method. This method is called once per epoch tick. Between ticks, the scheduler uses stale coherence data. With a 10ms epoch interval, scores can be up to 10ms old.
- **F-DATA-08**: IPC messages (`IpcMessage`) have a fixed payload of 64 bytes. This matches the WitnessRecord size, enabling zero-copy IPC where the message IS a witness record. However, the IPC manager and witness log are separate systems -- there is no mechanism to correlate an IPC message with its witness record.

### 3.3 State Machine Patterns

**Partition State Machine:**
```
Created --> Running --> Suspended --> Running (cycle)
Running --> Destroyed
Running --> Hibernated --> Reconstructed --> Running
Created --> Destroyed (early termination)
```

**Agent State Machine:**
```
Initializing --> Running --> Suspended --> Running (cycle)
Running --> Migrating --> Running (at destination)
Running --> Hibernated --> Reconstructing --> Running
Any non-terminal --> Terminated
```

**Migration State Machine:**
```
Serializing --> PausingComms --> TransferringRegions --> UpdatingEdges
--> UpdatingGraph --> Verifying --> Resuming --> Complete
Any non-terminal --> Aborted (on timeout or error)
```

**Memory Tier State Machine:**
```
Hot <--> Warm <--> Dormant <--> Cold
(only adjacent transitions, no skipping)
```

**Findings:**

- **F-DATA-09**: The partition state machine allows `Created -> Destroyed` but the `lifecycle::valid_transition()` tests cover this transition. However, there is no test for the `Hibernated -> Destroyed` transition (cleaning up a hibernated partition without reconstruction).
- **F-DATA-10**: The agent state machine allows `Initializing -> Terminated` (never ran), but the `terminate()` implementation frees the slot immediately. If the agent held resources (memory pages), there is no cleanup logic visible.

### 3.4 Data Integrity Mechanisms

- **Hash Chains**: FNV-1a chain linking in WitnessRecords (`prev_hash` = hash of previous record)
- **HMAC Signing**: Optional SHA-256 HMAC signing of witness records (8-byte truncated aux field)
- **Constant-Time Comparison**: `subtle` crate used for timing-safe comparisons in proof verification
- **Compile-Time Assertions**: `const _: () = { assert!(size_of::<WitnessRecord>() == 64); }` ensures layout stability
- **Checkpoint Integrity**: Reconstruction pipeline verifies checksums on compressed checkpoints

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-DATA-01 | WitnessLog ring buffer overwrites without drain notification | High | Likely |
| R-DATA-02 | CapToken.truncated_hash() provides zero collision resistance | Medium | Possible |
| R-DATA-03 | No validation of WitnessRecord reserved/padding fields on read | Low | Unlikely |
| R-DATA-04 | Agent termination does not clean up held resources | Medium | Possible |
| R-DATA-05 | Coherence score clamping silently masks out-of-range inputs | Low | Possible |

---

## 4. INTERFACES: How the Product CONNECTS

### 4.1 Inter-Crate APIs

Each crate exposes a clean public API through its `lib.rs`:

- **rvm-types**: Pure data types, zero behavior (except hash functions)
- **rvm-hal**: Trait definitions (`Platform`, `MmuOps`, `TimerOps`, `InterruptOps`)
- **rvm-cap**: `CapabilityManager`, `CapabilityTable`, `ProofVerifier`, `DerivationTree`
- **rvm-witness**: `WitnessLog`, `WitnessEmitter`, chain verification functions, `WitnessSigner` trait
- **rvm-proof**: `Proof` construction, `verify()`, `verify_with_cap()`, signer/TEE abstractions
- **rvm-partition**: `PartitionManager`, `PartitionOps` trait, IPC, split/merge
- **rvm-sched**: `Scheduler<MAX_CPUS, MAX_PARTITIONS>`, `SwitchContext`, `PerCpuScheduler`
- **rvm-memory**: `BuddyAllocator<TOTAL_PAGES, BITMAP_WORDS>`, `TierManager`, `RegionManager`
- **rvm-coherence**: `CoherenceGraph<MAX_NODES, MAX_EDGES>`, `CoherenceEngine` trait, `MinCutBridge`
- **rvm-boot**: `BootTracker`, `BootSequence`, `MeasuredBootState`
- **rvm-wasm**: `AgentManager`, `MigrationTracker`, `WasmModuleInfo`
- **rvm-security**: `SecurityGate`, `SignedSecurityGate`, `PolicyRequest`, `evaluate()`/`enforce()`
- **rvm-gpu**: `GpuDevice`, `GpuContext`, `GpuKernel`, `GpuBuffer`, `GpuQueue`
- **rvm-kernel**: `Kernel` struct combining all above

**Findings:**

- **F-INTF-01**: The `Kernel` struct uses concrete types throughout (`Scheduler<8, 256>`, `WitnessLog<256>`, etc.). There are no trait-based abstractions for subsystem injection, making the kernel untestable without compiling the full subsystem stack.
- **F-INTF-02**: The `PartitionOps` trait provides an interface for partition operations, but the kernel integration uses the concrete `PartitionManager` directly rather than through the trait. The trait exists for extensibility but is unused.
- **F-INTF-03**: The signer bridge in `rvm-kernel::signer_bridge` adapts between two different `WitnessSigner` traits (one in rvm-proof for 64-byte signatures, one in rvm-witness for 8-byte signatures). This bridging is correct but adds a truncation step that could weaken signature security.

### 4.2 HAL Abstraction Layer

The HAL defines 4 traits:

```rust
trait Platform { fn cpu_count(&self) -> usize; fn total_memory(&self) -> u64; fn halt(&self) -> !; }
trait MmuOps { fn map_page(...); fn unmap_page(...); fn translate(...); fn flush_tlb(...); }
trait TimerOps { fn now_ns(&self) -> u64; fn set_deadline_ns(...); fn cancel_deadline(...); }
trait InterruptOps { fn enable(...); fn disable(...); fn acknowledge(...); fn end_of_interrupt(...); }
```

The AArch64 implementation (`rvm-hal::aarch64`) provides:
- UART (PL011) driver
- GICv2 interrupt controller
- ARM generic timer
- Stage-2 MMU (4-level page tables)
- EL2 boot entry

**Findings:**

- **F-INTF-04**: The HAL traits return `RvmResult` for all fallible operations, enabling consistent error propagation.
- **F-INTF-05**: There is no RISC-V or x86-64 HAL implementation. The code compiles for these targets in test mode only, using stub/no-op implementations. The HAL module is gated with `#[cfg(target_arch = "aarch64")]`.
- **F-INTF-06**: The AArch64 MMU implementation uses a 4-level page table with 4K pages. The page table walk is implemented in `rvm-hal::aarch64::mmu`, which uses `unsafe` for register access (documented with `// SAFETY:` comments).

### 4.3 WASM Host Context

The WASM subsystem exposes host functions through a capability-gated interface:

- `host_functions` module defines the host function table
- All host function calls require a valid `CapToken`
- WASM module validation checks magic number, version, section ordering, and size limits
- LEB128 decoding with non-canonical encoding rejection

**Findings:**

- **F-INTF-07**: WASM module validation reuses `RvmError::ProofInvalid` for parsing failures (bad magic number, invalid section ID, truncated LEB128). This conflates proof system errors with WASM parsing errors. A dedicated error variant would improve diagnostics.
- **F-INTF-08**: The WASM interpreter itself is not implemented. The crate provides validation, agent lifecycle management, and migration protocol, but does not execute WASM bytecode. This is expected for the current development phase but means the WASM host function interface is untestable end-to-end.

### 4.4 GPU Device Interface

The GPU interface follows a layered pattern:

```
GpuDevice (discovery) --> GpuContext (per-partition) --> GpuKernel (compiled) --> GpuQueue (submission)
```

**Findings:**

- **F-INTF-09**: The GPU interface is well-structured but entirely conceptual. There is no `unsafe` code in `rvm-gpu`, which means no actual hardware interaction. The `cuda-rust-wasm` dependency provides transpilation but requires runtime backends that are not linked.
- **F-INTF-10**: The `GpuMinCutConfig` and `GpuScoringConfig` in the `accel` module provide GPU-accelerated coherence computation. These are designed to integrate with `rvm-coherence` but the integration path through `rvm-kernel` is not wired up.

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-INTF-01 | Kernel uses concrete types, preventing mock-based integration testing | High | Certain |
| R-INTF-02 | Signer bridge truncation weakens 64-byte signature to 8 bytes | Medium | Certain |
| R-INTF-03 | No RISC-V or x86-64 HAL implementation beyond test stubs | Medium | Certain |
| R-INTF-04 | WASM error types conflated with proof system errors | Low | Certain |

---

## 5. PLATFORM: What the Product DEPENDS ON

### 5.1 Target Platforms

| Platform | Status | Build | Test |
|----------|--------|-------|------|
| AArch64 bare-metal (`aarch64-unknown-none`) | Primary target | Linker script (`rvm.ld`), EL2 boot | QEMU virt (Cortex-A72) |
| x86-64 Linux (`x86_64-unknown-linux-gnu`) | Test/dev only | `cargo test --workspace` | Host native |
| WASM (`wasm32-unknown-unknown`) | GPU fallback | Via `cuda-rust-wasm` | Not tested |

**Findings:**

- **F-PLAT-01**: The linker script (`rvm.ld`) places `_start` at `0x4000_0000` which is the QEMU virt machine's kernel load address. The stack is set up by the assembly stub in `main.rs`. This is correct for QEMU but may need adjustment for other SoC platforms.
- **F-PLAT-02**: The `main.rs` binary uses `#[cfg(not(test))]` to provide a trivial `main()` for host builds. This is a clean separation but means the kernel binary is never tested as a binary -- only as a library.
- **F-PLAT-03**: The release profile uses `panic = "abort"`, `lto = "fat"`, `codegen-units = 1`, `strip = true`. This is correct for bare-metal but means panics are unrecoverable. The panic handler writes to UART and enters a WFE loop.

### 5.2 Hardware Requirements

| Requirement | Specification | Source |
|-------------|--------------|--------|
| CPU | AArch64 (Cortex-A72 tested) | ADR-133 |
| Exception Level | EL2 (hypervisor) | main.rs |
| MMU | Stage-2 page tables, 4K pages | rvm-hal |
| Timer | ARM generic timer (CNTVCT_EL0) | rvm-hal |
| Interrupts | GICv2 | rvm-hal |
| UART | PL011 (0x0900_0000 on QEMU virt) | rvm-hal |
| GPU | Optional (NVIDIA CUDA, WebGPU, OpenCL, Vulkan) | ADR-144 |
| Memory | 64KB-1MB (Seed), 1-32GB (Appliance) | ADR-138 |

**Findings:**

- **F-PLAT-04**: The UART base address (`0x0900_0000`) is hardcoded for QEMU virt. Real hardware (e.g., Raspberry Pi, HiKey) uses different UART base addresses. There is no device tree parsing to discover the UART.
- **F-PLAT-05**: The GICv2 driver hardcodes distributor and CPU interface base addresses. GICv3 (used on newer ARM platforms) is not supported.
- **F-PLAT-06**: The memory hierarchy assumes 4 tiers but the Seed profile (64KB-1MB) may only have 1-2 physical tiers (no SRAM separate from DRAM, no persistent storage). The tier manager would need profile-specific configuration.

### 5.3 External Dependencies

| Dependency | Version | Purpose | no_std |
|------------|---------|---------|--------|
| `bitflags` | 2.x | CapRights bitmap | Yes |
| `spin` | 0.9 | Spinlock synchronization | Yes |
| `sha2` | 0.10 | SHA-256 hashing | Yes |
| `hmac` | 0.12 | HMAC signing | Yes |
| `subtle` | 2.6 | Constant-time comparison | Yes |
| `lz4_flex` | 0.11 | Compression for checkpoints | Yes |
| `ed25519-dalek` | 2.1 | Ed25519 signatures | Yes (optional) |
| `cuda-rust-wasm` | 0.1.7 | GPU compute transpilation | Yes (optional) |
| `criterion` | 0.5 | Benchmarking | std only (dev) |
| `proptest` | 1.5 | Property-based testing | std only (dev) |

**Findings:**

- **F-PLAT-07**: All runtime dependencies support `no_std` with `default-features = false`. This is critical for bare-metal and is correctly configured.
- **F-PLAT-08**: `proptest` is listed as a workspace dependency but there is no evidence of property-based tests in the codebase. This is an unused dependency that represents a missed testing opportunity.
- **F-PLAT-09**: The `cuda-rust-wasm` dependency (v0.1.7) is an external crate from crates.io. Being at version 0.1.x, it is pre-stable. API changes could break the GPU subsystem.

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-PLAT-01 | No real AArch64 hardware testing (only QEMU) | High | Certain |
| R-PLAT-02 | Hardcoded MMIO addresses prevent use on non-QEMU-virt platforms | Medium | Certain |
| R-PLAT-03 | GICv2-only limits to older ARM platforms | Medium | Likely |
| R-PLAT-04 | Seed profile (64KB) may not accommodate all 4 memory tiers | Medium | Possible |
| R-PLAT-05 | `proptest` imported but unused -- property-based testing opportunity missed | Low | Certain |

---

## 6. OPERATIONS: How the Product Is USED

### 6.1 Boot Sequence

The 7-phase deterministic boot sequence:

```
Phase 0: HalInit       -- Timer, MMU, interrupt controller
Phase 1: MemoryInit    -- Physical page allocator
Phase 2: CapabilityInit -- Capability table
Phase 3: WitnessInit   -- Witness trail (genesis record)
Phase 4: SchedulerInit -- Scheduler loop
Phase 5: RootPartition -- Root partition creation
Phase 6: Handoff       -- Enter scheduler loop
```

**Findings:**

- **F-OPS-01**: The boot sequence is enforced by `BootTracker` which requires phases to complete in strict order. Attempting to complete a phase out of order returns `RvmError::InternalError`. This is correct.
- **F-OPS-02**: Each boot phase emits a witness record via `emit_boot_witness()`. The genesis witness (Phase 3) establishes the hash chain root. This means Phases 0-2 are NOT witnessed (they execute before the witness subsystem is initialized). Any failure in HAL init, memory init, or capability init is invisible to the audit trail.
- **F-OPS-03**: The boot sequence in the kernel integration (`Kernel::boot()`) completes all 7 phases in a tight loop with no error recovery. If Phase 3 (WitnessInit) fails, the kernel returns an error but Phases 0-2 are already complete and cannot be rolled back. There is no incremental boot recovery.
- **F-OPS-04**: The `MeasuredBootState` computes a hash chain across boot phase measurements (SHA-256 chained). This provides attestation of the boot sequence but is only available when the `crypto-sha256` feature is enabled.

### 6.2 Runtime Operations

#### Epoch Tick

The primary runtime operation is `Kernel::tick()` which:

1. Advances the scheduler epoch (`scheduler.tick_epoch()`)
2. Ticks the coherence engine (`coherence.tick(cpu_load)`)
3. Advances the tier manager epoch and decays recency scores
4. Synchronizes coherence scores to partition objects
5. Emits a `SchedulerEpoch` witness record

**Findings:**

- **F-OPS-05**: The CPU load estimate is hardcoded to `20u8` ("a future HAL integration will read real CPU utilisation"). The coherence engine's adaptive recomputation frequency depends on this value, so it is always operating in a low-load mode.
- **F-OPS-06**: The `sync_partition_scores()` method propagates coherence scores from the engine to partition objects. If this method fails (e.g., partition was destroyed between coherence computation and sync), the error is silently swallowed.

#### Partition Operations via Kernel

The kernel provides:
- `create_partition()`: Creates a new partition with configuration
- `destroy_partition()`: Destroys a partition and reclaims resources
- `send_ipc()`: Send an IPC message between partitions
- `receive_ipc()`: Receive an IPC message
- `record_communication()`: Update coherence graph edge weights
- `apply_coherence_decision()`: Apply split/merge based on coherence engine output

**Findings:**

- **F-OPS-07**: `apply_coherence_decision()` handles `CoherenceDecision::Split` and `CoherenceDecision::Merge` but the split implementation creates a new partition and moves resources. The actual mechanism for deciding which resources move is delegated to `scored_region_assignment()` which, as noted above, uses a simplistic heuristic.
- **F-OPS-08**: There is no operation for manual partition split/merge. The only path is through the coherence engine's automatic recommendations. An administrator cannot force a split or merge.

### 6.3 Recovery Operations

The recovery model defines 4 failure classes:

| Class | Name | Response |
|-------|------|----------|
| F1 | Transient | Retry |
| F2 | Recoverable | Checkpoint restore |
| F3 | Permanent | Partition destroy |
| F4 | Catastrophic | System-wide recovery |

**Findings:**

- **F-OPS-09**: The recovery types (`FailureClass`, `RecoveryCheckpoint`, `ReconstructionReceipt`) are defined in `rvm-types` but there is no recovery engine implementation. The types exist for the checkpoint/restore mechanism in `rvm-memory::reconstruction`, but the actual failure detection and escalation logic is not implemented.
- **F-OPS-10**: The scheduler's `Recovery` mode is defined but there is no code that transitions the scheduler into Recovery mode based on failure detection. The mode switch is manual via `set_mode(SchedulerMode::Recovery)`.

### 6.4 Monitoring and Observability

- **Witness Trail**: Every privileged action emits a 64-byte witness record
- **Coherence Scores**: Per-partition coherence scores updated per epoch
- **Epoch Summaries**: Switch count and runnable count per epoch
- **MinCut Statistics**: Budget exceeded count, compute count
- **Manager Statistics**: CapabilityManager provides `ManagerStats`

**Findings:**

- **F-OPS-11**: The witness trail is the primary observability mechanism but there is no query API exposed through the kernel. The `WitnessLog` provides `get()`, `total_emitted()`, and the replay module provides `query_by_partition()`, `query_by_action_kind()`, `query_by_time_range()`, but these are not exposed through the `Kernel` struct.
- **F-OPS-12**: There is no health check or self-test operation. Once booted, the kernel provides no way to verify its own integrity (e.g., re-verify the witness chain, check capability table consistency, validate memory region non-overlap).

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-OPS-01 | Boot phases 0-2 are unwitnessed | Medium | Certain |
| R-OPS-02 | No incremental boot recovery | Medium | Possible |
| R-OPS-03 | CPU load is hardcoded, affecting coherence adaptive behavior | Medium | Certain |
| R-OPS-04 | No manual split/merge for administrator intervention | Medium | Possible |
| R-OPS-05 | Recovery engine not implemented despite type definitions | High | Certain |
| R-OPS-06 | No runtime self-test or health check operation | Medium | Likely |

---

## 7. TIME: WHEN Things Happen

### 7.1 Real-Time Constraints

| Constraint | Budget | Source | Status |
|------------|--------|--------|--------|
| Partition switch | < 10 us | ADR-132 | Stub (0 ns on host, no measurement on AArch64) |
| Mincut computation | < 50 us/epoch | ADR-132 DC-2 | Approximated by iteration count, not wall clock |
| P1 capability check | < 1 us | ADR-135 | No benchmarks |
| P2 policy validation | < 100 us | ADR-135 | No benchmarks |
| P3 deep proof | < 10 ms | ADR-135 | No benchmarks |
| Migration timeout | 100 ms | DC-7 | Enforced by `MigrationTracker` |
| Kernel execution timeout | 100 ms | DC-7 | Defined but not enforced |
| Scheduler epoch interval | 10 ms | Default | Configurable |

**Findings:**

- **F-TIME-01**: The partition switch hot path (`switch.rs`) is a stub on host builds (`elapsed_ns: 0`). On AArch64, it would execute MRS/MSR instructions, TLB invalidation, and barrier instructions. The < 10 us budget is untested on real hardware.
- **F-TIME-02**: The mincut computation uses `max_iterations` as a proxy for the 50 us time budget. There is no actual wall-clock measurement. The `MinCutBridge::new(100)` default of 100 iterations may or may not correlate with 50 us depending on graph size and hardware speed.
- **F-TIME-03**: Migration timeout enforcement uses nanosecond comparison (`current_ns - start_ns > deadline_ns`) with `saturating_sub`. This is correct for preventing underflow but relies on the caller providing accurate timestamps. If the timer drifts or the caller provides monotonically increasing but inaccurate timestamps, the timeout may fire early or late.

### 7.2 Epoch-Based Timing

The scheduler epoch is the fundamental time unit:

- Default interval: 10 ms (`epoch_interval_ns = 10_000_000`)
- Each epoch triggers: scheduler tick, coherence recomputation, tier decay, witness emission
- Edge weight decay: Configured per epoch (default 200 basis points = 2% per epoch)
- Recency decay: 200 basis points per epoch for tier manager

**Findings:**

- **F-TIME-04**: The epoch interval is a configuration constant, not enforced by a hardware timer. The kernel's `tick()` method must be called by the scheduler loop. If the loop stalls (e.g., a long-running partition switch), epochs will be delayed.
- **F-TIME-05**: Edge weight decay at 2% per epoch means a weight of 1000 decays to approximately 818 after 10 epochs (100 ms). At 100 epochs (1 second), it decays to approximately 133. This is aggressive and could prune important edges too quickly for bursty communication patterns.

### 7.3 Concurrency and Synchronization

The codebase uses `spin` locks (spinlocks) for synchronization:

- `WitnessLog` uses `spin::Mutex` for append protection
- `CapabilityManager` uses `spin::Mutex` for table protection
- `CoherenceGraph` uses no explicit synchronization (assumed single-threaded access)

**Findings:**

- **F-TIME-06**: The `CoherenceGraph` has no synchronization primitives. If the coherence engine is accessed from multiple CPUs (e.g., communication recording from different CPUs), the graph will have data races. This is a critical correctness issue for SMP configurations.
- **F-TIME-07**: Spinlocks in `no_std` bare-metal are appropriate (no OS scheduler to yield to), but there are no tests for contended access patterns. A spinlock on the witness log under high-frequency privileged operations (100K ops/sec as documented) could become a bottleneck.
- **F-TIME-08**: The `WitnessLog` ring buffer has a capacity of 262,144 records. At 100,000 privileged actions per second, this provides 2.6 seconds of storage. There is no backpressure mechanism -- once the ring wraps, records are silently overwritten. For forensic analysis after an incident, the oldest 2.6 seconds of records may have been lost.

### 7.4 Lifecycle Timing

| Event | Timing | Source |
|-------|--------|--------|
| Boot sequence | All 7 phases in sequence | Synchronous, no timeout |
| Partition creation | Within one epoch | No timeout |
| Agent migration | 7 steps, 100 ms deadline | MigrationTracker |
| Device lease expiry | Configurable (expiry_ns field) | DeviceLease struct |
| Scheduler mode switch | Immediate | No transition delay |

**Findings:**

- **F-TIME-09**: The boot sequence has no timeout. If any phase hangs (e.g., HAL init waiting for a hardware response that never comes), the system is permanently stuck. The panic handler enters a WFE loop, but there is no watchdog timer integration.
- **F-TIME-10**: Device lease expiry is defined in the `DeviceLease` struct (`expiry_ns` field, 0 = no expiry) but there is no visible lease expiry enforcement mechanism. The `DeviceLeaseManager` tracks leases but does not appear to check expiry on each access.

**Risk Areas:**

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R-TIME-01 | Partition switch time budget untested on real hardware | Critical | Certain |
| R-TIME-02 | Mincut time budget approximated by iteration count, not wall clock | High | Certain |
| R-TIME-03 | CoherenceGraph has no synchronization for SMP access | Critical | Likely |
| R-TIME-04 | Witness log overflow under sustained high-throughput operations | High | Likely |
| R-TIME-05 | Boot sequence has no timeout/watchdog | Medium | Possible |
| R-TIME-06 | Device lease expiry not enforced at runtime | Medium | Likely |
| R-TIME-07 | Edge weight decay too aggressive for bursty communication | Medium | Possible |

---

## Cross-Dimensional Interactions and Risks

### Interaction 1: Structure x Time -- Coherence Graph Concurrency

The `CoherenceGraph` (Structure) is accessed by the coherence engine during epoch ticks (Time) and by the kernel during `record_communication()` calls (potentially from multiple CPUs). The graph has no synchronization (Time), and its O(1) hash-based lookup (Structure) has collision issues (Data). Under SMP operation, this combination creates:

- **Race condition**: Two CPUs calling `record_communication()` simultaneously can corrupt the adjacency list.
- **Hash collision during concurrent access**: The linear-scan fallback in `find_node()` is not thread-safe.
- **Risk level**: Critical (P0) for SMP configurations.

### Interaction 2: Function x Data -- Security Gate Actor Tracking

The security gate (Function) emits witness records (Data) with `actor_partition_id = 0` hardcoded. Meanwhile, the capability system (Function) provides `CapToken` with an `id` field that could identify the actor. This means:

- The audit trail records WHAT happened but not WHO did it.
- Forensic analysis of a security breach cannot trace actions to specific partitions.
- **Risk level**: High (P1) for any deployment with multiple partitions.

### Interaction 3: Platform x Operations -- Boot Without Witness

Boot phases 0-2 (Platform) execute before the witness subsystem (Operations) is initialized. If a hardware initialization failure occurs during Phase 0 (HAL init) or Phase 1 (memory init), there is no audit record of:

- What hardware was detected
- What memory regions were configured
- What capabilities were created

A malicious firmware could manipulate Phase 0-2 operations without leaving any trace in the witness trail.

- **Risk level**: High (P1) for security-critical deployments.

### Interaction 4: Function x Time -- Scheduler Priority Scale Mismatch

The scheduler (Function) combines `deadline_urgency` (u16, max 65535) with `cut_pressure` (u32, potentially billions) using saturating addition. Under high cut pressure (Time-driven by epoch-accumulated communication), the deadline urgency signal becomes insignificant. This means:

- A partition with an imminent deadline but low coherence cut pressure will be deprioritized.
- Real-time guarantees (Reflex mode) depend on the coherence engine being disabled or in degraded mode.
- **Risk level**: Medium (P2) -- affects real-time scheduling correctness.

### Interaction 5: Data x Interfaces -- Signer Bridge Truncation

The kernel's signer bridge (Interfaces) truncates 64-byte proof-crate signatures to 8 bytes for the witness record's `aux` field (Data). The `SignedSecurityGate` (Function) then verifies these 8-byte signatures by reconstructing minimal records. This creates:

- 56 bytes of signature entropy lost in truncation.
- Verification uses reconstructed records with most fields zeroed, not matching the original signed content.
- A sophisticated attacker could forge the 8-byte truncated signature more easily than the full 64-byte version.
- **Risk level**: Medium (P2) for environments requiring cryptographic witness integrity.

---

## Prioritized Recommendations

### P0 -- Critical (Address Before Any Deployment)

1. **Add SMP synchronization to CoherenceGraph** (R-TIME-03)
   - Wrap the graph in a `spin::RwLock` or redesign with per-CPU local graphs and periodic merging.
   - Test with multiple threads calling `add_edge`, `remove_node`, and `decay_weights` concurrently.

2. **Validate partition switch timing on real AArch64 hardware** (R-TIME-01)
   - Run benchmark on QEMU virt with `-icount` for deterministic cycle counting.
   - Run on real Cortex-A72 hardware (e.g., Raspberry Pi 4) with performance counters.
   - Fail-fast if switch exceeds 10 us budget.

3. **Create end-to-end integration test for boot-to-tick pipeline** (R-STRUCT-01)
   - Test: `Kernel::with_defaults() -> boot() -> create_partition() -> enqueue() -> tick() -> coherence_score()`.
   - Cover: witness record emission across all phases, coherence score propagation, epoch advancement.

4. **Fix SignedSecurityGate record reconstruction** (F-FUNC-14)
   - The `verify_chain_signatures()` method must reconstruct complete records matching what the signer originally signed, not minimal 3-field records.
   - Add a test that signs a full record and verifies the chain.

### P1 -- High (Address Before Beta)

5. **Add epoch staleness check to SecurityGate** (F-FUNC-16)
   - The `GateRequest`-based path does not check capability epoch. Add `current_epoch` field to `GateRequest` and enforce freshness.

6. **Implement witness log drain mechanism** (R-DATA-01)
   - Add a `drain()` or `export()` method to `WitnessLog` that bulk-reads and optionally persists records before they are overwritten.
   - Add a high-water-mark callback that fires when the ring is 80% full.

7. **Populate actor_partition_id in security gate witness records** (F-FUNC-15)
   - Add `actor_partition_id` to `GateRequest` and propagate it to emitted witness records.

8. **Address CoherenceGraph hash collision** (F-FUNC-10)
   - The `id_to_node` lookup using `partition_id % 256` will collide for IDs differing by 256. Use open addressing or a proper hash function.

9. **Guard against wrong-order free in buddy allocator** (F-FUNC-07)
   - Track the allocated order per block (e.g., in a parallel metadata array) and reject `free_pages()` calls with mismatched orders.

10. **Wire up regions_overlap_host check in kernel** (F-FUNC-08)
    - The host-physical overlap check exists in `rvm-memory` but may not be called during partition creation. Verify and add integration test.

### P2 -- Medium (Address Before GA)

11. **Implement wall-clock mincut budget enforcement** (F-TIME-02)
    - Replace iteration-count proxy with actual timer reads from `TimerOps::now_ns()`.
    - Requires HAL integration in the mincut bridge.

12. **Add property-based tests using proptest** (F-PLAT-08)
    - The dependency exists but is unused. Priority targets:
      - CoherenceScore arithmetic (saturation, clamping, ordering)
      - Buddy allocator alloc/free sequences
      - WitnessRecord hash chain integrity under arbitrary record sequences
      - Capability derivation depth and rights attenuation

13. **Implement recovery engine** (R-OPS-05)
    - The failure class types exist but no detection or escalation logic. Implement F1 (retry) and F2 (checkpoint restore) handlers.

14. **Fix priority scale mismatch in scheduler** (F-FUNC-04)
    - Normalize `deadline_urgency` and `cut_pressure` to the same scale before combination.
    - Consider a weighted formula: `priority = alpha * normalized_deadline + beta * normalized_pressure`.

15. **Separate WASM parsing errors from proof errors** (F-INTF-07)
    - Add `RvmError::WasmValidationFailed` or similar variant for module parsing failures.

16. **Add device lease expiry enforcement** (R-TIME-06)
    - Check `expiry_ns > 0 && current_ns > expiry_ns` in lease access methods.
    - Add expired-lease cleanup to the epoch tick.

### P3 -- Low (Address in Maintenance)

17. **Implement Phi-to-coherence transfer function** (F-FUNC-11)
    - Replace the stub linear mapping with a calibrated function based on IIT theory.

18. **Add RISC-V HAL implementation** (F-INTF-05)
    - At minimum, provide SBI-based timer, interrupt, and UART for testing on RISC-V QEMU.

19. **Expose witness query API through Kernel** (F-OPS-11)
    - Wrap `query_by_partition()`, `query_by_action_kind()`, and `query_by_time_range()` in the Kernel struct.

20. **Add runtime self-test / health check** (R-OPS-06)
    - Verify witness chain integrity, capability table consistency, memory region non-overlap.

21. **Parameterize edge weight decay rate** (F-TIME-05)
    - Allow per-partition or per-edge decay rates for workloads with different communication patterns.

22. **Add boot timeout / watchdog integration** (R-TIME-05)
    - Set a hardware watchdog timer at boot entry; pet it at each phase completion.

---

## Test Coverage Mapping by SFDIPOT Dimension

| Dimension | Unit Tests | Integration Tests | Property Tests | Hardware Tests | Human Exploration |
|-----------|-----------|-------------------|----------------|----------------|-------------------|
| **Structure** | 895 across 69 files | None identified | None (proptest unused) | None | Recommended: Feature flag matrix |
| **Function** | High for core logic, medium for security | Low | None | None | Recommended: Adversarial capability forging |
| **Data** | High for boundary values | Low | None | None | Recommended: WitnessRecord field corruption |
| **Interfaces** | Medium for API contracts | None for cross-crate | None | None | Recommended: Signer bridge verification |
| **Platform** | Host-only via test stubs | None on AArch64 QEMU | None | None (QEMU untested in CI) | Recommended: Real hardware boot |
| **Operations** | Medium for lifecycle flows | None for full boot-to-tick | None | None | Recommended: Boot failure injection |
| **Time** | Low for timing constraints | None for concurrency | None | None | Recommended: Spinlock contention profiling |

---

## Appendix A: Crate-Level Test Distribution

| Crate | #[test] Count | Key Test Areas |
|-------|--------------|----------------|
| rvm-types | 0 | (Pure data types, no behavior to test) |
| rvm-hal | 18 | MMU page tables, GIC, timer |
| rvm-cap | 45 | Derivation, grant, revoke, table, verify, manager |
| rvm-witness | 45 | Emit, hash, log, replay, signer |
| rvm-proof | 139 | Engine, context, policy, constant_time, signer, TEE |
| rvm-partition | 89 | Lifecycle, manager, split, merge, IPC, device, comm_edge |
| rvm-sched | 51 | Scheduler, switch, epoch, priority, modes, degraded, SMP |
| rvm-memory | 110 | Allocator, region, tier, reconstruction |
| rvm-coherence | 73 | Graph, scoring, pressure, mincut, adaptive, engine, bridge |
| rvm-boot | 26 | Entry, hal_init, measured, sequence |
| rvm-wasm | 38 | Agent, migration, quota, host_functions, module validation |
| rvm-security | 54 | Gate, validation, attestation, budget |
| rvm-gpu | 134 | Context, buffer, queue, kernel, accel, budget, device, error |
| rvm-kernel | 67 | Kernel integration, signer bridge |
| **Total** | **895** | |

## Appendix B: ADR Cross-Reference

| ADR | Title | Crate(s) | Coverage Assessment |
|-----|-------|----------|---------------------|
| ADR-132 | Core Architecture | All | Partial (DC-2 time budget not enforced with wall clock) |
| ADR-133 | Partition Object Model | rvm-partition | Good (lifecycle, split/merge tested) |
| ADR-134 | Witness Schema | rvm-types, rvm-witness | Good (64-byte record, hash chain) |
| ADR-135 | Proof Verifier | rvm-cap, rvm-proof | Partial (ZK tier not implemented) |
| ADR-136 | Memory Hierarchy | rvm-memory | Good (4-tier, buddy allocator) |
| ADR-137 | Boot Sequence | rvm-boot | Good (7-phase, measured boot) |
| ADR-138 | Seed Profile | rvm-types, rvm-memory | Low (no profile-specific testing) |
| ADR-139 | Appliance Deployment | rvm-coherence | Partial (Phi mapping is stub) |
| ADR-140 | Agent Runtime | rvm-wasm | Good (lifecycle, migration) |
| ADR-141 | Coherence Engine | rvm-coherence | Partial (ruvector integration pending) |
| ADR-142 | TEE Verification | rvm-proof | Good (software TEE provider) |
| ADR-143 | Release Pipeline | N/A | Not implemented |
| ADR-144 | GPU Compute | rvm-gpu | Partial (all stubs, no hardware) |
| ADR-145 | IPC Protocol | rvm-partition | Good (message queues) |
| ADR-146 | SMP Scheduling | rvm-sched | Partial (SMP coordinator, no real SMP test) |
| ADR-147 | HAL | rvm-hal | Partial (AArch64 only, no RISC-V/x86) |
| ADR-148 | Error Model | rvm-types | Good (30 error variants, Display impl) |
| ADR-150 | Device Lease | rvm-partition | Partial (expiry not enforced) |
| ADR-152 | GPU MinCut | rvm-gpu | Partial (correctness stubs only) |

---

*Analysis performed using James Bach's Heuristic Test Strategy Model (HTSM) Product Factors framework (SFDIPOT). All findings are based on direct source code inspection of 32,543 lines of Rust across 14 crates.*
