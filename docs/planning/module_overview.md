# V-CSD Module Overview & Interactions

## Quick Reference

### Core Algorithm Modules (Priority 1)

#### 1. **vcsd_ppk** (Per-Page Key Manager)
- **Purpose**: Puncturable key tree for no-migration crypto deletion
- **Key Functions**:
  - `DeriveLeafKey(chunk_id, page_index) → key[32]`: HKDF-based key derivation
  - `Puncture(chunk_id, leaves[])`: Rotate ancestors, make keys unrecoverable
  - `CommitDigests(chunk_id) → digest[16]`: Compact tree representation
- **Dependencies**: OpenSSL (HKDF, SHA256)
- **Phase**: 1.5-1.6

#### 2. **vcsd_pcd** (Proof-Carrying Deletion)
- **Purpose**: Generate cryptographic receipts proving deletion
- **Key Functions**:
  - `Sign(snapshot, plan, health, fw_ver) → receipt`: Build & sign receipt
- **Receipt Contents**: scope, pre_state Merkle, actions, post_state Merkle, signature
- **Dependencies**: vcsd_merkle, vcsd_crypto_stub, nlohmann/json
- **Phase**: 2.2-2.4

#### 3. **vcsd_chunk_manager** (State Management)
- **Purpose**: Track chunk↔block mapping and page states
- **Key Functions**:
  - `SnapshotOf(scope) → snapshot`: Capture pre-deletion state
  - `UpdateOnWrite(lpn, old_ppn, new_ppn)`: Maintain page state bitmaps
  - `Meta(chunk_id) → chunk_metadata`: Access chunk info
- **State**: PageStateBitmap (free/valid/invalid), ChunkMapEntry (blocks, key_page, merkle_root)
- **Phase**: 1.3-1.4

### Planning & Execution Modules (Priority 2)

#### 4. **vcsd_planner** (Hybrid Optimization Planner)
- **Purpose**: Decide optimal mix of PPK puncture, erasure, and BDS
- **Key Functions**:
  - `Build(scope, health, quotas) → plan`: Generate deletion plan
- **Strategies**:
  - **PPK-only**: No migrations, pure crypto deletion
  - **Hybrid**: Warm-start biclique (rows=PPK, columns=erasure)
  - **BDS**: Optional fast sanitization if ECC permits
- **Dependencies**: vcsd_chunk_manager, vcsd_ecc
- **Phase**: 3.2 (PPK-only), 4.2-4.3 (hybrid)

#### 5. **vcsd_deletion** (Execution Engine)
- **Purpose**: Execute planned deletions
- **Key Functions**:
  - `ExecPpk(leaves[])`: Puncture PPK keys
  - `ExecErase(blocks[])`: Migrate valid pages, erase blocks
  - `ExecBds(targets[])`: Run bounded disturb sanitization
- **Dependencies**: vcsd_ppk, FTL, PAL
- **Phase**: 3.3 (PPK), 4.4 (erase), 5.1 (BDS)

### Supporting Modules (Priority 3)

#### 6. **vcsd_ecc** (ECC Monitor)
- **Purpose**: Track NAND health and gate BDS operations
- **Key Functions**:
  - `Measure(plane, block) → margins`: Get ECC margins (tmax, tobs, llr_mean)
  - `AcceptAfterBds(block, before, after) → bool`: Verify safe after BDS
- **Phase**: 4.5 (basic), 5.1 (full)

#### 7. **vcsd_merkle** (Header-Only Utilities)
- **Purpose**: Merkle tree for state commitments
- **Key Functions**:
  - `MerkleRoot(leaves[]) → root[32]`: Compute SHA256 Merkle root
- **Dependencies**: OpenSSL (SHA256)
- **Phase**: 1.2

#### 8. **vcsd_journal** (Crash Consistency)
- **Purpose**: Ensure atomic deletion operations
- **Key Functions**:
  - `Append(record)`: Log operation markers
  - `Replay()`: Recover from crashes
- **Markers**: SNAPSHOT_TAKEN, PLAN_COMMITTED, ACTION_APPLIED, RECEIPT_SIGNED
- **Phase**: 5.2

#### 9. **vcsd_sdt** (Secure Deletion Transport)
- **Purpose**: Abstraction over NVMe/SATA/SCSI interfaces
- **Key Functions**:
  - `Send(op, payload_in, payload_out) → status`: Execute admin command
- **Operations**: DELETE_SELECTIVE, DELETE_COMMIT, GET_PCD, GET_HEALTH
- **Phase**: 5.3

#### 10. **vcsd_nvme_admin** (NVMe Integration)
- **Purpose**: NVMe vendor admin commands
- **Commands**: 
  - DELETE_SELECTIVE: Initiate deletion with scope/SLO
  - GET_PCD: Retrieve signed receipt
  - GET_HEALTH: Query ECC margins, P/E distribution
- **Phase**: 5.4

---

## Data Flow: Deletion Request

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOST APPLICATION                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │ DELETE_SELECTIVE(scope, SLO)
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   NVMe Admin Command Handler                     │
│                      (vcsd_nvme_admin.cc)                        │
└──────────────────┬──────────────────────────────────────────────┘
                   │ Parse payload, validate
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Config Manager                              │
│                     (vcsd_config.cc)                             │
│                  - Load quotas, policies                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │ Authorized, get quotas
                   ▼
╔═════════════════════════════════════════════════════════════════╗
║                    PHASE 1: SNAPSHOT                             ║
╠═════════════════════════════════════════════════════════════════╣
║  Chunk Manager: SnapshotOf(scope)                               ║
║    - Identify chunks covering scope                              ║
║    - Capture PageStateBitmap (valid/invalid marks)              ║
║    - Collect ChunkMapEntry (blocks, key_page, merkle_root)      ║
║  Merkle: MerkleRoot(chunk_bitmaps) → pre_state_root             ║
║  Journal: Append(SNAPSHOT_TAKEN)                                ║
╚═════════════════════════════════════════════════════════════════╝
                   │ snapshot
                   ▼
╔═════════════════════════════════════════════════════════════════╗
║                    PHASE 2: PLANNING                             ║
╠═════════════════════════════════════════════════════════════════╣
║  ECC Monitor: Measure(blocks) → health                          ║
║  Planner: Build(snapshot, health, quotas) → plan                ║
║    PPK-only mode (Phase 1-3):                                   ║
║      - plan.punctures = all invalid pages                        ║
║    Hybrid mode (Phase 4):                                       ║
║      - Warm-start biclique heuristic                            ║
║      - High valid density → PPK (rows)                          ║
║      - High invalid density → Erase (columns)                   ║
║      - plan.punctures = selected leaves                          ║
║      - plan.erasures = selected blocks                           ║
║    Full mode (Phase 5):                                         ║
║      - plan.bds = targets if ECC permits & SLO tight            ║
║  Journal: Append(PLAN_COMMITTED)                                ║
╚═════════════════════════════════════════════════════════════════╝
                   │ plan
                   ▼
╔═════════════════════════════════════════════════════════════════╗
║                   PHASE 3: EXECUTION                             ║
╠═════════════════════════════════════════════════════════════════╣
║  Deletion Engine:                                               ║
║    ExecPpk(plan.punctures):                                     ║
║      - PPK.Puncture(chunk, leaves) → rotate ancestor seeds      ║
║      - Update key_page with new digests                         ║
║    ExecErase(plan.erasures):                                    ║
║      - Migrate minimal valid pages to new blocks                ║
║      - Erase old blocks via PAL                                 ║
║      - Update FTL mapping                                       ║
║    ExecBds(plan.bds):                                           ║
║      - BDS.Run(target) → pre/post ECC margins                   ║
║      - ECC.AcceptAfterBds() → verify safe                       ║
║  Chunk Manager: UpdateMetadata() → recompute merkle_roots       ║
║  Merkle: MerkleRoot(updated_chunks) → post_state_root           ║
║  Journal: Append(ACTION_APPLIED)                                ║
╚═════════════════════════════════════════════════════════════════╝
                   │ execution_result
                   ▼
╔═════════════════════════════════════════════════════════════════╗
║                  PHASE 4: RECEIPT GENERATION                     ║
╠═════════════════════════════════════════════════════════════════╣
║  PCD: Sign(snapshot, plan, health, fw_ver) → receipt            ║
║    Receipt JSON:                                                 ║
║      {                                                           ║
║        "req_id": "...",                                          ║
║        "scope": { "type": "LBA", "ranges": [...] },             ║
║        "pre_state": { "merkle_root": "..." },                   ║
║        "actions": {                                              ║
║          "ppk_punctures": [...],                                 ║
║          "erased_blocks": [...],                                 ║
║          "bds_runs": [...]                                       ║
║        },                                                        ║
║        "post_state": { "merkle_root": "..." },                  ║
║        "slo": { "promised_ms": 10000, "completed_ms": 7200 },  ║
║        "controller": { "fw_ver": "x.y" },                       ║
║        "signature": { "alg": "Ed25519", "value": "..." }        ║
║      }                                                           ║
║  Crypto: Ed25519_Sign(receipt_json) → signature                 ║
║  Journal: Append(RECEIPT_SIGNED)                                ║
╚═════════════════════════════════════════════════════════════════╝
                   │ receipt
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Return to NVMe Command                         │
│                    Store in PCD buffer                           │
└──────────────────┬──────────────────────────────────────────────┘
                   │ SUCCESS
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                         HOST: GET_PCD                            │
│                  Retrieve receipt via admin cmd                  │
└──────────────────┬──────────────────────────────────────────────┘
                   │ receipt JSON
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PYTHON VERIFIER                               │
│                    (verify_pcd.py)                               │
│  - Parse JSON                                                    │
│  - Verify Ed25519 signature                                     │
│  - Verify pre_state Merkle root matches chunk states            │
│  - Simulate actions: apply punctures/erasures to pre → post     │
│  - Verify post_state Merkle root matches computation            │
│  - Check SLO compliance                                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │ 
                   ▼
              VERIFICATION OK ✓
```

---

## Module Dependencies Graph

```
                         ┌──────────────┐
                         │   OpenSSL    │
                         │ (SHA256,HKDF,│
                         │   Ed25519)   │
                         └──────┬───────┘
                                │
                    ┌───────────┴────────────┐
                    │                        │
            ┌───────▼────────┐      ┌───────▼────────┐
            │ vcsd_merkle.h  │      │vcsd_crypto_stub│
            │  (header-only) │      │      .h        │
            └───────┬────────┘      └───────┬────────┘
                    │                       │
        ┌───────────┴───────────┐           │
        │                       │           │
┌───────▼────────┐     ┌────────▼───────┐  │
│ vcsd_chunk_mgr │     │   vcsd_ppk     │  │
│      .h/.cc    │     │    .h/.cc      │  │
│  - Snapshots   │     │  - Derive keys │  │
│  - Bitmaps     │     │  - Puncture    │  │
│  - Merkle roots│     └────────┬───────┘  │
└───────┬────────┘              │          │
        │                       │          │
        │          ┌────────────┴──────┐   │
        │          │                   │   │
        │  ┌───────▼────────┐  ┌───────▼───▼───────┐
        │  │  vcsd_planner  │  │    vcsd_pcd       │
        │  │     .h/.cc     │  │     .h/.cc        │
        │  │  - Build plan  │  │  - Sign receipt   │
        │  └───────┬────────┘  └───────────────────┘
        │          │
        │  ┌───────▼────────┐
        │  │ vcsd_deletion  │
        │  │     .h/.cc     │
        │  │  - ExecPpk     │
        │  │  - ExecErase   │
        └─►│  - ExecBds     │
           └───────┬────────┘
                   │
           ┌───────┴────────┐
           │                │
   ┌───────▼───────┐ ┌─────▼──────┐
   │  vcsd_ecc     │ │ vcsd_bds   │
   │    .h/.cc     │ │   .h/.cc   │
   │ - Measure     │ │ - Run BDS  │
   │ - AcceptBds   │ │            │
   └───────────────┘ └────────────┘

   ┌─────────────────────────────────────┐
   │       Infrastructure Layer          │
   ├─────────────────────────────────────┤
   │  vcsd_journal.h/.cc                 │
   │  vcsd_sdt.h/.cc                     │
   │  vcsd_nvme_admin.cc                 │
   │  vcsd_config.h/.cc                  │
   └─────────────────────────────────────┘
```

---

## Testing Pyramid

```
                              ┌────────────────────┐
                              │   End-to-End Tests │
                              │  - Full workflow   │
                              │  - NVMe → Receipt  │
                              └──────────┬─────────┘
                                         │
                           ┌─────────────┴─────────────┐
                           │   Integration Tests       │
                           │  - PPK flow               │
                           │  - Hybrid flow            │
                           │  - Crash recovery         │
                           └────────────┬──────────────┘
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 │           Unit Tests                        │
                 │  test_merkle     test_ppk                   │
                 │  test_chunk_mgr  test_planner               │
                 │  test_ecc        test_deletion              │
                 └─────────────────────────────────────────────┘
```

---

## Critical Paths

### Path 1: PPK Deletion (No Migration) - **CORE PROOF**
```
Snapshot → PPK-only Plan → ExecPpk → Receipt
   ↓           ↓              ↓          ↓
 Chunks → Punct.leaves → Rotate keys → Verifiable
```
**Success Metric**: Delete invalid pages without migrating valid pages

### Path 2: Hybrid Deletion (Cost Optimized)
```
Snapshot → Hybrid Plan → ExecPpk + ExecErase → Receipt
   ↓           ↓              ↓                    ↓
 Chunks → PPK+Erase → No-mig + Min-mig → Lower cost than pure erase
```
**Success Metric**: Reduce migrations by >50% vs pure erasure

### Path 3: Full Deletion (Performance Optimized)
```
Snapshot → Full Plan → ExecPpk + ExecErase + ExecBds → Receipt
   ↓           ↓              ↓                           ↓
 Chunks → PPK+Era+BDS → Fast sanitize if safe → Meet SLO
```
**Success Metric**: SLO hit rate >95% for D10s

---

## Configuration Examples

### PPK-Only Mode (Phase 1-3)
```yaml
vcsd:
  mode: ppk_only
  ppk:
    tree_height: 6        # 2^6 = 64 pages per chunk
    digest_size: 16       # Compact digest (128-bit)
  planner:
    ppk_only: true
  bds:
    enabled: false
```

### Hybrid Mode (Phase 4)
```yaml
vcsd:
  mode: hybrid
  ppk:
    tree_height: 6
  planner:
    ppk_only: false
    erase_threshold: 0.6  # Erase if >60% invalid
  bds:
    enabled: false
```

### Full Mode (Phase 5)
```yaml
vcsd:
  mode: full
  ppk:
    tree_height: 6
  planner:
    ppk_only: false
    erase_threshold: 0.6
  bds:
    enabled: true
    pre_margin_sym: 2     # Require 2 symbols margin before
    post_margin_sym: 4    # Require 4 symbols margin after
    per_plane_budget: 8   # Max 8 BDS ops per plane
  slo_classes:
    D10s: 10000           # 10 second deadline
    D60s: 60000           # 60 second deadline
```

---

## Key Metrics to Track

### Performance Metrics
- **Deletion Latency**: Time from DELETE_SELECTIVE to receipt
- **SLO Hit Rate**: % requests meeting deadline
- **Throughput**: Deletions per second

### Cost Metrics
- **Migrations**: Valid pages moved during deletion
- **Erasures**: Block erase operations
- **P/E Cycles**: Per-block erase count variance
- **BDS Ops**: Bounded disturb sanitization count

### Verification Metrics
- **PCD Size**: Receipt JSON bytes
- **Verify Time**: Time to validate receipt
- **Signature Overhead**: Ed25519 sign/verify latency

### Reliability Metrics
- **ECC Margins**: Pre/post BDS (tmax - tobs)
- **RBER**: Raw bit error rate trends
- **Scrub Events**: Triggered error corrections

---

## Common Pitfalls & Debugging

### Issue: PPK keys still recoverable after puncture
**Cause**: Ancestor seeds not rotated correctly  
**Debug**: Log seed values before/after puncture, verify HKDF rotation  
**Test**: Try to derive deleted leaf key, should fail

### Issue: Merkle root mismatch in receipt
**Cause**: Bitmap not updated after write/erase  
**Debug**: Dump chunk bitmaps, compare snapshot vs actual  
**Test**: Force write, verify bitmap update before snapshot

### Issue: Planner chooses suboptimal strategy
**Cause**: Cost model doesn't match actual latency  
**Debug**: Log cost estimates vs measured latency, tune weights  
**Test**: Run on diverse invalid distributions, verify cost reduction

### Issue: BDS causes ECC failures
**Cause**: Insufficient pre-BDS margin checking  
**Debug**: Check ECC margins before/after, verify threshold  
**Test**: Inject weak blocks, verify BDS rejection

### Issue: Receipt verification fails
**Cause**: Signature computed over wrong payload  
**Debug**: Log JSON before signing, verify canonical serialization  
**Test**: Manually verify signature with openssl

---

## Performance Targets (Prototype)

| Metric | Target | Measurement |
|--------|--------|-------------|
| PPK Derive | <1μs/key | Microbenchmark |
| PPK Puncture | <10μs/leaf | Microbenchmark |
| Snapshot | <1ms | Phase 1 test |
| PPK-only Plan | <5ms | Phase 3 test |
| Hybrid Plan | <10ms | Phase 4 test |
| Receipt Sign | <1ms | Phase 2 test |
| Receipt Verify | <1ms | Python verifier |
| Delete Latency (PPK) | <10ms | End-to-end |
| Delete Latency (Hybrid) | <100ms | End-to-end |
| Migration Reduction | >50% | vs baseline |
| SLO Hit (D10s) | >95% | Workload trace |

---

## References

1. **ErasuCrypto** (FAST 2017): Warm-start biclique heuristic for hybrid crypto+erase
2. **Selective Deletion** (ICCAD 2018): Security-aware chunk placement, key page metadata
3. **Survey** (2020): ECC-gated sanitization concerns, program disturb risks
4. **Puncturable PRF**: Tree-based key derivation with selective revocation
5. **Merkle Trees**: Authenticated data structures for state commitment

---

**Last Updated**: 2026-03-08
