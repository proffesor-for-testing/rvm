# RVM Code Quality & Complexity Analysis Report

**Date:** 2026-04-07
**Scope:** All 14 crates in `/workspaces/rvm/crates/`
**Analyzer:** QE Code Complexity Analyzer v3
**Total LOC:** 32,543 across 106 Rust source files
**Total Tests:** 895 (in-source `#[test]` functions)

---

## Executive Summary

The RVM codebase demonstrates **excellent overall code quality** for a `no_std` Rust microhypervisor. The project is well-architected with clear separation of concerns across 14 crates, comprehensive documentation, and strong safety guarantees through Rust's type system and `#![forbid(unsafe_code)]` enforcement on 12 of 14 crates.

**Key Strengths:**
- Zero `todo!()`, `unimplemented!()`, `FIXME`, `HACK`, or `XXX` markers in the entire codebase
- All 14 crates enforce `#![deny(clippy::all)]` and `#![warn(clippy::pedantic)]`
- 12 of 14 crates use `#![forbid(unsafe_code)]`; only `rvm-hal` and `rvm-kernel/main.rs` permit unsafe (correctly, at the hardware boundary)
- Consistent use of const generics for zero-allocation, stack-only data structures
- Constant-time security paths to prevent timing side-channel leakage (ADR-135)
- Comprehensive test coverage: 895 tests with integration-style lifecycle tests

**Key Concerns:**
- **rvm-kernel/src/lib.rs** at 2,389 lines is a monolith that should be decomposed
- **Duplicated nonce ring buffer logic** in `rvm-cap/verify.rs` and `rvm-proof/policy.rs`
- **Duplicated FNV-1a hash** implementation in `rvm-memory/reconstruction.rs` and `rvm-witness/signer.rs` (separate from the canonical `rvm-types::fnv1a_64`)
- **Duplicated `SecurityGate` / `SignedSecurityGate`** check_and_execute logic (~50 lines)
- `stoer_wagner_mincut` function has the highest cognitive complexity in the codebase
- `Kernel` struct is a god object holding 11 fields with direct mutable access to all subsystems

**Overall Rating:** GOOD (maintainability index ~72/100)

---

## Per-Crate Complexity Metrics

| Crate | LOC | Files | Tests | Unsafe | Complexity Rating |
|-------|-----|-------|-------|--------|-------------------|
| rvm-types | 1,573 | 14 | -- | `forbid` | LOW |
| rvm-hal | 1,872 | 7 | 18 | `deny` + `allow(aarch64)` | MEDIUM |
| rvm-cap | 2,553 | 8 | 45 | `forbid` | MEDIUM |
| rvm-witness | 1,404 | 7 | 45 | `forbid` | LOW |
| rvm-proof | 3,992 | 10 | 151 | `forbid` | MEDIUM |
| rvm-partition | 2,208 | 11 | 89 | `forbid` | LOW |
| rvm-sched | 1,445 | 9 | 51 | `forbid` | LOW |
| rvm-memory | 3,628 | 5 | 110 | `forbid` | MEDIUM |
| rvm-coherence | 3,194 | 8 | 68 | `forbid` | HIGH |
| rvm-boot | 1,179 | 5 | 26 | `forbid` | LOW |
| rvm-wasm | 1,946 | 5 | 38 | `forbid` | LOW |
| rvm-security | 1,984 | 5 | 54 | `forbid` | MEDIUM |
| rvm-kernel | 2,595 | 2 | 67 | `forbid(lib)` / `allow(main)` | HIGH |
| rvm-gpu | 2,970 | 10 | 133 | `forbid` | MEDIUM |

---

## Top 10 Most Complex Functions

### 1. `stoer_wagner_mincut` -- rvm-coherence/src/mincut.rs:226
- **Cyclomatic Complexity:** ~18
- **Cognitive Complexity:** ~22
- **Lines:** 80
- **Nesting Depth:** 4 levels (while > if > for > if)
- **Issue:** Nested loops with array mutations, multiple tracking variables (`best_cut_weight`, `best_cut_node`, `active_count`, `iteration_count`). Core algorithm logic, but readability suffers from flat array manipulation in `no_std` context.
- **Recommendation:** Extract the merge step and group-transfer step into helper functions.

### 2. `find_min_cut` (MinCutBridge) -- rvm-coherence/src/mincut.rs:104
- **Cyclomatic Complexity:** ~16
- **Cognitive Complexity:** ~20
- **Lines:** 95
- **Nesting Depth:** 4 levels (if-let > for > if > if-let)
- **Issue:** Subgraph collection involves checking neighbors, incoming edges, and deduplication across three separate loops with deeply nested conditionals.
- **Recommendation:** Extract subgraph collection into a dedicated helper function.

### 3. `recommend` (CoherenceEngine) -- rvm-coherence/src/engine.rs:312
- **Cyclomatic Complexity:** ~14
- **Cognitive Complexity:** ~18
- **Lines:** 72
- **Nesting Depth:** 4 levels (for > if > for > match)
- **Issue:** O(N^2) pairwise merge candidate search with nested match arms. The active entries collection into a local array adds an extra loop.
- **Recommendation:** Extract split-candidate search and merge-candidate search into separate methods.

### 4. `check_and_execute` (SecurityGate) -- rvm-security/src/gate.rs:155
- **Cyclomatic Complexity:** ~12
- **Cognitive Complexity:** ~16
- **Lines:** 50
- **Nesting Depth:** 3 levels
- **Issue:** Pipeline of 4 security stages with early-return error paths and witness emission on each. Duplicated almost entirely in `SignedSecurityGate::check_and_execute`.
- **Recommendation:** Extract shared pipeline logic into a generic helper, parameterized on witness emission behavior.

### 5. `compress` -- rvm-memory/src/reconstruction.rs:332
- **Cyclomatic Complexity:** ~12
- **Cognitive Complexity:** ~14
- **Lines:** 57
- **Nesting Depth:** 3 levels (while > if/else > while)
- **Issue:** Inner while loops for zero-run and literal collection with boundary checks at each step. Clean for a compression codec, but slightly over the complexity threshold.
- **Recommendation:** Acceptable for an algorithmic function; could extract `emit_zero_run` and `emit_literal` helpers.

### 6. `decompress` -- rvm-memory/src/reconstruction.rs:392
- **Cyclomatic Complexity:** ~11
- **Cognitive Complexity:** ~13
- **Lines:** 50
- **Nesting Depth:** 3 levels (while > match > if)
- **Issue:** Mirror of `compress` with tag-dispatch loop. Well-structured but over threshold.

### 7. `reconstruct` (ReconstructionPipeline) -- rvm-memory/src/reconstruction.rs:183
- **Cyclomatic Complexity:** ~11
- **Cognitive Complexity:** ~13
- **Lines:** 76
- **Nesting Depth:** 3 levels (for > if-let > if)
- **Issue:** Three-phase pipeline (decompress, verify hash, apply deltas) with multiple early-return error paths. Well-documented but dense.

### 8. `verify_p3` (ProofVerifier) -- rvm-cap/src/verify.rs:219
- **Cyclomatic Complexity:** ~11
- **Cognitive Complexity:** ~14
- **Lines:** 80
- **Nesting Depth:** 3 levels (loop > match > if)
- **Issue:** Derivation tree walk with 5 validation checks per ancestor node. Each check produces a distinct early-return error, making the control flow dense.
- **Recommendation:** Extract per-ancestor validation into a `validate_ancestor` helper.

### 9. `create` (RegionManager) -- rvm-memory/src/region.rs:165
- **Cyclomatic Complexity:** ~10
- **Cognitive Complexity:** ~12
- **Lines:** 55
- **Nesting Depth:** 3 levels (for > if > if)
- **Issue:** Combined single-pass overlap check and free-slot search. Both guest and host overlap must be checked. Well-optimized but complex.

### 10. `minimum_cut_phase` -- rvm-coherence/src/mincut.rs:310
- **Cyclomatic Complexity:** ~10
- **Cognitive Complexity:** ~12
- **Lines:** 52
- **Nesting Depth:** 3 levels (for > for > if)
- **Issue:** Maximum adjacency ordering with manual tracking of `max_key`, `max_node`, and `found` flag. Standard Stoer-Wagner phase, but the flat-array approach adds complexity.

---

## Code Smell Findings

### CS-1: God Object -- `Kernel` struct (MEDIUM severity)
**Location:** `rvm-kernel/src/lib.rs:238-261`
- Holds 11 fields: `partitions`, `scheduler`, `witness_log`, `cap_manager`, `coherence`, `ipc`, `tier_manager`, `devices`, `boot`, `config`, `booted`
- 40+ public methods on a single `impl` block
- File is 2,389 lines (largest in the project by far)
- **Impact:** Hard to test subsystem interactions in isolation; large compile unit
- **Recommendation:** Extract subsystem facades (e.g., `KernelIpc`, `KernelCoherence`, `KernelScheduling`) that hold references to the relevant fields and implement domain-specific methods. The `Kernel` struct would delegate to these facades.

### CS-2: Duplicated Nonce Ring Buffer (HIGH severity)
**Location:**
- `rvm-cap/src/verify.rs:42-78` (`ProofVerifier` nonce ring)
- `rvm-proof/src/policy.rs:36-76` (`PolicyEvaluator` nonce ring)

Both implement identical logic:
- `nonce_ring: [u64; 4096]`
- `nonce_hash: [u64; 4096]`
- `nonce_write_pos: usize`
- `nonce_watermark: u64`
- `allow_zero_nonce: bool`
- Identical `is_nonce_replayed()`, `record_nonce()`, `check_nonce()` methods

**Impact:** ~200 lines of duplicated security-critical logic. A bug fix in one location could be missed in the other.
**Recommendation:** Extract a `NonceTracker<const SIZE: usize>` struct into `rvm-types` or a shared `rvm-nonce` module, and embed it in both `ProofVerifier` and `PolicyEvaluator`.

### CS-3: Duplicated FNV-1a Hash Implementation (MEDIUM severity)
**Location:**
- `rvm-types/src/witness.rs:334` -- Canonical `pub fn fnv1a_64()` (re-exported as `rvm_types::fnv1a_64`)
- `rvm-witness/src/signer.rs:111` -- Private `fn fnv1a_64()` (identical)
- `rvm-memory/src/reconstruction.rs:453` -- Private `fn fnv1a_hash()` (identical)

**Impact:** Three copies of the same hash function. The canonical one in `rvm-types` is already public and re-exported, so the other two are unnecessary.
**Recommendation:** Remove the private copies in `rvm-witness/signer.rs` and `rvm-memory/reconstruction.rs`; use `rvm_types::fnv1a_64` instead.

### CS-4: Duplicated SecurityGate Pipeline (LOW severity)
**Location:**
- `rvm-security/src/gate.rs:155-204` (`SecurityGate::check_and_execute`)
- `rvm-security/src/gate.rs:268-312` (`SignedSecurityGate::check_and_execute`)

~50 lines of near-identical pipeline logic (P1 type check, P1 rights check, P2 proof commitment, P3 chain verification), differing only in witness emission and P3 signature verification.
**Recommendation:** Extract a common pipeline method parameterized by a `WitnessEmitter` trait.

### CS-5: Witness Record Construction Boilerplate (LOW severity)
**Location:** Throughout `rvm-kernel/src/lib.rs` -- at least 15 instances of:
```rust
let mut record = WitnessRecord::zeroed();
record.action_kind = ActionKind::Xxx as u8;
record.proof_tier = 1;
record.actor_partition_id = ...;
record.target_object_id = ...;
self.witness_log.append(record);
```
**Recommendation:** Add a builder or helper function: `WitnessRecord::for_action(action, tier, actor, target)`.

### CS-6: Large Stack Arrays in Security-Critical Structs (INFO)
**Location:**
- `ProofVerifier` contains `[u64; 4096]` x2 = 64 KiB per instance
- `PolicyEvaluator` contains `[u64; 4096]` x2 = 64 KiB per instance
- `CoherenceGraph` contains `[[u64; 32]; 32]` = 8 KiB adjacency matrix

These are correctly marked with `#[allow(clippy::large_stack_arrays)]` and are intentional for `no_std`. Not a smell per se, but consumers should be aware of stack requirements.

---

## Unsafe Code Inventory

### Summary

| Crate | Unsafe Policy | Unsafe Blocks | Assessment |
|-------|--------------|---------------|------------|
| rvm-types | `forbid` | 0 | SAFE |
| rvm-hal | `deny` + `allow(aarch64)` | ~35 | NECESSARY |
| rvm-cap | `forbid` | 0 | SAFE |
| rvm-witness | `forbid` | 0 | SAFE |
| rvm-proof | `forbid` | 0 | SAFE |
| rvm-partition | `forbid` | 0 | SAFE |
| rvm-sched | `forbid` | 0 | SAFE |
| rvm-memory | `forbid` | 0 | SAFE |
| rvm-coherence | `forbid` | 0 | SAFE |
| rvm-boot | `forbid` | 0 | SAFE |
| rvm-wasm | `forbid` | 0 | SAFE |
| rvm-security | `forbid` | 0 | SAFE |
| rvm-kernel (lib.rs) | `forbid` | 0 | SAFE |
| rvm-kernel (main.rs) | `allow` | ~12 | NECESSARY |
| rvm-gpu | `forbid` | 0 | SAFE |

### Unsafe Code Detail (rvm-hal)

All unsafe code is concentrated in `rvm-hal/src/aarch64/`, which is explicitly designated as the hardware boundary. Every unsafe block has a `// SAFETY:` comment.

**uart.rs** (6 unsafe functions):
- `uart_write()` / `uart_read()` -- MMIO volatile access. Necessary.
- `uart_init()` -- PL011 register configuration. Necessary.
- `uart_putc()` / `uart_puts()` / `uart_put_hex()` -- Serial output. Necessary.

**interrupts.rs** (7 unsafe functions):
- `gic_write()` / `gic_read()` -- GIC MMIO access. Necessary.
- `gic_init()` / `gic_enable_irq()` / `gic_disable_irq()` -- Interrupt controller setup. Necessary.
- `gic_ack()` / `gic_eoi()` -- Interrupt acknowledgment. Necessary.

**boot.rs** (5 unsafe functions):
- `current_el()` -- Reads `CurrentEL` system register via inline assembly. Necessary.
- `configure_hcr_el2()` -- Writes hypervisor configuration register. Necessary, with hard assert on EL2.
- `context_switch()` / `context_switch_inner()` -- Register save/restore. Necessary.
- `context_switch_vmid()` -- VMID-aware context switch. Necessary.
- `clear_bss()` -- BSS zeroing. Necessary.

**mmu.rs** (1 unsafe function):
- `install()` -- MMU page table installation. Necessary.

**timer.rs** (5 unsafe blocks):
- System register access for timer configuration. Necessary.

**Assessment:** All unsafe code is well-justified, correctly documented with SAFETY comments, and confined to the hardware abstraction layer. The architecture correctly isolates unsafe to a single crate module, with all other crates using `#![forbid(unsafe_code)]`.

### Unsafe Code Detail (rvm-kernel/main.rs)

Entry point for the bare-metal binary. Contains ~12 unsafe blocks for:
- Calling HAL initialization functions
- Reading system registers
- Early boot operations

All are necessary for a bare-metal hypervisor entry point.

---

## Lines of Code Distribution

### By Crate (source only, excluding tests)

| Crate | Total LOC | Largest File | File LOC |
|-------|-----------|--------------|----------|
| rvm-proof | 3,992 | signer.rs | 1,011 |
| rvm-memory | 3,628 | reconstruction.rs | 1,217 |
| rvm-coherence | 3,194 | graph.rs | 726 |
| rvm-gpu | 2,970 | tests.rs | 689 |
| rvm-cap | 2,553 | verify.rs | 569 |
| rvm-kernel | 2,595 | lib.rs | 2,389 |
| rvm-partition | 2,208 | ipc.rs | 633 |
| rvm-security | 1,984 | gate.rs | 821 |
| rvm-wasm | 1,946 | quota.rs | 517 |
| rvm-hal | 1,872 | boot.rs | 478 |
| rvm-types | 1,573 | (distributed) | ~200 |
| rvm-sched | 1,445 | smp.rs | 417 |
| rvm-witness | 1,404 | signer.rs | 502 |
| rvm-boot | 1,179 | sequence.rs | ~300 |

### Files Over 500 Lines (Complexity Hotspots)

| File | LOC | Concern |
|------|-----|---------|
| rvm-kernel/src/lib.rs | 2,389 | GOD OBJECT -- needs decomposition |
| rvm-memory/src/reconstruction.rs | 1,217 | Acceptable: compression + pipeline + tests |
| rvm-proof/src/signer.rs | 1,011 | Acceptable: multiple signer impls + tests |
| rvm-memory/src/tier.rs | 890 | Acceptable: tier manager + 41 tests |
| rvm-security/src/gate.rs | 821 | Acceptable: gate + signed gate + tests |
| rvm-coherence/src/graph.rs | 726 | Acceptable: adjacency structure + iterators |
| rvm-memory/src/allocator.rs | 691 | Acceptable: buddy allocator + tests |
| rvm-gpu/src/tests.rs | 689 | Tests only -- fine |
| rvm-coherence/src/engine.rs | 668 | Acceptable: engine + tests |
| rvm-memory/src/region.rs | 663 | Acceptable: region manager + tests |
| rvm-partition/src/ipc.rs | 633 | Acceptable: IPC manager + message queue |
| rvm-proof/src/policy.rs | 630 | Contains nonce ring duplication |
| rvm-proof/src/engine.rs | 620 | Acceptable: proof engine + test helpers |
| rvm-cap/src/verify.rs | 569 | Contains nonce ring duplication |
| rvm-cap/src/manager.rs | 565 | Acceptable: capability manager |

---

## Maintainability Assessment

### Module Coupling (GOOD)

The crate dependency graph follows a clean layered architecture:

```
Layer 4: rvm-kernel (integration)
Layer 3: rvm-partition, rvm-sched, rvm-memory, rvm-boot
Layer 2: rvm-cap, rvm-witness, rvm-proof, rvm-security, rvm-wasm, rvm-gpu
Layer 1: rvm-coherence
Layer 0: rvm-types, rvm-hal
```

- No circular dependencies
- `rvm-types` is the shared vocabulary crate (correct)
- `rvm-hal` provides hardware abstractions consumed by upper layers
- Each crate has a well-defined bounded context

### Cohesion (GOOD)

Most crates have high cohesion -- each focuses on a single concern:
- `rvm-cap`: Capability table, derivation tree, grant, revoke, verify
- `rvm-coherence`: Graph, scoring, mincut, pressure, adaptive engine
- `rvm-memory`: Allocator, region manager, tier manager, reconstruction
- `rvm-proof`: Proof engine, policy evaluator, signers, TEE integration

**Exception:** `rvm-kernel` has low cohesion -- it mixes integration wiring, subsystem facades, host context implementation, and 67 tests in a single file.

### Abstraction Quality (EXCELLENT)

- Consistent use of Rust traits for extensibility (`MinCutBackend`, `CoherenceBackend`, `WitnessSigner`, `HostContext`)
- Feature gates (`crypto-sha256`, `ruvector`, `gpu`, `wasm`) for optional functionality
- Const generics (`MAX_NODES`, `MAX_EDGES`, `TOTAL_PAGES`, etc.) for zero-allocation parameterization
- Newtype wrappers (`PartitionId`, `OwnedRegionId`, `PhysAddr`, `GuestPhysAddr`) prevent value confusion

### Documentation Quality (EXCELLENT)

- Every public item has doc comments
- Module-level documentation with architecture diagrams
- ADR references in doc comments (ADR-132, ADR-134, ADR-135, ADR-136, ADR-137, ADR-138, ADR-142)
- Security notes on sensitive operations
- SAFETY comments on every unsafe block

---

## Clippy/Lint Status

All 14 crates enforce:
```rust
#![deny(clippy::all)]
#![warn(clippy::pedantic)]
```

With consistent, well-justified allowances:
```rust
#![allow(
    clippy::cast_possible_truncation,  // Intentional for u64->u32 in no_std
    clippy::cast_lossless,
    clippy::missing_errors_doc,
    clippy::missing_panics_doc,
    clippy::must_use_candidate,
    clippy::doc_markdown,
    clippy::new_without_default
)]
```

No `todo!()`, `unimplemented!()`, `FIXME`, `HACK`, or `XXX` markers found in the entire codebase.

---

## Code Duplication Summary

| Pattern | Locations | Lines Duplicated | Severity |
|---------|-----------|------------------|----------|
| Nonce ring buffer (struct + methods) | rvm-cap/verify.rs, rvm-proof/policy.rs | ~200 | HIGH |
| FNV-1a hash function | rvm-types, rvm-witness/signer.rs, rvm-memory/reconstruction.rs | ~30 | MEDIUM |
| SecurityGate pipeline | gate.rs (SecurityGate + SignedSecurityGate) | ~50 | LOW |
| WitnessRecord construction boilerplate | rvm-kernel/lib.rs (15+ instances) | ~90 | LOW |
| Measured boot hash accumulation pattern | rvm-boot/entry.rs, rvm-boot/measured.rs, rvm-security/attestation.rs | ~20 | LOW |

---

## Recommendations

### Priority 1 (HIGH) -- Should fix before next major release

**R1: Extract shared `NonceTracker` from duplicated nonce ring implementations**
- Create `pub struct NonceTracker<const SIZE: usize>` in `rvm-types` or a new `rvm-nonce` crate
- Move nonce ring, hash index, watermark, and all replay-detection logic into it
- Embed in both `ProofVerifier` and `PolicyEvaluator` via composition
- Estimated effort: 2-3 hours
- Removes ~200 lines of duplication in security-critical code

**R2: Decompose `rvm-kernel/src/lib.rs` into focused modules**
- Extract `kernel_ipc.rs` (IPC channel management + witness emission)
- Extract `kernel_coherence.rs` (coherence score sync, split/merge execution)
- Extract `kernel_sched.rs` (enqueue, switch, degraded mode)
- Extract `kernel_tier.rs` (region management, promotion/demotion)
- Extract `kernel_device.rs` (device lease management)
- Extract `kernel_security.rs` (checked_create_partition, checked_ipc_send)
- Keep `lib.rs` as the `Kernel` struct definition + boot + tick + re-exports
- Estimated effort: 4-6 hours
- Reduces `lib.rs` from 2,389 to ~400 lines

### Priority 2 (MEDIUM) -- Should fix in upcoming sprint

**R3: Remove duplicated FNV-1a implementations**
- Replace private `fnv1a_64` in `rvm-witness/src/signer.rs` with `rvm_types::fnv1a_64`
- Replace private `fnv1a_hash` in `rvm-memory/src/reconstruction.rs` with `rvm_types::fnv1a_64`
- Estimated effort: 30 minutes

**R4: Add `WitnessRecord` builder helper**
- Add `WitnessRecord::for_action(action: ActionKind, tier: u8, actor: PartitionId, target: u64) -> Self`
- Reduces boilerplate in `rvm-kernel` (15+ sites) and `rvm-security` (4+ sites)
- Estimated effort: 1 hour

**R5: Extract `recommend()` into split/merge helpers in coherence engine**
- Split `recommend()` into `find_split_candidate()` and `find_merge_candidate()`
- Reduces cyclomatic complexity from ~14 to ~7 each
- Estimated effort: 1 hour

### Priority 3 (LOW) -- Nice to have

**R6: Unify SecurityGate and SignedSecurityGate pipeline**
- Introduce a trait or closure-based approach for witness emission
- Reduces ~50 lines of duplication

**R7: Extract mincut subgraph collection into a helper**
- `MinCutBridge::find_min_cut` has a 60-line subgraph collection block
- Moving to `collect_local_subgraph()` would improve readability

**R8: Consider splitting the `Kernel` struct into facade objects**
- Instead of 11 owned fields, use a `KernelSubsystems` struct pattern
- This would enable more granular borrowing (e.g., borrow IPC without blocking scheduler access)
- Larger refactor, but would significantly improve testability

---

## Testability Assessment

| Crate | Test Count | Tests/100 LOC | Testability Score | Rating |
|-------|-----------|---------------|-------------------|--------|
| rvm-kernel | 67 | 2.8 | 75/100 | Good |
| rvm-gpu | 133 | 4.5 | 90/100 | Excellent |
| rvm-proof | 151 | 3.8 | 85/100 | Excellent |
| rvm-memory | 110 | 3.0 | 85/100 | Excellent |
| rvm-coherence | 68 | 2.1 | 80/100 | Good |
| rvm-partition | 89 | 4.0 | 85/100 | Excellent |
| rvm-cap | 45 | 1.8 | 75/100 | Good |
| rvm-security | 54 | 2.7 | 80/100 | Good |
| rvm-witness | 45 | 3.2 | 80/100 | Good |
| rvm-sched | 51 | 3.5 | 80/100 | Good |
| rvm-wasm | 38 | 2.0 | 75/100 | Good |
| rvm-boot | 26 | 2.2 | 70/100 | Acceptable |
| rvm-hal | 18 | 1.0 | 50/100 | Limited (hardware) |
| rvm-types | -- | -- | 95/100 | Excellent (pure data) |

**Notes:**
- `rvm-hal` has limited testability because its functions require hardware (MMIO, system registers). The project correctly tests what it can and marks HAL as the unsafe boundary.
- `rvm-types` has no direct tests because it's a pure data/type crate tested transitively by all consumers.
- All `.unwrap()` calls in non-test code are justified (e.g., `HMAC key length is 32 bytes` -- cryptographic invariants).

---

## Complexity Distribution

| Level | Cyclomatic Range | Functions | Percentage |
|-------|-----------------|-----------|------------|
| Low | 1-5 | ~280 | ~75% |
| Medium | 6-10 | ~70 | ~19% |
| High | 11-20 | ~20 | ~5% |
| Critical | >20 | 0 | 0% |

The codebase has **zero functions with critical complexity** (>20 cyclomatic). The highest-complexity functions are all in algorithmic modules (Stoer-Wagner mincut, compression/decompression, derivation tree walk) where the complexity is inherent to the algorithm.

---

## Conclusion

The RVM codebase is in **strong condition** for a systems-level Rust project. The architecture is clean, the safety guarantees are properly enforced, and the test coverage is comprehensive. The main actionable items are:

1. **Extract the duplicated nonce ring buffer** to eliminate the highest-severity code duplication in security-critical logic
2. **Decompose `rvm-kernel/src/lib.rs`** to address the god object pattern and improve long-term maintainability
3. **Remove the duplicated FNV-1a hash implementations** in favor of the canonical one in `rvm-types`

These changes would bring the maintainability index from ~72/100 to an estimated ~82/100 with modest effort.
