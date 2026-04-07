# QX-04: Quality Experience Analysis -- RVM Developer Experience

**Date**: 2026-04-07
**Analyst**: QX Partner (Agentic QE v3)
**Scope**: All 14 crates in `/workspaces/rvm/crates/`
**Target Users**: Systems developers building agent runtimes, embedded developers, security engineers
**Method**: Full public API surface inspection of every `lib.rs`, key submodules, error types, doc comments, tests, and user guide

---

## Executive Summary

RVM presents a remarkably well-engineered API surface for a bare-metal `no_std` microhypervisor. The codebase achieves a rare combination: principled systems-level rigor (zero `unsafe` in type-safe crates, `forbid(unsafe_code)`, `deny(missing_docs)`) paired with genuine developer ergonomics (builder patterns, const constructors, sensible defaults, comprehensive doc tables). The 15-chapter user guide and 945-test suite demonstrate serious commitment to developer experience.

**Overall QX Score: 82/100 (B+)**

**Top Strengths**:
1. Exceptional documentation quality -- every `lib.rs` opens with a design-context table mapping types to their ADR specifications. The user guide is publication-quality.
2. Consistent error architecture -- a unified `RvmError` with 34 domain-specific variants, layered per-crate error types with `From` conversions, and `RvmResult<T>` alias used uniformly.
3. Strong API consistency across 14 crates -- identical lint directives, identical `no_std`/`alloc`/`std` feature flag patterns, identical re-export conventions.
4. Thoughtful `no_std` ergonomics -- const-generic capacity parameters, sentinel-based arrays avoiding `Option` overhead, power-of-two const assertions.

**Top Pain Points**:
1. Kernel API requires manual witness record construction -- 8 lines of boilerplate per operation instead of a helper method or macro.
2. Error variants sometimes overloaded across domains (e.g., `InvalidPartitionState` used for boot errors, agent state errors, migration errors, GPU context errors).
3. No builder pattern for the most common developer task: configuring and creating a partition with capabilities, scheduling, and IPC in one fluent chain.
4. GPU integration requires understanding 3 crate boundaries (`rvm-gpu`, `rvm-wasm` host functions, `rvm-kernel`) with no unified "GPU quickstart" path.

---

## API Ergonomics Scorecard

| Crate | Naming | Patterns | Errors | Docs | Defaults | Score |
|-------|--------|----------|--------|------|----------|-------|
| rvm-types | 95 | 90 | 92 | 95 | 90 | **92** |
| rvm-hal | 90 | 88 | 85 | 90 | 80 | **87** |
| rvm-cap | 92 | 90 | 95 | 90 | 88 | **91** |
| rvm-witness | 88 | 85 | 80 | 92 | 85 | **86** |
| rvm-proof | 85 | 82 | 82 | 90 | 78 | **83** |
| rvm-partition | 90 | 85 | 85 | 90 | 85 | **87** |
| rvm-sched | 88 | 85 | 78 | 88 | 82 | **84** |
| rvm-memory | 90 | 88 | 88 | 92 | 85 | **89** |
| rvm-coherence | 88 | 85 | 80 | 88 | 82 | **85** |
| rvm-boot | 90 | 88 | 75 | 85 | 88 | **85** |
| rvm-wasm | 85 | 82 | 72 | 85 | 80 | **81** |
| rvm-security | 92 | 90 | 88 | 90 | 85 | **89** |
| rvm-kernel | 85 | 78 | 80 | 85 | 82 | **82** |
| rvm-gpu | 90 | 85 | 90 | 90 | 82 | **87** |

**Scoring criteria**: Naming (convention clarity), Patterns (builder/fluent/const), Errors (specificity, recoverability), Docs (doc comments, examples), Defaults (sensible out-of-box behavior).

---

## Detailed Crate-by-Crate Analysis

### 1. rvm-types (Score: 92/100)

**Strengths**:
- Exemplary module documentation with ADR-linked tables mapping types to purposes.
- All identifier types are `Copy + Clone + Eq + Hash`-compatible newtypes -- exactly right for kernel IDs.
- `CoherenceScore` and `CutPressure` use fixed-point basis-point representation (0..10000) with named constants (`MAX`, `MIN`, `DEFAULT_THRESHOLD`). This is a strong DX decision: developers never need to guess units.
- `CapRights` uses `bitflags!` with self-documenting names (`READ`, `WRITE`, `EXECUTE`, `GRANT`, `REVOKE`, `PROVE`, `GRANT_ONCE`).
- Error type has 34 variants grouped by subsystem with clear section comments.

**Pain Points**:
- `RvmError::Display` messages are terse ("proof invalid", "internal error"). For a developer debugging at 3 AM, "proof invalid" does not tell them *what* was invalid or *what to do about it*. Compare with `CapError::StaleHandle` which adds "(generation mismatch)" -- this pattern should be universal.
- No `std::error::Error` implementation (understandable for `no_std`, but should be gated on `feature = "std"`).
- `RvmError::InternalError` is a catch-all. 7+ call sites map to it, making debugging harder. Consider splitting into `InternalSlotExhausted`, `InternalIndexOutOfRange`, etc.

**Recommendation**: Add `#[cfg(feature = "std")] impl std::error::Error for RvmError {}` and enrich `Display` messages with remediation hints (e.g., "partition limit exceeded -- maximum is 256, reduce active partitions or destroy unused ones").

### 2. rvm-hal (Score: 87/100)

**Strengths**:
- Clean trait-based abstraction: `Platform`, `MmuOps`, `TimerOps`, `InterruptOps` are exactly the right granularity.
- Every trait method that can fail documents its `# Errors` section.
- `deny(unsafe_code)` at trait level with `allow(unsafe_code)` only for arch-specific modules -- correct boundary placement.

**Pain Points**:
- 12 clippy `allow` directives suppressed in the module header. Some (`clippy::missing_errors_doc`) suppress useful warnings. These should be scoped to specific functions, not the entire crate.
- No example code showing how to implement `Platform` for a new target. This is the single most important developer journey for the HAL.
- `Platform::halt(&self) -> !` has no error path -- but what if halt is not available on the platform? Consider a fallback behavior.

**Recommendation**: Add a doc-test example in the `Platform` trait showing a minimal stub implementation.

### 3. rvm-cap (Score: 91/100)

**Strengths**:
- Layered error types: `CapError` (10 variants for table ops), `ProofError` (6 variants for verification), both with `From<CapError> for RvmError`. This is textbook error design.
- Builder pattern for `CapManagerConfig::new().with_max_depth(2)` -- fluent and const-compatible.
- `CapabilityManager` uses const generics (`CapabilityManager<const N: usize = DEFAULT_CAP_TABLE_CAPACITY>`) with a default, so `CapabilityManager::<64>::with_defaults()` works ergonomically.
- Clear checked vs unchecked variants: `create_root_capability()` vs `create_root_capability_checked()`. Good pattern.
- `ProofError::PolicyViolation` deliberately omits specifics to prevent timing side-channels. This is documented and correct.

**Pain Points**:
- The `grant()` method takes 5 parameters (`source_index`, `source_generation`, `requested_rights`, `badge`, `target_owner`). This is a high cognitive load. A `GrantRequest` struct or builder would reduce errors.
- Return type `(u32, u32)` for `(index, generation)` is opaque. A named `CapHandle { index: u32, generation: u32 }` would prevent parameter-swap bugs.

**Recommendation**: Introduce `CapHandle` newtype for the `(index, generation)` pair. This single change would eliminate a class of bugs where callers swap index and generation.

### 4. rvm-witness (Score: 86/100)

**Strengths**:
- The core invariant is stated clearly at the top: "No witness, no mutation."
- Byte-level record format is documented in a table with offsets, sizes, and field names.
- Replay/query functions (`query_by_partition`, `query_by_action_kind`, `query_by_time_range`) provide essential forensic capability.
- `WitnessLog::append()` is zero-copy, lock-free ring buffer -- correct for hot-path usage.

**Pain Points**:
- Constructing a `WitnessRecord` requires manual field-by-field assignment. Every call site in rvm-kernel does 6-8 lines of boilerplate:
  ```rust
  let mut record = WitnessRecord::zeroed();
  record.action_kind = ActionKind::PartitionCreate as u8;
  record.proof_tier = 1;
  record.actor_partition_id = id.as_u32();
  record.target_object_id = target.as_u32() as u64;
  self.witness_log.append(record);
  ```
  This should be a builder or macro: `WitnessRecord::new(ActionKind::PartitionCreate).actor(id).target(target).tier(1)`.
- `action_kind` is stored as `u8` but `ActionKind` is an enum -- the cast `as u8` is manual at every call site.

**Recommendation**: Add `WitnessRecord::builder()` or `WitnessRecord::for_action(kind: ActionKind)` that handles the `as u8` cast and provides typed setters. This would eliminate 40+ lines of boilerplate across rvm-kernel alone.

### 5. rvm-proof (Score: 83/100)

**Strengths**:
- Clear tier documentation with cost model (Hash=O(1), Witness=O(n), Zk=expensive).
- Builder pattern for proof context in the `context` module.
- Explicit "not yet implemented" for ZK tier with `Err(RvmError::Unsupported)` -- honest and safe.
- `Proof::hash_proof()` handles truncation gracefully with clear documentation.

**Pain Points**:
- `verify()` and `verify_with_cap()` live at crate root with no module prefix. For a crate with 8 modules, this can confuse discoverability -- developers may not find them.
- The `Proof` struct has `data: [u8; 64]` with `data_len: u8` -- this is a manual length-prefixed buffer. Consider a newtype `ProofData` that encapsulates the truncation logic.
- Multiple signer types (`HmacSha256WitnessSigner`, `DualHmacSigner`, `Ed25519WitnessSigner`, `NullSigner`) are feature-gated but exported flat at the crate root. A `signers` module namespace would aid discoverability.

**Recommendation**: Group signer exports under `rvm_proof::signers::*` and add a "Which signer should I use?" decision table in the module docs.

### 6. rvm-partition (Score: 87/100)

**Strengths**:
- Clear module separation: `lifecycle`, `manager`, `split`, `merge`, `ipc`, `cap_table`, `comm_edge`, `device`.
- `PartitionManager` has O(1) lookup via direct index with linear-scan fallback -- pragmatic optimization.
- `merge_preconditions_met()` and `merge_preconditions_full()` separate quick-check from detailed validation -- good for callers with different needs.
- `valid_transition()` function encodes the state machine -- prevents invalid lifecycle transitions at compile-time-adjacent checking.

**Pain Points**:
- `PartitionManager::create()` takes raw primitives (`partition_type: PartitionType, vcpu_count: u16, epoch: u32`) instead of a config struct. Adding fields later is a breaking change.
- `PartitionOps` trait and `SplitConfig` are exported from `ops` module but the relationship to `PartitionManager` is not documented.
- `PartitionState` enum has 5 variants but there is no ASCII art state machine diagram in the docs showing valid transitions.

**Recommendation**: Add a state transition diagram in the `lifecycle` module docs. Replace raw parameters in `create()` with `PartitionConfig` struct.

### 7. rvm-sched (Score: 84/100)

**Strengths**:
- `Scheduler<const MAX_CPUS: usize, const MAX_PARTITIONS: usize>` -- const-generic dimensions are ergonomic and enable compile-time sizing.
- Binary max-heap for O(log n) insertion/extraction -- well-documented performance choice.
- `RunQueueEntry::EMPTY` sentinel avoids `Option<>` overhead -- documented rationale.
- Clean `enqueue` / `switch_next` / `tick_epoch` lifecycle.

**Pain Points**:
- `enqueue()` returns `bool` (success/failure) instead of `RvmResult<()>`. This is inconsistent with every other crate that uses `RvmResult`. The kernel wrapper `enqueue_partition()` has to manually convert: `if !self.scheduler.enqueue(...) { return Err(...) }`.
- `Scheduler::new()` defaults to `Flow` mode with no configuration. There is no builder or config struct for initial parameters.
- No method to inspect what partition is currently running on a CPU without going through `per_cpu()`.

**Recommendation**: Change `enqueue()` return type to `RvmResult<()>` for consistency. Add `Scheduler::current_partition(cpu: usize) -> Option<PartitionId>` convenience method.

### 8. rvm-memory (Score: 89/100)

**Strengths**:
- Four-tier memory model is clearly documented with an ASCII table in the module docs.
- `MemoryPermissions` with named constants (`READ_ONLY`, `READ_WRITE`, `READ_EXECUTE`) -- zero cognitive overhead.
- `validate_region()` returns specific errors for each failure mode (`AlignmentError`, `ResourceLimitExceeded`, `Unsupported`).
- `regions_overlap()` and `regions_overlap_host()` have detailed doc comments explaining the isolation semantics.
- Legacy `MemoryRegion` is explicitly documented as deprecated in favor of `OwnedRegion`.

**Pain Points**:
- `MemoryRegion` uses `MemoryPermissions` with 3 boolean fields. A `bitflags!`-based permission would be more compact and consistent with `CapRights`.
- `validate_region()` returns `RvmError::Unsupported` for zero-permission regions. This is semantically wrong -- "unsupported" implies the feature does not exist, not that the input is invalid. `InvalidConfiguration` or similar would be clearer.
- The `allocator::BuddyAllocator` is not re-exported with any usage examples.

**Recommendation**: Use `RvmError::ResourceLimitExceeded` or add a `RvmError::InvalidConfiguration` variant for zero-permission regions.

### 9. rvm-coherence (Score: 85/100)

**Strengths**:
- `CoherenceEngine` lifecycle is documented as a 5-step flow in the module docs: `add_partition` -> `record_communication` -> `tick` -> `score` -> `recommend`. This is excellent DX.
- Pluggable backends via generics (`CoherenceEngine<MCB: MinCutBackend, CB: CoherenceBackend>`) with `DefaultCoherenceEngine` type alias.
- `EmaFilter` with basis-point arithmetic avoids floating point -- correct for `no_std`.
- `CoherenceDecision` enum is self-documenting: `NoAction`, `SplitRecommended { partition, pressure }`, `MergeRecommended { a, b, mutual_coherence }`.

**Pain Points**:
- The `graph` module has `NodeIdx` and `EdgeIdx` as `u16` type aliases rather than newtypes. This allows mixing them up silently.
- `GraphError` is a separate error type from `RvmError` with manual mapping at every call site. Some mappings are questionable (e.g., `GraphError::DuplicateNode` -> `RvmError::InvalidPartitionState`).
- The `adaptive` module's recomputation frequency logic is not documented with examples showing when it skips.

**Recommendation**: Make `NodeIdx` and `EdgeIdx` newtypes to prevent mixing. Document the adaptive frequency behavior with a table showing CPU load vs. recomputation interval.

### 10. rvm-boot (Score: 85/100)

**Strengths**:
- Deterministic phased boot with clear ASCII phase diagrams (both ADR-137 and ADR-140 variants).
- `BootPhase::name()` returns `&'static str` -- useful for logging.
- `BootTracker` prevents out-of-order phase execution.
- `MeasuredBootState` implements measured boot hash chains.

**Pain Points**:
- `BootTracker::complete_phase()` returns `RvmError::InternalError` for out-of-order phases and `RvmError::Unsupported` for already-complete boot. Neither is semantically descriptive. A `RvmError::BootPhaseError` or similar would be clearer.
- Two competing phase numbering schemes (ADR-137 7-phase and ADR-140 legacy) documented in the same crate without clear guidance on which to use.
- No `BootTracker::reset()` for testing scenarios.

**Recommendation**: Add `RvmError::BootPhaseOutOfOrder` and `RvmError::BootAlreadyComplete` for specific error recovery.

### 11. rvm-wasm (Score: 81/100)

**Strengths**:
- Agent lifecycle state machine (`Initializing` -> `Running` -> `Suspended` -> `Migrating` -> `Hibernated` -> `Terminated`) is well-defined.
- `AgentManager<const MAX: usize>` with const-generic capacity.
- `HostContext` trait with default stub implementations -- enables incremental development.
- `HostFunction::required_rights()` maps each host call to its capability requirements -- security by construction.
- `dispatch_host_call()` performs capability checking before dispatch.
- WASM module validation (`validate_module`) is thorough: magic number, version, section ordering, LEB128 decoding, size limits.

**Pain Points**:
- `AgentManager::terminate()` returns `RvmError::PartitionNotFound` for a missing agent -- this is the wrong error type. It should be something like `RvmError::AgentNotFound` or at minimum `RvmError::InternalError`.
- `WasmModuleState` and `AgentState` overlap conceptually but are distinct types with no conversion. A developer managing both must track two state machines.
- `validate_module()` returns `RvmError::ProofInvalid` for WASM parsing errors. This is semantically incorrect -- a malformed WASM module is not a "proof" failure.
- `HostCallArgs` uses `arg0`, `arg1`, `arg2` -- opaque positional arguments. Each `HostFunction` interprets them differently, but this is documented only in doc comments, not enforced by the type system.
- `MigrationTracker::advance()` takes a `WitnessLog` reference on every step -- the tracker does not own or cache it, creating a threading ergonomics issue.

**Recommendation**: Add `RvmError::WasmValidationFailed` for module parsing errors. Introduce typed argument structs per host function (e.g., `SendArgs { target: PartitionId, length: u64 }`).

### 12. rvm-security (Score: 89/100)

**Strengths**:
- `PolicyRequest` struct bundles all security context in a single, well-documented type with lifetime annotations.
- Clear three-stage gate pipeline: capability check -> proof verification -> witness logging.
- `evaluate()` returns `PolicyDecision::Allow | Deny(RvmError)` -- composable and testable.
- `enforce()` convenience function converts to `RvmResult<()>` for the common case.
- Epoch freshness check (`token.epoch() != epoch`) catches stale capabilities.

**Pain Points**:
- `SecurityGate` lives in the `gate` module but is the primary developer-facing type. Consider re-exporting at crate root (it already is, which is correct).
- No rate-limiting or audit-throttle mechanism in the security gate.

### 13. rvm-kernel (Score: 82/100)

**Strengths**:
- ASCII architecture diagram in the module docs showing the dependency tree.
- `Kernel` struct unifies all subsystems behind a single facade -- correct integration pattern.
- `Kernel::with_defaults()` provides zero-config entry point.
- `Kernel::boot()` drives the full 7-phase boot sequence.
- `apply_decision()` method closes the coherence feedback loop: `tick()` -> decision -> `apply_decision()`.
- Re-exports all crates as named modules (`kernel::types`, `kernel::partition`, etc.) -- good for unified access.

**Pain Points**:
- `Kernel` struct hardcodes 6 const-generic dimensions (`DEFAULT_MAX_CPUS=8`, `DEFAULT_WITNESS_CAPACITY=256`, etc.). These are not configurable at construction time. A `KernelBuilder` with configurable dimensions would be more flexible.
- 40+ lines of witness record boilerplate across `create_partition`, `destroy_partition`, `execute_split`, `execute_merge`, `create_channel`, `ipc_send`, `destroy_channel`, `promote_region`, `demote_region`, etc. Each follows the identical pattern of constructing a `WitnessRecord::zeroed()`, setting fields, and calling `append()`.
- `record_communication()` maps all coherence errors to `RvmError::InternalError`, losing the original error context.
- `create_partition()` always creates `PartitionType::Agent` regardless of what the caller might want. The `PartitionConfig` parameter is partially ignored.
- Signer bridge module is complex (proof-signer 64-byte -> witness-signer 8-byte truncation) but has no usage example.

**Recommendation**: Extract witness emission into a `fn emit_witness(&self, action: ActionKind, actor: PartitionId, target: impl Into<u64>)` helper. Add `KernelBuilder` for dimension configuration.

### 14. rvm-gpu (Score: 87/100)

**Strengths**:
- Excellent module documentation mapping feature flags to GPU backends and hardware profiles.
- `GpuError` (13 variants) with domain-specific names (`IommuViolation`, `KernelTimeout`, `BudgetExceeded`).
- `GpuContext` tracks per-partition GPU state with budget enforcement.
- `GpuBudget` provides atomic check-and-record semantics (same pattern as `rvm-wasm::QuotaTracker`).
- `GpuTier` enum with clear hardware-class mapping.

**Pain Points**:
- `From<GpuError> for RvmError` maps `KernelTimeout` and `IommuViolation` to `RvmError::InternalError`. An IOMMU violation is a security event, not an internal error. It should map to `RvmError::InsufficientCapability` or a new security-specific variant.
- `GpuContext::new()` starts in `Initializing` state -- caller must manually transition to `Ready`. No validation prevents operations on non-ready contexts at the type level.
- No integration example showing end-to-end GPU kernel launch from a WASM agent.

**Recommendation**: Map `GpuError::IommuViolation` to a security-specific error. Add typestate pattern for `GpuContext` (`GpuContext<Initializing>` vs `GpuContext<Ready>`).

---

## Developer Journey Maps

### Journey 1: Creating and Managing Partitions

```
Developer Goal: Create a partition, assign capabilities, schedule it, run an agent

Step 1: Boot the kernel
  kernel = Kernel::with_defaults()        // Easy: zero-config
  kernel.boot()?                          // Easy: single call
  Pain: None. Excellent entry point.

Step 2: Create a partition
  config = PartitionConfig { vcpu_count: 1, ... }
  id = kernel.create_partition(&config)?  // Easy
  Pain: PartitionConfig is partially ignored (type always Agent).
        Developer cannot create Root or Infrastructure partitions.

Step 3: Assign capabilities
  cap_mgr = kernel.cap_manager_mut()
  (idx, gen) = cap_mgr.create_root_capability(
      CapType::Partition, rights, badge, owner
  )?                                      // Medium: 4 params
  Pain: Returns (u32, u32) tuple -- must remember order.
        No connection between partition creation and cap creation.
        Developer must manually wire the pieces together.

Step 4: Schedule the partition
  kernel.enqueue_partition(cpu, id, urgency)?  // Easy
  kernel.switch_next(cpu)?                      // Easy
  Pain: Must manually calculate deadline_urgency.
        No helper for "schedule this partition fairly".

Step 5: Send IPC
  edge = kernel.create_channel(from, to)?      // Easy
  msg = IpcMessage { sender, receiver, edge_id, payload_len, msg_type,
                     sequence, capability_hash } // PAIN: 7 fields, manual
  kernel.ipc_send(edge, msg, caller_id)?
  Pain: IpcMessage construction requires manual sequence numbering
        and capability hash computation.

Friction Rating: MEDIUM
Key Bottleneck: Steps 3 and 5 require too much manual assembly.
```

### Journey 2: Setting Up Capability Chains

```
Developer Goal: Create a root capability, derive child caps, verify proof chain

Step 1: Create root cap
  let mgr = CapabilityManager::<64>::with_defaults();
  let (idx, gen) = mgr.create_root_capability(
      CapType::Region, all_rights, 0, owner
  )?;
  Pain: Low. Clean API.

Step 2: Grant derived capability
  let (child_idx, child_gen) = mgr.grant(
      idx, gen, CapRights::READ, 42, target
  )?;
  Pain: Must track (index, generation) pairs manually.
        Easy to accidentally swap them.

Step 3: Verify P1/P2/P3
  mgr.verify_p1(idx, gen, CapRights::READ)?;     // Easy
  mgr.verify_p2(idx, gen, &PolicyContext { .. })?; // Medium
  mgr.verify_p3(idx, gen, max_depth)?;            // Easy
  Pain: PolicyContext construction not well-documented.

Step 4: Revoke and propagate
  let result = mgr.revoke(idx, gen)?;
  // result.revoked_count tells how many were invalidated
  Pain: Low. Clean revocation semantics.

Friction Rating: LOW-MEDIUM
Key Bottleneck: (index, generation) pair tracking.
```

### Journey 3: Running WASM Agents

```
Developer Goal: Load a WASM module, spawn an agent, manage lifecycle

Step 1: Validate WASM module
  let result = rvm_wasm::validate_module(bytes)?;
  Pain: Returns RvmError::ProofInvalid for parse errors.
        Developer wonders: "What proof? I'm loading WASM."

Step 2: Spawn agent
  let mgr = AgentManager::<8>::new();
  let config = AgentConfig { badge: 42, partition_id, max_memory_pages: 16 };
  let id = mgr.spawn(&config, &witness_log)?;
  Pain: Must provide WitnessLog reference -- threading concern.
        max_memory_pages in config is not enforced by AgentManager.

Step 3: Activate and run
  mgr.activate(id)?;
  // Execute WASM instructions...
  Pain: No actual WASM interpreter -- this is a lifecycle manager only.
        Developer must provide their own interpreter.

Step 4: Host function dispatch
  let result = dispatch_host_call(agent_id, HostFunction::Send, &args, &token, &mut ctx);
  Pain: HostCallArgs is positional (arg0, arg1, arg2).
        Must read docs to know which arg means what for each function.

Step 5: Migration
  let plan = MigrationPlan { agent_id, source, dest, deadline_ns };
  let mut tracker = MigrationTracker::begin(plan, now)?;
  for step in 0..7 {
      tracker.advance(now + step * 1_000_000, &witness_log)?;
  }
  Pain: Must call advance() 7 times manually.
        No way to execute full migration in one call.

Friction Rating: MEDIUM-HIGH
Key Bottleneck: No WASM interpreter, migration is manually stepped.
```

### Journey 4: Using the Coherence Engine

```
Developer Goal: Monitor partition communication, get split/merge recommendations

Step 1: Create engine
  let engine = DefaultCoherenceEngine::with_defaults(100);
  Pain: None. Clean factory method.

Step 2: Register partitions
  engine.add_partition(pid(1))?;
  engine.add_partition(pid(2))?;
  Pain: None. Straightforward.

Step 3: Record communication
  engine.record_communication(pid(1), pid(2), 500)?;
  Pain: None. Clear API.

Step 4: Tick and read scores
  let decision = engine.tick(cpu_load_percent);
  let score = engine.score(pid(1));
  let pressure = engine.pressure(pid(1));
  Pain: cpu_load_percent must be provided externally.
        No integration with actual CPU metrics.

Step 5: Act on decision
  match decision {
      CoherenceDecision::SplitRecommended { partition, pressure } => { ... }
      CoherenceDecision::MergeRecommended { a, b, mutual_coherence } => { ... }
      CoherenceDecision::NoAction => {}
  }
  Pain: Low. Pattern matching is idiomatic.

Friction Rating: LOW
Key Bottleneck: Standalone engine use is excellent. Integration with kernel
is where friction increases (kernel.tick() handles it automatically).
```

### Journey 5: GPU Compute Integration

```
Developer Goal: Launch a GPU compute kernel from a WASM agent

Step 1: Create GPU context
  let budget = GpuBudget::new(compute_ns, memory_bytes, transfer_bytes, max_launches);
  let ctx = GpuContext::new(partition_id, device_id, budget);
  Pain: Must construct GpuBudget with 4 raw u64 values.
        No sensible defaults or builder.

Step 2: Compile kernel
  let kernel = GpuKernel { ... };
  Pain: No actual compilation -- this is a metadata container.
        Must provide pre-compiled kernel bytes externally.

Step 3: Submit via WASM host function
  dispatch_host_call(agent, HostFunction::GpuLaunch, &args, &token, &mut ctx);
  Pain: GpuLaunch args are positional u64 values (kernel_id, config).
        No type safety for grid/block dimensions.
        Feature-gated behind "gpu" -- must enable at compile time.

Step 4: Transfer data
  dispatch_host_call(agent, HostFunction::GpuTransfer, &args, &token, &mut ctx);
  Pain: src/dst are buffer handles encoded as u64. No type safety.

Step 5: Synchronize
  dispatch_host_call(agent, HostFunction::GpuSync, &HostCallArgs::empty(), &token, &mut ctx);
  Pain: Blocking sync with no timeout parameter.

Friction Rating: HIGH
Key Bottleneck: End-to-end GPU path requires understanding 3 crates,
feature flags, and untyped u64 argument conventions.
```

---

## Error Handling Assessment

### Architecture: Strong (88/100)

The error architecture follows a well-designed layered pattern:

```
Per-crate errors          Unified error           Result alias
  CapError (10)    ---->   RvmError (34)    <----  RvmResult<T>
  ProofError (6)   ---->       ^
  GpuError (13)    ---->       |
  GraphError (5)   ---->       |
  SecurityError    ---->       |
```

Every per-crate error type implements `From<CrateError> for RvmError`, allowing transparent conversion with `?` across crate boundaries. This is the correct pattern for a multi-crate workspace.

### Specificity: Moderate (72/100)

| Issue | Instances | Impact |
|-------|-----------|--------|
| `InternalError` used as catch-all | 7+ call sites | Cannot distinguish failure causes |
| `InvalidPartitionState` overloaded | Boot, agent, migration, GPU | Wrong error for non-partition contexts |
| `ProofInvalid` used for WASM parsing | 8 call sites in rvm-wasm | Confusing error messages |
| `PartitionNotFound` used for agents | 3 call sites in rvm-wasm | Agent errors look like partition errors |
| `ResourceLimitExceeded` for zero-page regions | 1 call site | Zero pages is not a "limit exceeded" |

### Recoverability: Moderate (70/100)

Errors tell you *what* went wrong but rarely *how to recover*:

- `PartitionLimitExceeded`: Does not tell you the current count or maximum.
- `DelegationDepthExceeded`: Does not tell you the current depth or maximum.
- `WitnessLogFull`: Does not suggest draining or increasing capacity.
- `OutOfMemory`: Does not suggest which regions to free or tier to demote.

**Recommendation**: For high-frequency errors, add a `context()` or `suggestion()` method that returns a recovery hint string. Example:
```rust
RvmError::PartitionLimitExceeded.suggestion()
// -> "Maximum 256 partitions reached. Destroy unused partitions to free slots."
```

---

## Documentation Coverage

### Source Documentation

| Metric | Status | Score |
|--------|--------|-------|
| `deny(missing_docs)` enforced | All 14 crates | 100% |
| Module-level doc comments | All 14 crates | 100% |
| ADR cross-references | All 14 crates | 100% |
| `# Errors` sections on fallible fns | ~80% coverage | 80% |
| `# Safety` comments on unsafe blocks | HAL only (correct) | 100% |
| `#[must_use]` on non-consuming methods | Consistently applied | 95% |
| Doc-test examples | Very few | 15% |

### User Guide (15 chapters)

| Chapter | Quality | Notes |
|---------|---------|-------|
| 01-quickstart | Excellent | 5-step flow, copy-paste commands |
| 02-core-concepts | Excellent | Coherence domains well explained |
| 03-architecture | Excellent | Dependency diagrams |
| 04-crate-reference | Excellent | Every crate with types, flags, deps |
| 05-capabilities-proofs | Good | Missing end-to-end example |
| 06-witness-audit | Good | Replay queries well covered |
| 07-partitions-scheduling | Good | Missing scheduling strategy guide |
| 08-memory-model | Good | Four-tier model clear |
| 09-wasm-agents | Good | Missing interpreter guidance |
| 10-security | Good | Gate pipeline documented |
| 11-performance | Good | Benchmark methodology |
| 12-bare-metal | Good | QEMU instructions |
| 13-advanced-exotic | Good | Novel use cases |
| 14-troubleshooting | Adequate | Common errors listed |
| 15-glossary | Good | Terms defined |

**Overall Documentation Score: 88/100**

The documentation is significantly above average for an embedded systems project. The gap is in executable examples -- doc-tests that a developer can cargo-test to verify their understanding.

---

## Consistency Findings

### Highly Consistent (Positive)

1. **Lint directives**: Every crate uses identical `#![no_std]`, `#![deny(missing_docs)]`, `#![deny(clippy::all)]`, `#![warn(clippy::pedantic)]`.
2. **Feature flags**: Every crate supports `alloc` and `std` features with identical conditional extern crate blocks.
3. **Re-export pattern**: Every crate re-exports key types at root level with `// Re-export commonly used types from rvm-types.` comment.
4. **Error pattern**: Every crate uses `RvmResult<T>` for fallible public APIs.
5. **Constructor pattern**: Every manager type has `fn new() -> Self` and `impl Default`.
6. **ID types**: All identifiers are `#[repr(transparent)]` newtypes over `u32` or `u64`.

### Inconsistencies Found

| Issue | Crates Affected | Severity |
|-------|----------------|----------|
| `enqueue()` returns `bool` instead of `RvmResult` | rvm-sched | Medium |
| `clippy::missing_errors_doc` suppressed in 5 crates | rvm-hal, rvm-sched, rvm-coherence, rvm-boot, rvm-wasm | Low |
| Some crates allow `clippy::new_without_default` but implement Default anyway | rvm-hal, rvm-sched | Low |
| `PartitionState` defined in both rvm-types and rvm-partition | rvm-types, rvm-partition | Medium |
| `CommEdge` and `CommEdgeId` defined in both rvm-types and rvm-partition | rvm-types, rvm-partition | Medium |
| `WitnessRecord` field types use `u8`/`u64` raw types instead of domain newtypes | rvm-types, rvm-witness | Medium |
| Test count discrepancy: README says 945, quickstart says 648 | README.md, userguide | Low |

---

## Discoverability Assessment

### What Works Well

1. **`rvm-kernel` as facade**: Developers can access the entire system through `rvm_kernel::{types, partition, cap, witness, proof, sched, memory, coherence, boot, wasm, security, gpu}`. This is the correct "one crate to rule them all" pattern.
2. **Re-exports at crate roots**: Each crate re-exports its most important types at the top level, so `use rvm_cap::CapabilityManager` works without knowing the module structure.
3. **ADR references in docs**: Every type links to its specification, so developers can trace design decisions.

### What Could Improve

1. **No "cookbook" or "recipes" section**: Developers need to read multiple chapters to accomplish common tasks. A recipes page showing "Create a partition with GPU access in 10 lines" would be high-value.
2. **No `prelude` module**: A `rvm_kernel::prelude` re-exporting the 15-20 most commonly used types would reduce import boilerplate.
3. **Feature flag interactions undocumented**: Which feature flags can be combined? What happens with `gpu` + `ruvector`? No feature interaction matrix exists.
4. **No `examples/` directory**: The workspace has no `examples/` crate with runnable demonstrations. The user guide has code blocks but they are not compiled.

---

## Top 15 Recommendations for DX Improvement

### Priority 1: High Impact, Low Effort

| # | Recommendation | Effort | Impact |
|---|---------------|--------|--------|
| 1 | **Introduce `CapHandle { index: u32, gen: u32 }` newtype** to replace all `(u32, u32)` returns in rvm-cap. Eliminates parameter-swap bugs. | Small | High |
| 2 | **Add `WitnessRecord::for_action(kind: ActionKind)` builder** with typed setters `.actor(id)`, `.target(id)`, `.tier(n)`. Eliminates 40+ lines of boilerplate in rvm-kernel. | Small | High |
| 3 | **Add `IpcMessage::new(sender, receiver, edge_id)` constructor** with auto-sequencing and hash computation. Reduce 7-field manual struct construction. | Small | Medium |
| 4 | **Change `Scheduler::enqueue()` to return `RvmResult<()>`** for consistency with all other crate APIs. | Small | Medium |
| 5 | **Add `#[cfg(feature = "std")] impl std::error::Error for RvmError {}`** to enable integration with the broader Rust error ecosystem. | Tiny | Medium |

### Priority 2: Medium Impact, Medium Effort

| # | Recommendation | Effort | Impact |
|---|---------------|--------|--------|
| 6 | **Add `RvmError::WasmValidationFailed`** and stop using `ProofInvalid` for WASM parsing. Add `RvmError::AgentNotFound` for agent lookups. | Small | Medium |
| 7 | **Enrich `RvmError::Display` messages** with context (e.g., "partition limit exceeded (max: 256)" instead of "maximum partition count reached"). | Medium | Medium |
| 8 | **Add `Kernel::emit_witness()` private helper** to deduplicate witness record construction across 10+ methods. | Small | Medium |
| 9 | **Add partition state machine diagram** in rvm-partition docs showing valid transitions between Created/Running/Suspended/Destroyed/Hibernated. | Small | Medium |
| 10 | **Add `rvm_kernel::prelude` module** re-exporting `Kernel, KernelConfig, PartitionId, CapRights, CapType, RvmError, RvmResult, CoherenceDecision`. | Tiny | Medium |

### Priority 3: High Impact, High Effort

| # | Recommendation | Effort | Impact |
|---|---------------|--------|--------|
| 11 | **Add `KernelBuilder`** allowing configurable dimensions (max CPUs, witness capacity, cap table size) instead of hardcoded constants. | Medium | High |
| 12 | **Add typed host function argument structs** (`SendArgs`, `AllocArgs`, `GpuLaunchArgs`) replacing positional `HostCallArgs { arg0, arg1, arg2 }`. | Medium | High |
| 13 | **Add `examples/` workspace crate** with 5-6 runnable examples: basic boot, partition lifecycle, capability chains, IPC messaging, coherence monitoring, GPU compute. | Large | High |
| 14 | **Add `MigrationTracker::run_to_completion()`** that executes all 7 steps in one call, checking timeout at each step. Keep `advance()` for fine-grained control. | Small | Medium |
| 15 | **Split `RvmError::InternalError`** into at least 3 variants: `SlotExhausted`, `IndexOutOfRange`, `InvariantViolation`. | Medium | Medium |

---

## Appendix: Error Variant Inventory

### RvmError (34 variants)

**Partition (6)**: PartitionNotFound, InvalidPartitionState, PartitionLimitExceeded, SplitPreconditionFailed, MergePreconditionFailed, MigrationTimeout

**vCPU (2)**: VcpuNotFound, VcpuLimitReached

**Capability (5)**: InsufficientCapability, StaleCapability, CapabilityTypeMismatch, DelegationDepthExceeded, CapabilityConsumed

**Witness (3)**: WitnessVerificationFailed, WitnessChainBroken, WitnessLogFull

**Proof (3)**: ProofInvalid, ProofTierInsufficient, ProofBudgetExceeded

**Coherence (2)**: CoherenceBelowThreshold, MinCutBudgetExceeded

**Memory (4)**: MemoryOverlap, AlignmentError, InvalidTierTransition, OutOfMemory

**Device (3)**: DeviceLeaseNotFound, DeviceLeaseExpired, DeviceLeaseConflict

**Recovery (3)**: CheckpointNotFound, CheckpointCorrupted, FailureEscalated

**General (3)**: ResourceLimitExceeded, Unsupported, InternalError

### CapError (10 variants)
InvalidHandle, StaleHandle, TableFull, Revoked, DelegationDepthExceeded, GrantNotPermitted, RightsEscalation, TreeFull, TypeMismatch, Consumed

### ProofError (6 variants)
InvalidHandle, StaleCapability, InsufficientRights, PolicyViolation, P3NotImplemented, DerivationChainBroken

### GpuError (13 variants)
DeviceNotFound, DeviceNotReady, OutOfMemory, BudgetExceeded, KernelTimeout, KernelCompilationFailed, InvalidLaunchConfig, BufferTooLarge, QueueFull, IommuViolation, CapabilityDenied, TransferFailed, Unsupported

### GraphError (5 variants)
DuplicateNode, NodeCapacityExhausted, EdgeCapacityExhausted, NodeNotFound, EdgeNotFound

**Total unique error variants across the project: 68**

---

## Appendix: Const-Generic Usage Inventory

| Type | Generic | Default | Purpose |
|------|---------|---------|---------|
| `CapabilityManager<N>` | Table capacity | 256 | Per-partition cap table size |
| `CapabilityTable<N>` | Slot count | 256 | Capability storage |
| `DerivationTree<N>` | Node count | 256 | Derivation tracking |
| `ProofVerifier<N>` | Unused (matches table) | 256 | Proof verification |
| `WitnessLog<W>` | Ring capacity | 262144 | Witness record buffer |
| `AgentManager<MAX>` | Agent limit | None | WASM agent registry |
| `QuotaTracker<MAX>` | Partition limit | None | Resource budget tracking |
| `Scheduler<CPUS, PARTS>` | CPU count, partitions | None | Per-CPU run queues |
| `IpcManager<EDGES, QSIZE>` | Channel count, queue depth | None | IPC message queues |
| `MessageQueue<CAP>` | Queue depth | None | Per-channel ring buffer |
| `TierManager<MAX>` | Region count | 256 | Memory tier tracking |
| `CoherenceGraph<N, E>` | Nodes, edges | None | Communication topology |

This consistent use of const generics is a strong DX choice: it enables compile-time sizing without heap allocation, directly supporting the `no_std` constraint.

---

*Analysis performed on the complete public API surface of all 14 RVM crates. Every `lib.rs`, key submodule, error type, and user guide chapter was read and assessed.*
