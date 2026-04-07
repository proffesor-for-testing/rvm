# RVM Security Analysis Report

**Date:** 2026-04-07
**Reviewer:** QE Security Reviewer (V3)
**Scope:** Full security audit of 14 crates, focused on rvm-security, rvm-cap, rvm-proof, rvm-witness, rvm-gpu
**Risk Rating:** MODERATE (improved from previous HIGH)
**Previous Audit:** 11 findings (8 fixed, 3 residual/accepted)

---

## Executive Summary

The RVM codebase demonstrates strong security engineering fundamentals. All 14 crates use `#![no_std]` and `#![forbid(unsafe_code)]` (except rvm-hal, which correctly confines `unsafe` to MMIO hardware boundary operations). The three-tier proof system (P1/P2/P3), capability-based access control with derivation trees, and append-only witness trail form a defense-in-depth architecture.

The previous audit identified 11 issues. Of those, 8 have been fully remediated: timing side channels now use the `subtle` crate for constant-time comparison, the nonce ring has been expanded from 64 to 4096 entries with a monotonic watermark, GRANT_ONCE consumption is implemented, slot leak on derivation failure is fixed, the stack overflow in revocation traversal has been replaced with iterative bounded-stack walks, revocation now synchronizes both derivation tree and capability table, NullSigner is gated behind `#[cfg(any(test, feature = "null-signer"))]`, and generation counters skip zero on wrap-around.

This review identifies **7 new findings** (0 Critical, 2 High, 3 Medium, 2 Low) and confirms **3 residual accepted risks** from the prior audit.

---

## Finding Summary

| ID | Severity | Category | Title | Status |
|----|----------|----------|-------|--------|
| SEC-001 | HIGH | Cryptography | FNV-1a used for proof verification commitments | NEW |
| SEC-002 | HIGH | Cryptography | 8-byte HMAC-SHA256 truncation in witness signatures | NEW |
| SEC-003 | MEDIUM | Nonce/Replay | Nonce hash-index collision allows replay in edge cases | NEW |
| SEC-004 | MEDIUM | Side Channel | StrictSigner uses non-constant-time comparison | NEW |
| SEC-005 | MEDIUM | Input Validation | P3 witness chain limited to 4 links, no binding to capability | NEW |
| SEC-006 | LOW | Cryptography | Default HMAC key is publicly known | NEW |
| SEC-007 | LOW | Architecture | SignedSecurityGate silently skips unsigned chain links | NEW |
| RES-001 | LOW | Accepted | Generation counter u32 wrap-around (2^32 cycle window) | RESIDUAL |
| RES-002 | LOW | Accepted | Witness record hash fields truncated to 32 bits | RESIDUAL |
| RES-003 | INFO | Accepted | TOCTOU in separate check_budget/record_* paths | RESIDUAL (mitigated) |

**Weighted Score:** 2x HIGH(2) + 3x MEDIUM(1) + 2x LOW(0.5) = 8.0 (meets minimum threshold of 3.0)

---

## Detailed Findings

### SEC-001: FNV-1a Used for Proof Verification Commitments [HIGH]

**Location:** `/workspaces/rvm/crates/rvm-proof/src/lib.rs`, lines 126-133 (`compute_data_hash`), lines 148-160 (`verify`)

**Description:** The `compute_data_hash` function and `verify` function for `ProofTier::Hash` proofs use FNV-1a as the commitment scheme. FNV-1a is a non-cryptographic hash function. An attacker who knows the expected commitment value can trivially compute a preimage that produces the same FNV-1a hash, bypassing Hash-tier proof verification entirely.

Specifically, the verify function computes `FNV-1a(proof.data)` and compares it against `proof.commitment`. Since FNV-1a has no preimage resistance, an attacker can craft arbitrary proof data that matches any given commitment.

```rust
// rvm-proof/src/lib.rs:128-133
pub fn compute_data_hash(data: &[u8]) -> WitnessHash {
    let digest = fnv1a_64(data);  // NON-CRYPTOGRAPHIC
    let mut bytes = [0u8; 32];
    bytes[..8].copy_from_slice(&digest.to_le_bytes());
    WitnessHash::from_bytes(bytes)
}
```

**Impact:** A malicious partition could forge Hash-tier proofs, bypassing the proof-gated mutation requirement. This affects the foundational "no mutation without proof" invariant for all Hash-tier state transitions.

**OWASP:** A02:2021 - Cryptographic Failures

**Recommendation:** Replace `fnv1a_64` with SHA-256 in `compute_data_hash` and the Hash-tier verification path. The `crypto-sha256` feature is already available and used elsewhere. The FNV-1a path should only remain as a non-security hash-table index function, not as a proof commitment scheme.

---

### SEC-002: 8-byte HMAC-SHA256 Truncation in Witness Signatures [HIGH]

**Location:** `/workspaces/rvm/crates/rvm-witness/src/signer.rs`, lines 189-199 (`HmacWitnessSigner::compute_signature`)

**Description:** The witness record `aux` field is only 8 bytes (constrained by the 64-byte record format from ADR-134). HMAC-SHA256 produces a 32-byte MAC, which is truncated to 8 bytes. This reduces the security margin from 2^256 to 2^64 for forgery resistance.

While 2^64 is non-trivial, it is within reach of a well-resourced attacker using birthday-attack techniques. For a system that stakes its entire audit trail integrity on this signature, the margin is narrower than ideal.

```rust
// rvm-witness/src/signer.rs:196-198
let mut sig = [0u8; 8];
sig.copy_from_slice(&tag[..8]);  // 32-byte MAC truncated to 8 bytes
sig
```

**Impact:** An attacker with significant compute resources could forge witness record signatures, potentially tampering with the audit trail to conceal unauthorized mutations.

**OWASP:** A02:2021 - Cryptographic Failures

**Recommendation:** This is a known architectural constraint documented in ADR-134. Short-term: accept the risk with documentation. Medium-term: plan a witness format v2 with wider hash/signature fields (at minimum 16 bytes, ideally 32 bytes). The proof-crate's `WitnessSigner` trait already uses 64-byte signatures -- the bottleneck is the record format. Consider appending a separate signature record that extends the 64-byte record with additional MAC bytes.

---

### SEC-003: Nonce Hash-Index Collision Allows Replay in Edge Cases [MEDIUM]

**Location:** `/workspaces/rvm/crates/rvm-cap/src/verify.rs`, lines 309-323 (`check_nonce`), `/workspaces/rvm/crates/rvm-proof/src/policy.rs`, lines 193-207 (`is_nonce_replayed`)

**Description:** Both the `ProofVerifier` and `PolicyEvaluator` use an O(1) hash-indexed nonce lookup: `nonce_hash[nonce % NONCE_RING_SIZE]`. Two distinct nonces that are congruent modulo 4096 will map to the same hash slot. When nonce A is recorded and then nonce B (where B = A + 4096k) arrives, the hash slot now contains B and the original nonce A is no longer detectable via the hash path.

The watermark provides a floor, but if nonces are not strictly monotonically increasing (e.g., if they are randomly generated), a nonce that was used could be replayed after its hash slot is overwritten by a congruent nonce, provided the watermark hasn't advanced past it.

```rust
// verify.rs:318-319
let hash_slot = (nonce as usize) % NONCE_RING_SIZE;
if self.nonce_hash[hash_slot] == nonce {
    return false;  // Only catches exact match in this slot
}
```

**Impact:** Under specific conditions (non-monotonic nonce generation, high churn), a previously used nonce could bypass replay detection. The monotonic watermark mitigates this for strictly increasing nonces, but the system does not enforce monotonicity.

**OWASP:** A07:2021 - Identification and Authentication Failures

**Recommendation:** Either (a) enforce strictly monotonic nonces and document this requirement, or (b) use a proper hash set (e.g., a cuckoo filter or bloom filter) instead of single-slot hash indexing. Given the `no_std` constraint, a 2-way set-associative hash table would halve the collision probability with minimal space overhead.

---

### SEC-004: StrictSigner Uses Non-Constant-Time Comparison [MEDIUM]

**Location:** `/workspaces/rvm/crates/rvm-witness/src/signer.rs`, lines 81-84 (`StrictSigner::verify`)

**Description:** The `StrictSigner` verify method uses direct byte-array equality (`record.aux == expected`), which compiles to a short-circuiting `memcmp`. This leaks timing information about how many bytes of the signature match, potentially allowing an attacker to forge signatures byte-by-byte.

```rust
// rvm-witness/src/signer.rs:81-84
fn verify(&self, record: &WitnessRecord) -> bool {
    let expected = Self::compute_signature(record);
    record.aux == expected  // NOT constant-time
}
```

In contrast, `HmacWitnessSigner::verify` correctly uses a manual constant-time XOR comparison loop (lines 217-225), and the proof-crate's signers use `ct_eq_64` from the `subtle` crate.

**Impact:** In the fallback configuration (without `crypto-sha256`), `StrictSigner` is the default signer. An attacker with network timing access could progressively determine the correct FNV-1a signature bytes. Combined with FNV-1a's non-cryptographic nature, this significantly reduces the bar for witness record forgery.

**OWASP:** A02:2021 - Cryptographic Failures

**Recommendation:** Replace `record.aux == expected` with a constant-time comparison. Use the same XOR-based pattern already employed in `HmacWitnessSigner::verify`, or import the `subtle` crate:
```rust
fn verify(&self, record: &WitnessRecord) -> bool {
    let expected = Self::compute_signature(record);
    let mut diff = 0u8;
    for i in 0..8 {
        diff |= expected[i] ^ record.aux[i];
    }
    diff == 0
}
```

---

### SEC-005: P3 Witness Chain Limited to 4 Links, No Capability Binding [MEDIUM]

**Location:** `/workspaces/rvm/crates/rvm-security/src/gate.rs`, lines 82-93 (`P3WitnessChain`), lines 111-124 (`verify_p3_chain`)

**Description:** The `P3WitnessChain` struct is limited to 4 links (hard-coded array size), but the maximum delegation depth is 8. This means P3 verification cannot fully validate derivation chains deeper than 4 levels using this structure.

Additionally, the P3 chain verification only checks hash linkage (`link[i].record_hash == link[i+1].prev_hash`) but does not bind the chain to the specific capability being verified. An attacker could present a valid chain from a different capability and it would pass verification.

```rust
// gate.rs:84-86
pub struct P3WitnessChain {
    pub links: [[u64; 2]; 4],  // Only 4 links, but max depth is 8
    pub signatures: [[u8; 8]; 4],
    pub link_count: u8,
}
```

**Impact:** (1) Derivation chains deeper than 4 cannot be fully verified at the gate level. (2) A chain from one valid operation could be reused to authorize a different operation, since the chain is not bound to the specific capability handle or operation context.

**OWASP:** A01:2021 - Broken Access Control

**Recommendation:** (1) Increase the chain link array to match `DEFAULT_MAX_DELEGATION_DEPTH` (8). (2) Add a binding field to `P3WitnessChain` (e.g., the capability index or a hash of the gate request) and verify it during `verify_p3_chain`. (3) Consider including the target_object_id in the chain verification to prevent cross-operation reuse.

---

### SEC-006: Default HMAC Key is Publicly Known [LOW]

**Location:** `/workspaces/rvm/crates/rvm-witness/src/signer.rs`, line 168 (`DEFAULT_KEY_INPUT`)

**Description:** The default HMAC key is derived from `SHA-256(b"rvm-witness-default-key-v1")`, which is a publicly known constant in the source code. While extensively documented with security warnings, there is no runtime enforcement that prevents production use of this key.

The `default_signer()` function returns a signer with this known key, and there is no compile-time or runtime mechanism to detect or prevent its use in production builds.

**Impact:** If deployed without replacing the default key, all witness signatures are forgeable by anyone with source code access. The risk is mitigated by extensive documentation but relies on operator discipline.

**OWASP:** A05:2021 - Security Misconfiguration

**Recommendation:** Add a compile-time feature flag (e.g., `production`) that, when enabled, causes `default_signer()` to fail at compile time or return an error, forcing explicit key provisioning. Alternatively, log a prominent warning at initialization when the default key is detected.

---

### SEC-007: SignedSecurityGate Silently Skips Unsigned Chain Links [LOW]

**Location:** `/workspaces/rvm/crates/rvm-security/src/gate.rs`, lines 322-342 (`verify_chain_signatures`)

**Description:** When verifying P3 witness chain signatures, the `SignedSecurityGate` skips any link with an all-zero signature (treating it as "unsigned for backwards compatibility"). This means an attacker could strip signatures from chain links by zeroing the signature bytes, and the verification would silently accept the chain.

```rust
// gate.rs:327-329
if sig == [0u8; 8] {
    continue;  // Silently skips unsigned links
}
```

**Impact:** An attacker who can modify chain link data in transit could remove signature protection from individual links while the overall chain passes verification. This undermines the "signature covers all links" guarantee.

**OWASP:** A01:2021 - Broken Access Control

**Recommendation:** Add a `require_all_signed` flag to `P3WitnessChain` or to the `SignedSecurityGate` configuration. When set, reject any chain link with a zero signature. The backwards-compatible "skip unsigned" behaviour should only be enabled during migration periods and should be logged as a security event.

---

### RES-001: Generation Counter u32 Wrap-Around [LOW - ACCEPTED RESIDUAL]

**Location:** `/workspaces/rvm/crates/rvm-cap/src/table.rs`, lines 30-33

**Description:** The CapSlot generation counter is u32. After 2^32 allocate/free cycles on a single slot, a stale handle could alias a new capability. The code correctly skips generation 0 on wrap-around.

**Status:** Documented and accepted. Exploitation requires approximately 4 billion operations on a single slot, which is infeasible in practice. Widening to u64 would double CapSlot size.

---

### RES-002: Witness Record Hash Fields Truncated to 32 Bits [LOW - ACCEPTED RESIDUAL]

**Location:** `/workspaces/rvm/crates/rvm-witness/src/log.rs`, lines 86-89

**Description:** The `prev_hash` and `record_hash` fields in `WitnessRecord` are u32, constraining 64-bit chain hashes via XOR-folding. This is an ADR-134 layout constraint (64-byte records, cache-line aligned).

**Status:** Documented with a migration plan. SHA-256 chain hashing (feature `crypto-sha256`) improves entropy quality before folding. A format version bump is planned for wider fields.

---

### RES-003: TOCTOU in Separate check_budget / record_* Paths [INFO - MITIGATED]

**Location:** `/workspaces/rvm/crates/rvm-gpu/src/context.rs`

**Description:** The previous audit flagged a TOCTOU gap between `check_budget()` and `record_*()` calls. This has been **mitigated** with the addition of `try_launch()`, `try_transfer()`, and `try_alloc()` which atomically check AND record with rollback on partial failure (lines 95-121).

**Status:** The separate check/record API still exists for backwards compatibility but the atomic `try_*` methods are available. The TOCTOU gap is mitigated for callers using the new API.

---

## Cryptographic Assessment

| Algorithm | Usage | Assessment |
|-----------|-------|------------|
| **SHA-256** | Attestation chain extension, chain hashing (default), key derivation, record digest | APPROPRIATE. Industry standard, correctly implemented via `sha2` crate. |
| **HMAC-SHA256** | Witness record signing, dual-HMAC signer | APPROPRIATE for symmetric signing. Constant-time verification in `HmacWitnessSigner`. 8-byte truncation is a known limitation (SEC-002). |
| **Ed25519** | Witness signing (optional, `ed25519` feature) | APPROPRIATE. Uses `verify_strict()` per ADR-142 amendment, rejecting non-canonical encodings and small-order keys. Implemented via `ed25519-dalek`. |
| **FNV-1a** | Legacy chain hashing fallback, proof commitment (SEC-001), StrictSigner | INAPPROPRIATE for security-critical paths. Should be confined to hash-table indexing only. |
| **XOR-fold (256-bit to 64-bit)** | Chain hash compression | ACCEPTABLE for entropy preservation within the field-size budget. Four u64 words XORed together preserve collision resistance properties of the underlying hash. |
| **Constant-time comparison** | Attestation verification, HMAC verification, proof signer verification | GOOD. Uses `subtle::ConstantTimeEq` in attestation and proof signers. `HmacWitnessSigner` uses manual XOR loop. `StrictSigner` does NOT use constant-time (SEC-004). |

### Key Derivation

The key derivation scheme (`SHA-256(measurement || partition_id_le || domain_tag)`) uses proper domain separation with distinct tags for witness, attestation, and IPC keys. The construction is sound assuming the TEE measurement provides sufficient entropy. The dev measurement `SHA-256(b"rvm-dev-measurement")` is correctly documented as insecure (SEC-006).

---

## Attack Surface Map

```
External Input Boundaries:
  +-- Hypercall Interface (GateRequest)
  |     +-- CapToken (from untrusted partition)
  |     +-- P3WitnessChain (caller-supplied, SEC-005)
  |     +-- proof_commitment (WitnessHash)
  |     +-- target_object_id, timestamp_ns
  |
  +-- Proof Submission (ProofToken + ProofContext)
  |     +-- Capability handle/generation
  |     +-- Nonce (replay target, SEC-003)
  |     +-- Region bounds, time window
  |     +-- Proof data (up to 64 bytes)
  |
  +-- Capability Operations
  |     +-- grant(source_index, source_generation, rights, badge, target)
  |     +-- revoke(index, generation)
  |     +-- create_root_capability_checked(caller_id)
  |
  +-- GPU Operations (rvm-gpu)
        +-- Kernel launch (compute_ns, transfer_bytes)
        +-- Buffer allocation (size_bytes)
        +-- Transfer requests

Internal Trust Boundaries:
  +-- SecurityGate.check_and_execute() -- single entry point
  +-- ProofEngine.verify_and_witness() -- proof pipeline
  +-- CapabilityManager.verify_p1/p2/p3() -- capability verification
  +-- WitnessLog.append/signed_append() -- audit trail
  +-- GpuContext.try_launch/try_transfer/try_alloc() -- GPU budget
```

---

## Comparison with Previous 11-Finding Audit

| # | Previous Finding | Severity | Current Status | Evidence |
|---|-----------------|----------|----------------|----------|
| 1 | Timing side channels in P1 verification | HIGH | **FIXED** | `verify_p1` uses bitmask accumulation, all checks run unconditionally (verify.rs:111-148). Attestation uses `subtle::ConstantTimeEq` (attestation.rs:208-211). |
| 2 | Revocation bypass (table not synchronized) | CRITICAL | **FIXED** | `revoke_capability` now calls `collect_subtree` and `force_invalidate` on ALL descendants (revoke.rs:37-59). |
| 3 | Memory overlap not checked | HIGH | **FIXED** | `validate_region_bounds` checks page alignment and overflow (validation.rs:43-63). |
| 4 | Generation counter wraps to 0 | MEDIUM | **FIXED** | `invalidate()` skips generation 0 on wrap (table.rs:86-94). Residual 2^32 risk accepted (RES-001). |
| 5 | GRANT_ONCE not consumed | HIGH | **FIXED** | Manager strips GRANT_ONCE after successful grant (manager.rs:324-334). |
| 6 | Missing owner check on grant | HIGH | **FIXED** | `grant_with_caller` verifies caller ownership (manager.rs:282-287). |
| 7 | Slot leak on derivation tree failure | MEDIUM | **FIXED** | Rollback via `force_invalidate` on derivation failure (manager.rs:311-319). |
| 8 | Stack overflow in revocation traversal | MEDIUM | **FIXED** | `revoke_subtree` uses iterative bounded stack (derivation.rs:323-367). |
| 9 | Small nonce ring (64 entries) | MEDIUM | **FIXED** | Expanded to 4096 with monotonic watermark and O(1) hash lookup (verify.rs:16, policy.rs:32). |
| 10 | TOCTOU in capability check | LOW | **MITIGATED** | Gate holds mutex during check+emit. GPU adds atomic `try_*` methods. Residual in separate-API usage (RES-003). |
| 11 | NullSigner available in production | HIGH | **FIXED** | Gated behind `#[cfg(any(test, feature = "null-signer"))]`, deprecated (signer.rs:32-33). |

**Summary:** 8 of 11 findings fully fixed. 3 residual risks documented and accepted with appropriate justification.

---

## Positive Security Observations

1. **`#![forbid(unsafe_code)]` everywhere except HAL.** All 13 non-HAL crates forbid unsafe code at the crate level. The HAL crate (`rvm-hal`) correctly confines `unsafe` to MMIO register access with proper SAFETY comments.

2. **`#![no_std]` and zero heap allocation.** All security-critical crates operate without heap allocation in default configuration, eliminating entire classes of memory safety bugs (use-after-free, double-free, heap overflow).

3. **Defense in depth.** The three-stage security gate (capability check -> policy validation -> witness logging) with the "no witness, no mutation" invariant provides robust defense even if one layer has a weakness.

4. **Constant-time evaluation in P2 policy.** All six P2 rules execute regardless of intermediate failures, preventing timing-based information leakage about which specific check failed (policy.rs:151-182).

5. **Derivation tree integrity.** P3 verification walks the full chain from leaf to root, checking ancestor validity, depth monotonicity, and epoch non-decreasing -- a thorough defence against capability chain manipulation.

6. **GPU budget with atomic rollback.** The `try_launch` method implements proper snapshot-and-rollback semantics, eliminating the TOCTOU gap for callers using the atomic API (context.rs:95-121).

7. **Checked arithmetic throughout.** All budget accounting uses `checked_add` to prevent overflow-based budget bypass (budget.rs, gpu budget.rs).

---

## Recommendations (Prioritized by Risk)

### P0 - Fix Before Next Release

1. **[SEC-001] Replace FNV-1a in proof commitment verification with SHA-256.** The Hash-tier proof system's entire security depends on preimage resistance. FNV-1a provides none. This is the highest-impact finding.

2. **[SEC-004] Add constant-time comparison to StrictSigner.** A simple 3-line fix that closes a timing side-channel in the fallback signer path.

### P1 - Fix in Current Development Cycle

3. **[SEC-003] Strengthen nonce replay detection.** Either enforce monotonic nonces or use 2-way set-associative hashing to reduce collision probability.

4. **[SEC-005] Expand P3WitnessChain to 8 links and add capability binding.** The 4-link limit is below the max delegation depth, and the lack of capability binding weakens P3 verification.

### P2 - Plan for Next Major Version

5. **[SEC-002] Plan witness format v2 with wider signature fields.** The 8-byte truncation is an architectural constraint requiring a format revision.

6. **[SEC-007] Add `require_all_signed` mode to SignedSecurityGate.** Remove the silent skip of unsigned chain links for production deployments.

7. **[SEC-006] Add production-mode enforcement for HMAC key provisioning.** Prevent accidental deployment with the default publicly-known key.

---

## Files Examined

| Crate | File | Lines |
|-------|------|-------|
| rvm-security | src/lib.rs | 131 |
| rvm-security | src/gate.rs | 821 |
| rvm-security | src/validation.rs | 246 |
| rvm-security | src/budget.rs | 442 |
| rvm-security | src/attestation.rs | 348 |
| rvm-cap | src/lib.rs | 62 |
| rvm-cap | src/manager.rs | 566 |
| rvm-cap | src/verify.rs | 570 |
| rvm-cap | src/grant.rs | 220 |
| rvm-cap | src/revoke.rs | 136 |
| rvm-cap | src/table.rs | 449 |
| rvm-cap | src/derivation.rs | 443 |
| rvm-proof | src/lib.rs | 238 |
| rvm-proof | src/engine.rs | 621 |
| rvm-proof | src/policy.rs | 631 |
| rvm-proof | src/signer.rs | 500+ |
| rvm-proof | src/constant_time.rs | 141 |
| rvm-witness | src/lib.rs | 67 |
| rvm-witness | src/log.rs | 347 |
| rvm-witness | src/hash.rs | 182 |
| rvm-witness | src/signer.rs | 503 |
| rvm-witness | src/replay.rs | 165 |
| rvm-witness | src/emit.rs | 138 |
| rvm-witness | src/record.rs | 9 |
| rvm-gpu | src/lib.rs | 121 |
| rvm-gpu | src/budget.rs | 332 |
| rvm-gpu | src/buffer.rs | 153 |
| rvm-gpu | src/context.rs | 419 |
| rvm-hal | src/aarch64/uart.rs | (unsafe audit) |
| rvm-hal | src/aarch64/interrupts.rs | (unsafe audit) |

**Patterns checked:** SQL injection (N/A - no_std), buffer overflow (checked arithmetic throughout), use-after-free (no heap in default config), integer overflow (checked_add/wrapping_add), timing side channels (subtle crate, manual CT loops), secret exposure (no hardcoded secrets beyond documented dev keys), command injection (N/A - no_std), TOCTOU (mutex-guarded critical sections), stack overflow (iterative traversals with bounded stacks), cryptographic misuse (FNV-1a in proof commitments found).

---

*Report generated by QE Security Reviewer (V3) -- Agentic QE v3 security-compliance domain*
