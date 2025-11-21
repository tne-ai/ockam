# Quantum Q-Day Remediation Plan for Ockam
## Post-Quantum Cryptography Migration Strategy

**Document Version:** 1.0  
**Date:** 2025-11-14  
**Project:** Ockam Secure Communication Framework  
**Assessment Reference:** [`ciso5-qday-risk-assessment-ockam.md`](./ciso5-qday-risk-assessment-ockam.md)

---

## Executive Summary

This document provides a comprehensive remediation plan to address quantum computing threats identified in the Ockam codebase. The strategy leverages Ockam's existing vault abstraction architecture to enable a phased migration to NIST-standardized post-quantum cryptography (PQC) algorithms while maintaining backward compatibility and operational continuity.

**Key Recommendations:**
1. Replace X25519 ECDH with **ML-KEM-768** (NIST FIPS 203)
2. Replace Ed25519 signatures with **ML-DSA-65** (NIST FIPS 204)
3. Maintain **AES-256-GCM** for symmetric encryption (quantum-resistant with 128-bit quantum security)
4. Leverage Ockam's **pluggable vault architecture** for seamless PQC integration
5. Implement **hybrid cryptography** during transition period

---

## 1. Vulnerability Summary

Based on the risk assessment, Ockam faces the following quantum threats:

| Component | Current Algorithm | Quantum Risk Level | Quantum Attack | Migration Priority |
|-----------|-------------------|-------------------|----------------|-------------------|
| Key Exchange | X25519 ECDH | **CRITICAL** | Shor's Algorithm | **IMMEDIATE** |
| Digital Signatures | Ed25519 | **CRITICAL** | Shor's Algorithm | **IMMEDIATE** |
| Digital Signatures | ECDSA P-256 | **CRITICAL** | Shor's Algorithm | **IMMEDIATE** |
| Symmetric Encryption | AES-128-GCM | **HIGH** | Grover's Algorithm | **HIGH** |
| Symmetric Encryption | AES-256-GCM | **MEDIUM** | Grover's Algorithm | **ACCEPTABLE** |
| Hashing | SHA-256 | **MEDIUM** | Grover's Algorithm | **MONITOR** |

**Timeline Urgency:**  
With Q-Day estimated between 2028-2040 and "Harvest Now, Decrypt Later" attacks already occurring, migration should begin **immediately** with production deployment targeted for **2025-2026**.

---

## 2. NIST Post-Quantum Cryptography Standards

The National Institute of Standards and Technology (NIST) finalized three PQC standards on August 13, 2024:

### 2.1 ML-KEM (Module-Lattice-Based Key Encapsulation Mechanism)
- **Standard:** FIPS 203
- **Original Name:** CRYSTALS-Kyber
- **Purpose:** Quantum-resistant key exchange
- **Security Basis:** Learning With Errors (LWE) problem on module lattices
- **Replacement For:** X25519, ECDH, RSA key exchange

**Variants:**
- **ML-KEM-512**: 128-bit classical security, 64-bit quantum security
- **ML-KEM-768**: 192-bit classical security, 96-bit quantum security (**RECOMMENDED**)
- **ML-KEM-1024**: 256-bit classical security, 128-bit quantum security

**Reference:**
- NIST FIPS 203: https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.203.pdf
- Wikipedia: https://en.wikipedia.org/wiki/Kyber

### 2.2 ML-DSA (Module-Lattice-Based Digital Signature Algorithm)
- **Standard:** FIPS 204
- **Original Name:** CRYSTALS-Dilithium
- **Purpose:** Quantum-resistant digital signatures
- **Security Basis:** Module lattice problems
- **Replacement For:** Ed25519, ECDSA, RSA signatures

**Variants:**
- **ML-DSA-44**: 128-bit security (2420 byte signatures, 1312 byte keys)
- **ML-DSA-65**: 192-bit security (3293 byte signatures, 1952 byte keys) (**RECOMMENDED**)
- **ML-DSA-87**: 256-bit security (4595 byte signatures, 2592 byte keys)

**Reference:**
- NIST FIPS 204: https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.204.pdf
- PostQuantum.com: https://postquantum.com/post-quantum/cryptography-pqc-nist/

### 2.3 SLH-DSA (Stateless Hash-Based Digital Signature Algorithm)
- **Standard:** FIPS 205
- **Original Name:** SPHINCS+
- **Purpose:** Backup quantum-resistant signatures (hash-based)
- **Security Basis:** Hash functions (not lattices)
- **Use Case:** Backup/fallback if lattice-based schemes are broken

**Reference:**
- NIST FIPS 205: https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.205.pdf
- PostQuantum.com: https://postquantum.com/post-quantum/cryptography-pqc-nist/

### 2.4 Symmetric Encryption Guidance

**NIST Position on AES:**
- AES-128, AES-192, and AES-256 remain suitable for use in post-quantum era
- Grover's algorithm reduces effective security by half (256-bit → 128-bit quantum security)
- **AES-256-GCM provides 128-bit quantum security** — far exceeds NIST's 112-bit minimum
- No immediate need to transition symmetric encryption

**Recommendations:**
- **Continue using AES-256-GCM** for AEAD (current Ockam implementation is quantum-resistant)
- **Upgrade AES-128-GCM to AES-256-GCM** where found
- SHA-256 remains acceptable (provides ~128-bit quantum security with birthday attacks)
- Consider **SHAKE256** or **SHA-512** for future implementations

**References:**
- NIST PQC FAQ: https://csrc.nist.gov/projects/post-quantum-cryptography/faqs
- Certes DPRM White Paper: https://certes.ai/wp-content/uploads/2025/03/Certes-WP-Understanding-Certes-DPRM-AES-256-GCM-and-Quantum-Based-Multi-Part-Key-in-the-Context-of-NIST-PQC-Compliance.pdf
- FreeM indTronic Analysis: https://freemindtronic.com/aes-256-cbc-quantum-security-key-segmentation/

---

## 3. Ockam Vault Architecture Analysis

Ockam's **pluggable vault abstraction** is the key enabler for PQC migration. This architecture provides:

### 3.1 Current Vault Interface
**Location:** `implementations/rust/ockam/ockam_vault/src/`

The vault uses Rust traits to define cryptographic operations:
```rust
// Trait-based abstraction allows multiple implementations
pub trait VaultForSecureChannels {
    fn create_static_key(&self) -> Result<PublicKey>;
    fn create_ephemeral_key(&self) -> Result<PublicKey>;
    fn ecdh(&self, secret: &SecretKey, peer_public_key: &PublicKey) -> Result<SharedSecret>;
}

pub trait VaultForSigning {
    fn sign(&self, secret: &SecretKey, data: &[u8]) -> Result<Signature>;
    fn verify(&self, public_key: &PublicKey, signature: &Signature, data: &[u8]) -> Result<bool>;
}
```

### 3.2 Pluggability Benefits

**Why This Enables Easy PQC Migration:**

1. **Separation of Interface from Implementation**
   - Application code depends on traits, not concrete implementations
   - New PQC vault can implement same interfaces
   - Zero application code changes required

2. **Multiple Vault Support**
   - `SoftwareVault`: Pure Rust implementation
   - `AwsKmsVault`: HSM-backed via AWS KMS (coming in FIPS 203/204 support)
   - `CustomPQCVault`: New PQC-enabled implementation

3. **Runtime Vault Selection**
   - Configuration-driven vault selection
   - No recompilation needed to switch cryptographic backends
   - Enables A/B testing and gradual rollout

4. **Hybrid Cryptography Support**
   - Can run both classical and PQC vaults simultaneously
   - Enables dual-signature schemes during migration
   - Backward compatibility with legacy clients

**Assessment Finding:** Ockam's vault architecture is **exceptionally well-suited** for PQC migration with minimal code disruption.

---

## 4. Remediation Roadmap

### Phase 1: Research and Prototyping (Q1 2025)

**Objective:** Validate PQC library integration with Ockam vault architecture

**Tasks:**
1. Evaluate Rust PQC libraries:
   - **pqcrypto** (https://github.com/rustpq/pqcrypto) - Rust bindings to reference implementations
   - **oqs-rust** - Rust bindings to liboqs (Open Quantum Safe)
   - **Saorsa PQC** (https://github.com/dirvine/saorsa-pqc) - Pure Rust, production-ready, FIPS 203/204/205
   - **AWS-LC** - Amazon's FIPS-validated crypto library with ML-KEM support

2. Create prototype `PQCVault` implementation:
   - Implement `VaultForSecureChannels` with ML-KEM-768
   - Implement `VaultForSigning` with ML-DSA-65
   - Validate interface compatibility

3. Performance benchmarking:
   - ML-KEM vs X25519 key generation/encapsulation
   - ML-DSA vs Ed25519 signing/verification
   - Memory footprint analysis (larger keys/signatures)

**Deliverable:** Technical feasibility report with performance metrics

### Phase 2: Hybrid Implementation (Q2 2025)

**Objective:** Deploy hybrid classical+PQC cryptography for backward compatibility

**Hybrid Key Exchange:**
```rust
// Hybrid KEM: X25519 + ML-KEM-768
pub struct HybridKEM {
    classical: X25519,
    pqc: MLKEM768,
}

impl HybridKEM {
    fn encapsulate(&self, pk: &HybridPublicKey) -> Result<(Ciphertext, SharedSecret)> {
        let (ct_classical, ss_classical) = self.classical.encapsulate(&pk.classical)?;
        let (ct_pqc, ss_pqc) = self.pqc.encapsulate(&pk.pqc)?;
        
        // Combine shared secrets using KDF
        let combined_secret = HKDF::derive(&[ss_classical, ss_pqc])?;
        let combined_ct = Ciphertext::combine(ct_classical, ct_pqc);
        
        Ok((combined_ct, combined_secret))
    }
}
```

**Hybrid Signatures:**
```rust
// Dual signature: Ed25519 + ML-DSA-65
pub struct HybridSignature {
    ed25519_sig: Ed25519Signature,
    mldsa_sig: MLDSA65Signature,
}

// Both must verify for signature to be valid
```

**Benefits:**
- **Security:** Protected even if one algorithm is broken
- **Compatibility:** Works with both PQC and classical-only clients
- **Migration Path:** Gradual transition without breaking existing deployments

**References:**
- IETF Hybrid KEM Draft: https://datatracker.ietf.org/doc/draft-ietf-tls-hybrid-design/
- AWS Hybrid TLS: https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/

### Phase 3: Pure PQC Deployment (Q3-Q4 2025)

**Objective:** Production deployment of pure PQC vault

**Migration Steps:**

1. **Update Noise Protocol Implementation**
   - **Current:** Noise XX with X25519, AES-256-GCM, SHA-256
   - **Target:** Noise XX with ML-KEM-768, AES-256-GCM, SHA-512
   - Location: `implementations/rust/ockam/ockam_identity/src/secure_channel/handshake/handshake.rs`

2. **Update Signing Implementation**
   - Replace Ed25519 with ML-DSA-65
   - Replace ECDSA P-256 with ML-DSA-65 or SLH-DSA
   - Location: `implementations/rust/ockam/ockam_vault/src/software/vault_for_signing/`

3. **Configuration Updates**
   - Add vault selection configuration:
     ```yaml
     vault:
       type: pqc  # or 'hybrid', 'classical'
       kem: ml-kem-768
       signature: ml-dsa-65
       aead: aes-256-gcm
     ```

4. **Testing Strategy**
   - Unit tests for PQC vault operations
   - Integration tests for Noise protocol with PQC
   - Interoperability tests between vault types
   - Performance regression tests
   - Security audit of PQC implementation

**Deliverable:** Production-ready PQC vault with comprehensive test coverage

### Phase 4: Deprecation of Classical Cryptography (2026+)

**Objective:** Phase out classical-only cryptography

**Timeline:**
- **2025 Q4:** Announce deprecation of classical-only mode
- **2026 Q2:** Hybrid mode becomes default
- **2026 Q4:** Pure PQC mode becomes default
- **2027 Q2:** Remove classical-only support (breaking change)

**Communication Plan:**
- Public announcement on Ockam blog
- Migration guide for existing deployments
- Updated documentation and examples
- Deprecation warnings in classical vault

---

## 5. Implementation Details

### 5.1 ML-KEM Integration (Replace X25519)

**Current Implementation:**
```rust
// implementations/rust/ockam/ockam_identity/src/secure_channel/handshake/handshake.rs:309
let dh_result = vault.ecdh(&secret_key, &peer_public_key)?;
```

**PQC Implementation:**
```rust
// Use ML-KEM-768 for key encapsulation
let (ciphertext, shared_secret) = vault.ml_kem_encapsulate(&peer_public_key)?;
```

**Library Recommendation:** `Saorsa PQC` or `pqcrypto`
- **Saorsa:** https://github.com/dirvine/saorsa-pqc (Pure Rust, production-ready, FIPS 203 compliant)
- **pqcrypto:** https://github.com/rustpq/pqcrypto (Rust bindings, well-tested)

**Cargo.toml Addition:**
```toml
[dependencies]
saorsa-pqc = "0.1"  # Or pqcrypto-kem = "0.15"
```

**Reference:**
- Saorsa Benchmark: 82.1/100 quality score, production-ready
- Context7 Library ID: `/dirvine/saorsa-pqc`

### 5.2 ML-DSA Integration (Replace Ed25519)

**Current Implementation:**
```rust
// implementations/rust/ockam/ockam_vault/src/software/vault_for_signing/vault_for_signing.rs:68-76
pub fn sign(&self, secret: &SecretKey, data: &[u8]) -> Result<Signature> {
    let signing_key = SigningKey::from_bytes(&secret.key)?;
    let signature = signing_key.sign(data);
    Ok(Signature::from_bytes(signature.to_bytes()))
}
```

**PQC Implementation:**
```rust
pub fn sign(&self, secret: &MLDSASecretKey, data: &[u8]) -> Result<MLDSASignature> {
    let signing_key = MLDSA65::from_bytes(&secret.key)?;
    let signature = signing_key.sign(data)?;
    Ok(MLDSASignature::from_bytes(signature.to_bytes()))
}
```

**Considerations:**
- **Signature Size:** ML-DSA-65 produces ~3.3KB signatures (vs 64 bytes for Ed25519)
- **Public Key Size:** ~2KB (vs 32 bytes for Ed25519)
- **Performance:** Slightly slower signing/verification, but acceptable for most use cases

**Reference:**
- IBM Research Analysis: https://research.ibm.com/blog/nist-pqc-standards
- NIST FIPS 204 Draft: https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.204.pdf

### 5.3 AES-256-GCM Retention (Quantum-Safe)

**Current Implementation:** **ALREADY QUANTUM-RESISTANT**
```rust
// implementations/rust/ockam/ockam_vault/src/software/vault_for_secure_channels/aes_rs.rs:64
AES_256_GCM => {
    let cipher = Aes256Gcm::new(key);
    // ... encryption logic
}
```

**No Changes Required:**
- AES-256-GCM provides **128-bit quantum security** (Grover's algorithm reduces from 256→128 bits)
- Exceeds NIST's 112-bit minimum security requirement
- **NSA Commercial National Security Algorithm Suite (CNSA 2.0)** approved

**Upgrade Path for AES-128:**
```rust
// Replace any AES-128-GCM usage with AES-256-GCM
// implementations/rust/ockam/ockam_vault/src/software/vault_for_secure_channels/aes_rs.rs:77
- AES_128_GCM => { ... }  // REMOVE
+ // All encryption now uses AES_256_GCM
```

**Reference:**
- NIST Guidance: https://csrc.nist.gov/projects/post-quantum-cryptography/faqs
- Grover Analysis: https://community.bitwarden.com/t/quantum-resistant-encryption/58534

---

## 6. Rust PQC Library Recommendations

Based on Context7 search and ecosystem analysis:

| Library | Pros | Cons | Recommendation |
|---------|------|------|----------------|
| **Saorsa PQC** | Pure Rust, FIPS 203/204/205, production-ready, 82.1 quality score | Newer library | ⭐ **RECOMMENDED** for production |
| **pqcrypto** | Well-tested, comprehensive | C bindings, build complexity | Good for prototyping |
| **oqs-rust** | Open Quantum Safe, industry-backed | C dependencies | Alternative option |
| **AWS-LC** | FIPS-validated, Amazon-backed | Less Rust-native | Good for AWS deployments |

**Selected Library:** **Saorsa PQC**
- Context7 ID: `/dirvine/saorsa-pqc`
- GitHub: https://github.com/dirvine/saorsa-pqc
- Features: ML-KEM (Kyber), ML-DSA (Dilithium), SLH-DSA (SPHINCS+)
- Validation: FIPS 203, 204, 205 compliant
- Quality: Benchmark score 82.1/100

---

## 7. Testing and Validation Strategy

### 7.1 Functional Testing
- [ ] PQC vault implements all trait methods correctly
- [ ] ML-KEM key exchange produces valid shared secrets
- [ ] ML-DSA signatures verify correctly
- [ ] Hybrid mode works with both PQC and classical clients
- [ ] Configuration switching between vault types works

### 7.2 Security Testing
- [ ] Independent security audit of PQC implementation
- [ ] Fuzzing of PQC cryptographic operations
- [ ] Side-channel attack resistance analysis
- [ ] Compliance validation against FIPS 203/204/205

### 7.3 Performance Testing
- [ ] Benchmark key generation, signing, verification
- [ ] Memory footprint analysis (larger keys/signatures)
- [ ] Network bandwidth impact (larger messages)
- [ ] Latency impact on secure channel establishment
- [ ] Throughput testing at scale

### 7.4 Interoperability Testing
- [ ] Cross-platform compatibility (Rust, Elixir, Python implementations)
- [ ] Version negotiation between PQC and classical
- [ ] Backward compatibility with existing Ockam deployments

---

## 8. Risk Mitigation

### 8.1 Migration Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| PQC library bugs | Medium | High | Use well-vetted libraries (Saorsa, AWS-LC), extensive testing |
| Performance degradation | High | Medium | Benchmark early, optimize, consider hardware acceleration |
| Breaking changes | High | High | Hybrid mode for gradual transition, version negotiation |
| Increased message size | High | Medium | Compression, network optimization, document bandwidth requirements |

### 8.2 Rollback Strategy

**If PQC deployment fails:**
1. Configuration-based rollback to classical or hybrid vault
2. No code changes required (vault abstraction benefit)
3. Monitoring alerts trigger automatic fallback
4. Investigate issues in isolated environment

**Example Configuration:**
```yaml
vault:
  type: classical  # Rollback from 'pqc' to 'classical'
  enable_monitoring: true
  fallback_on_error: true
```

---

## 9. Documentation Updates

### 9.1 Required Documentation
- [ ] PQC migration guide for existing Ockam users
- [ ] Vault configuration reference (classical vs hybrid vs PQC)
- [ ] Performance characteristics comparison
- [ ] Security considerations and threat model
- [ ] FAQ addressing PQC questions

### 9.2 Code Examples
- [ ] Example of creating PQC-enabled secure channel
- [ ] Hybrid vault configuration example
- [ ] Migration script for existing deployments
- [ ] Testing utilities for PQC validation

---

## 10. Compliance and Certification

### 10.1 NIST Compliance
- ✅ ML-KEM (FIPS 203) - NIST standardized August 2024
- ✅ ML-DSA (FIPS 204) - NIST standardized August 2024
- ✅ SLH-DSA (FIPS 205) - NIST standardized August 2024
- ✅ AES-256-GCM - Already FIPS 140-3 validated

### 10.2 Industry Standards
- [ ] CNSA 2.0 (NSA Commercial National Security Algorithm Suite)
- [ ] IETF TLS 1.3 with PQC (draft specification)
- [ ] NIST SP 800-227 (Key Combiner Recommendations)

**References:**
- NIST PQC Project: https://csrc.nist.gov/projects/post-quantum-cryptography
- NSA CNSA 2.0: https://media.defense.gov/2022/Sep/07/2003071834/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS_.PDF

---

## 11. Cost-Benefit Analysis

### 11.1 Implementation Costs
- **Development:** 3-6 months (including testing and validation)
- **Performance:** 10-30% overhead for PQC operations
- **Bandwidth:** 2-5x increase in key exchange messages (larger keys/ciphertexts)
- **Storage:** Minimal (larger keys stored, but not significant)

### 11.2 Benefits
- **Security:** Protection against quantum computers (Q-Day resilience)
- **Compliance:** Meet future regulatory requirements
- **Competitive Advantage:** Early adopter of PQC
- **Future-Proofing:** Avoid costly emergency migration later
- **Reputation:** Demonstrate security leadership

### 11.3 Cost of Inaction
- **Harvest Now, Decrypt Later:** Adversaries collecting encrypted traffic today
- **Emergency Migration:** 10x cost if forced to migrate under pressure
- **Data Breach:** Catastrophic if quantum decryption occurs
- **Regulatory Fines:** Non-compliance with future PQC mandates

**Recommendation:** Benefits far outweigh costs. Begin migration immediately.

---

## 12. Long-Term Maintenance

### 12.1 Algorithm Agility
- Vault abstraction enables easy algorithm swaps
- Configuration-driven cryptographic selection
- Monitor NIST guidance for algorithm updates
- Plan for potential algorithm deprecations

### 12.2 Monitoring and Alerts
- Track vault type usage metrics
- Monitor PQC operation success/failure rates
- Alert on performance degradation
- Dashboard for deployment status

### 12.3 Regular Reviews
- Annual security audit of PQC implementation
- Quarterly review of NIST PQC updates
- Participate in PQC community (IETF, NIST workshops)
- Update libraries as new versions release

---

## 13. Conclusion

Ockam's pluggable vault architecture provides an **exceptional foundation** for post-quantum cryptography migration. The phased approach outlined in this document enables:

1. **Risk Mitigation:** Hybrid mode ensures backward compatibility during transition
2. **Minimal Disruption:** Vault abstraction means zero application code changes
3. **Future-Proofing:** Configuration-driven cryptography selection
4. **Compliance:** NIST FIPS 203/204/205 standardized algorithms
5. **Security:** Protection against quantum threats estimated to arrive 2028-2040

**Immediate Next Steps:**
1. Approve PQC migration roadmap
2. Allocate engineering resources for Q1 2025 prototyping
3. Select Rust PQC library (recommend: Saorsa PQC)
4. Begin performance benchmarking
5. Draft public communication plan

**Success Criteria:**
- Production PQC deployment by Q4 2025
- Zero breaking changes for existing users (via hybrid mode)
- Performance overhead <30%
- Full NIST FIPS 203/204/205 compliance
- Independent security audit passed

---

## Appendix A: Key File Locations

### Code Requiring PQC Updates

| File | Line(s) | Current | Target | Priority |
|------|---------|---------|--------|----------|
| `implementations/rust/ockam/ockam_identity/src/secure_channel/handshake/handshake.rs` | 309 | X25519 ECDH | ML-KEM-768 | **CRITICAL** |
| `implementations/rust/ockam/ockam_vault/src/software/vault_for_signing/vault_for_signing.rs` | 68-76 | Ed25519 | ML-DSA-65 | **CRITICAL** |
| `implementations/rust/ockam/ockam_vault/src/software/vault_for_signing/vault_for_signing.rs` | 78-88 | ECDSA P-256 | ML-DSA-65 | **CRITICAL** |
| `implementations/rust/ockam/ockam_vault/src/software/vault_for_secure_channels/aes_rs.rs` | 77 | AES-128-GCM | AES-256-GCM | **HIGH** |
| `implementations/rust/ockam/ockam_vault/src/software/vault_for_secure_channels/aes_rs.rs` | 64 | AES-256-GCM | *(No Change)* | **COMPLIANT** |

### Dependencies to Add

| Crate | Version | Purpose | Cargo.toml Location |
|-------|---------|---------|---------------------|
| `saorsa-pqc` | 0.1+ | ML-KEM, ML-DSA, SLH-DSA | `implementations/rust/ockam/ockam_vault/Cargo.toml` |

---

## Appendix B: References and Citations

### NIST Standards
1. NIST FIPS 203 (ML-KEM): https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.203.pdf
2. NIST FIPS 204 (ML-DSA): https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.204.pdf  
3. NIST FIPS 205 (SLH-DSA): https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.205.pdf
4. NIST PQC FAQ: https://csrc.nist.gov/projects/post-quantum-cryptography/faqs
5. NIST IR 8547 (Transition Guide): https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf

### Industry Resources
6. PostQuantum.com PQC Overview: https://postquantum.com/post-quantum/cryptography-pqc-nist/
7. IBM Research NIST PQC Standards: https://research.ibm.com/blog/nist-pqc-standards
8. AWS PQC Migration Plan: https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/
9. AppViewX NIST PQC Standards: https://www.appviewx.com/blogs/nist-announces-the-first-3-post-quantum-cryptography-standards-ready-or-not/

### Symmetric Encryption
10. Certes AES-256-GCM PQC Compliance: https://certes.ai/wp-content/uploads/2025/03/Certes-WP-Understanding-Certes-DPRM-AES-256-GCM-and-Quantum-Based-Multi-Part-Key-in-the-Context-of-NIST-PQC-Compliance.pdf
11. FreeMindTronic AES-256 Quantum Security: https://freemindtronic.com/aes-256-cbc-quantum-security-key-segmentation/
12. Bitwarden Quantum Resistant Encryption: https://community.bitwarden.com/t/quantum-resistant-encryption/58534
13. Cryptography Stack Exchange AES-128: https://crypto.stackexchange.com/questions/102671/is-aes-128-quantum-safe

### Rust Libraries
14. Saorsa PQC: https://github.com/dirvine/saorsa-pqc (Context7: `/dirvine/saorsa-pqc`)
15. pqcrypto: https://github.com/rustpq/pqcrypto (Context7: `/rustpq/pqcrypto`)
16. Kyber Wikipedia: https://en.wikipedia.org/wiki/Kyber

### Implementation References
17. Xiphera ML-KEM IP Core: https://xiphera.com/post-quantum-cryptography/ml-kem-key-encapsulation-mechanism/
18. ArXiv Kyber ESP32 Implementation: https://arxiv.org/html/2503.10207v1

---

**Document Prepared By:** CISO5 Quantum Q-Day Assessment Team  
**Review Date:** 2025-11-14  
**Next Review:** 2025-Q2 (or upon NIST guidance updates)