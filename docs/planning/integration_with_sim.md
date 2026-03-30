# VCSD-SimpleSSD Integration Plan
**Goal**: Integrate VCSD cryptographic secure deletion with SimpleSSD for performance evaluation against state-of-the-art secure deletion algorithms

---

## Phase 1: AES-GCM Encryption Layer (Week 1-2)

### 1.1 **Create Encryption/Deletion Manager (EDM)**
**Location**: `vcsd/core/edm/`

**Files to create**:
- `encryption_manager.hh/cc` - AES-256-GCM encryption/decryption
- `deletion_policy.hh/cc` - Policy definitions (VCSD, CRYPTOERASE, OVERWRITE, SANITIZE)
- `baseline_algorithms.hh/cc` - Implementations of comparison algorithms

**Functionality**:
```cpp
class EncryptionManager {
    // Actual AES-GCM encryption (not just simulation)
    bool encryptPage(const uint8_t* pageKey, const uint8_t* plaintext, 
                     size_t len, uint8_t* ciphertext, uint8_t* tag);
    
    bool decryptPage(const uint8_t* pageKey, const uint8_t* ciphertext,
                     size_t len, const uint8_t* tag, uint8_t* plaintext);
    
    // Integrate with existing PuncturablePRF
    PuncturablePRF* prf_;
};
```

**Dependencies**:
- OpenSSL EVP_aes_256_gcm()
- Existing PuncturablePRF for key derivation
- Use master key from config (not just stored but actively used)

---

## Phase 2: FTL Integration (Week 2-3)

### 2.1 **Extend FTL with Crypto Operations**
**Location**: `simplessd/ftl/` (modify existing)

**Files to modify**:
- `page_mapping.cc` - Add encryption on write, decryption on read
- `page_mapping.hh` - Add EDM pointer and crypto interfaces
- `ftl.hh` - Add VCSD parameters

**Key Changes**:
1. **Write Path**:
   ```cpp
   void PageMapping::writeIntern al(Request &req, uint64_t &tick, bool isGC) {
       // Derive page key from LPN
       uint8_t pageKey[32];
       edm_->derivePageKey(req.lpn, version, pageKey);
       
       // Encrypt data before writing to flash
       uint8_t ciphertext[PAGE_SIZE];
       uint8_t tag[16];
       edm_->encryptPage(pageKey, plaintext, PAGE_SIZE, ciphertext, tag);
       
       // Store ciphertext + tag to flash via PAL
       // ...existing write logic...
   }
   ```

2. **Read Path**:
   ```cpp
   void PageMapping::readInternal(Request &req, uint64_t &tick) {
       // Read ciphertext from flash
       // ...existing read logic...
       
       // Derive page key from LPN
       uint8_t pageKey[32];
       if (!edm_->derivePageKey(req.lpn, version, pageKey)) {
           // Key was punctured - return error
           return DELETED_DATA_ERROR;
       }
       
       // Decrypt data after reading from flash
       edm_->decryptPage(pageKey, ciphertext, PAGE_SIZE, tag, plaintext);
   }
   ```

3. **Trim Path** (Secure Deletion):
   ```cpp
   void PageMapping::trimInternal(Request &req, uint64_t &tick) {
       if (secureDeleteEnabled && req.secureDeleteFlag) {
           switch (deletionPolicy) {
               case POLICY_VCSD:
                   // VCSD: Puncture file ID (instant)
                   edm_->punctureFileID(req.fileID);
                   tick += PUNCTURE_LATENCY;
                   break;
               
               case POLICY_CRYPTOERASE:
                   // CryptoErase: Delete key from key manager
                   edm_->deleteKey(req.lpn);
                   tick += KEY_DELETE_LATENCY;
                   break;
               
               case POLICY_OVERWRITE:
                   // Traditional: Overwrite with zeros
                   performOverwrite(req, tick);
                   break;
               
               case POLICY_SANITIZE:
                   // Sanitize: Cryptographic erase + physical erase
                   edm_->deleteKey(req.lpn);
                   performErase(req, tick);
                   break;
           }
       } else {
           // Normal trim - just invalidate mapping
           invalidateMapping(req.lpn);
       }
   }
   ```

### 2.2 **Metadata Storage**
**Add to FTL**:
- FileID to LPN mapping (which pages belong to which file)
- Version tracking per LPN (for rewrites)
- Encryption metadata (IV, tag) storage alongside data

---

## Phase 3: Baseline Algorithms Implementation (Week 3-4)

### 3.1 **Implement Comparison Algorithms**
**Location**: `vcsd/core/edm/baseline_algorithms.cc`

**Algorithms to implement**:

1. **Traditional Overwrite** (NIST 800-88 compliant)
   - Single-pass overwrite with zeros
   - Multi-pass overwrite (3-pass, 7-pass options)
   - Measure: Time, write amplification, wear

2. **CryptoErase** (Key-based deletion)
   - Per-chunk key management (coarser than VCSD)
   - Key deletion for secure deletion
   - Compare with VCSD granularity

3. **Sanitize** (Combined crypto + physical)
   - Key deletion + block erase
   - Full erase verification
   - High security but high cost

4. **Secure Erase** (ATA/NVMe native)
   - Simulate vendor-specific secure erase
   - Block-level erase operations
   - Measure latency and effectiveness

### 3.2 **Policy Configuration**
**Location**: `config/vcsd_policies.cfg`

```ini
[SecureDeletion]
# Policy: VCSD | CRYPTOERASE | OVERWRITE | SANITIZE | SECUREERASE
Policy = VCSD

[VCSD]
# File-level granularity
KeyHierarchy = Master->File->Page
PunctureLatency = 500ns

[CryptoErase]
# Chunk-level granularity (e.g., 1MB chunks)
ChunkSize = 1048576
KeyPerChunk = true

[Overwrite]
# Number of overwrite passes
Passes = 1
Pattern = ZEROS  # ZEROS | RANDOM | NIST
```

---

## Phase 4: Performance Measurement Framework (Week 4-5)

### 4.1 **Benchmarking Infrastructure**
**Location**: `vcsd/benchmark/`

**Files to create**:
- `benchmark_runner.hh/cc` - Run experiments with different policies
- `metrics_collector.hh/cc` - Collect performance metrics
- `trace_generator.hh/cc` - Generate realistic workloads

**Metrics to collect**:
```cpp
struct DeletionMetrics {
    uint64_t deletionLatency;      // Time to complete deletion
    uint64_t verificationTime;      // Time to verify deletion
    uint64_t writeAmplification;    // Extra writes caused
    uint64_t energyConsumption;     // Energy cost
    uint64_t wearLeveling;          // Impact on flash wear
    bool dataRecoverable;           // Security verification
};
```

### 4.2 **Workload Scenarios**
1. **File deletion workload**: Create files, delete them, measure
2. **Mixed workload**: Read/write with periodic deletions
3. **Database workload**: Transaction logs with frequent deletes
4. **Cloud storage**: Large files with infrequent deletes

### 4.3 **Comparison Metrics**
- **Performance**: Deletion latency (μs)
- **Efficiency**: Write amplification factor
- **Durability**: Flash wear impact (P/E cycles)
- **Security**: Data recoverability (forensic analysis)
- **Scalability**: Performance with increasing file count

---

## Phase 5: Integration & Testing (Week 5-6)

### 5.1 **CMakeLists.txt Updates**
Add EDM module to build system:
```cmake
# EDM sources
set(EDM_SOURCES
    vcsd/core/edm/encryption_manager.cc
    vcsd/core/edm/deletion_policy.cc
    vcsd/core/edm/baseline_algorithms.cc
)

# Link with SimpleSSD FTL
target_link_libraries(simplessd-ftl vcsd_edm OpenSSL::Crypto)
```

### 5.2 **Test Suite**
**Location**: `vcsd/tests/integration/`

1. **Unit tests**: Each deletion policy independently
2. **Integration tests**: Full SSD simulation with workloads
3. **Security tests**: Verify data is truly irrecoverable
4. **Performance tests**: Compare all algorithms

### 5.3 **Validation**
- Functional correctness: All data operations work with encryption
- Security validation: Deleted data cannot be recovered
- Performance validation: VCSD outperforms baselines
- Wear validation: Flash lifetime not significantly impacted

---

## Phase 6: Experimental Evaluation (Week 6-7)

### 6.1 **Experimental Setup**
**Configuration**:
- SSD: 256GB capacity, 4KB page size
- NAND: TLC, 3000 P/E cycles
- Workloads: FileBench, YCSB, TPC-C
- Policies: VCSD vs CryptoErase vs Overwrite vs Sanitize

### 6.2 **Key Research Questions**
1. **RQ1**: How does VCSD deletion latency compare to traditional methods?
   - Hypothesis: VCSD is 100-1000× faster

2. **RQ2**: What is the write amplification of different policies?
   - Hypothesis: VCSD has near-zero write amplification

3. **RQ3**: How does deletion policy affect overall SSD lifetime?
   - Hypothesis: VCSD minimizes flash wear

4. **RQ4**: What is the performance overhead of encryption?
   - Hypothesis: AES-GCM with AES-NI has <5% overhead

5. **RQ5**: How does file granularity affect deletion efficiency?
   - Compare: VCSD (per-file) vs CryptoErase (per-chunk)

### 6.3 **Expected Results**
- **Deletion Latency**: VCSD ~0.5μs vs Overwrite ~10ms (20,000× speedup)
- **Write Amplification**: VCSD 1.0× vs Overwrite 3-5×
- **Flash Wear**: VCSD minimal vs Overwrite significant
- **Security**: All crypto-based methods provide provable security

---

## Implementation Priority

### **Critical Path (Must Have)**:
1. ✅ AES-GCM encryption/decryption implementation
2. ✅ Master key usage in encryption (not just derivation)
3. ✅ FTL write path integration
4. ✅ FTL read path integration
5. ✅ VCSD puncture-based deletion

### **High Priority (Should Have)**:
6. CryptoErase baseline implementation
7. Traditional overwrite implementation
8. Performance measurement framework
9. Basic benchmarking workloads

### **Medium Priority (Nice to Have)**:
10. Sanitize policy implementation
11. Advanced workload traces
12. Energy consumption modeling
13. Comprehensive test suite

### **Low Priority (Future Work)**:
14. Hardware accelerator simulation (AES-NI)
15. Multi-tenancy support
16. Remote verification protocol
17. Real-world trace replay

---

## File Structure (After Implementation)

```
vcsd/
├── core/
│   ├── ppk/              (existing - Puncturable PRF)
│   ├── crypto/           (existing - latency models)
│   ├── edm/              (NEW - Encryption/Deletion Manager)
│   │   ├── encryption_manager.hh/cc
│   │   ├── deletion_policy.hh/cc
│   │   └── baseline_algorithms.hh/cc
│   ├── merkle/           (existing - PCD support)
│   └── pcd/              (existing - receipts)
├── interface/
│   ├── vcsd_controller.hh/cc  (modify - add EDM integration)
│   └── trace_parser.hh/cc     (existing)
├── benchmark/            (NEW)
│   ├── benchmark_runner.hh/cc
│   ├── metrics_collector.hh/cc
│   └── workload_generator.hh/cc
├── tests/
│   ├── integration/      (NEW - full system tests)
│   └── ...existing tests...
└── INTEGRATION_PLAN.md   (this file)

simplessd/
└── ftl/
    ├── page_mapping.hh/cc    (modify - add crypto)
    ├── ftl.hh/cc             (modify - add EDM)
    └── ...existing files...
```

---

## Next Steps (Immediate Actions)

1. **Create EDM module structure**
   ```bash
   mkdir -p vcsd/core/edm
   mkdir -p vcsd/benchmark
   mkdir -p vcsd/tests/integration
   ```

2. **Implement AES-GCM encryption layer**
   - Start with `encryption_manager.cc`
   - Use OpenSSL EVP API
   - Test with known vectors

3. **Modify FTL write path**
   - Add encryption before PAL write
   - Store ciphertext + authentication tag

4. **Modify FTL read path**
   - Add decryption after PAL read
   - Verify authentication tag

5. **Implement secure trim**
   - VCSD: puncture file ID
   - Baseline: overwrite/erase

6. **Build test workload**
   - Simple write→read→delete cycle
   - Measure latencies

7. **Iterate and optimize**
   - Profile performance
   - Compare with baselines
   - Refine implementation

---

## Success Criteria

- ✅ All data written is encrypted with AES-256-GCM
- ✅ Master key is actually used for encryption (not just stored)
- ✅ VCSD deletion works via puncturing (instant)
- ✅ Baseline algorithms implemented and working
- ✅ Performance measurements show VCSD advantages
- ✅ Integration with SimpleSSD FTL complete
- ✅ Test suite passes all tests
- ✅ Research paper has quantitative comparison data
