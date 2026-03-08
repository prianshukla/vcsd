# V-CSD Implementation Plan
**Target**: SimpleSSD Standalone Simulator  
**Timeline**: Week 7 - Week 14 (March 8 - April 30, 2026) - **8 Weeks**  
**Goal**: Prove Verifiable Cryptographic Secure Deletion algorithm works  
**Focus**: PCD (Proof-Carrying Deletion) + PPK (Per-Page Key) - Controller-Agnostic Design

**Note**: V-CSD is designed to integrate with SimpleSSD's HIL (Host Interface Layer), not specific to NVMe. The architecture supports any controller implementing standard storage commands (WRITE/TRIM/DELETE).

---

## Architecture Overview

V-CSD implements verifiable secure deletion for SSDs through:
- **PPK (Per-Page Key)**: Puncturable key trees for cryptographic deletion without data migration
- **PCD (Proof-Carrying Deletion)**: Cryptographic receipts proving deletion occurred
- **Hybrid Planner**: Optimizes between PPK puncture, block erasure, and optional BDS
- **Merkle Commitments**: Pre/post state verification for tamper-proof receipts

---

## Module Organization

```
SimpleSSD-Standalone/
├── vcsd/                          # V-CSD implementation root
│   ├── core/                      # Core cryptographic components (PHASE 1-2)
│   │   ├── vcsd_ppk.h/.cc        # PPK: HKDF tree, derivation, puncture
│   │   ├── vcsd_pcd.h/.cc        # PCD: receipt generation & signing
│   │   ├── vcsd_merkle.h         # Merkle tree utilities (header-only)
│   │   └── vcsd_crypto_stub.h    # Ed25519/SHA256 stubs for simulator
│   │
│   ├── manager/                   # Data & state management (PHASE 1)
│   │   ├── vcsd_chunk_manager.h/.cc   # Chunk↔block mapping, bitmaps
│   │   └── vcsd_config.h/.cc          # Configuration (simplified, no TSD)
│   │
│   ├── planner/                   # Deletion planning (PHASE 3-4)
│   │   ├── vcsd_planner.h/.cc    # Hybrid optimization planner
│   │   └── vcsd_deletion.h/.cc   # Deletion execution engine
│   │
│   ├── monitor/                   # Health & reliability (PHASE 4-5)
│   │   ├── vcsd_ecc.h/.cc        # ECC monitoring
│   │   └── vcsd_bds.h/.cc        # Bounded Disturb Sanitization (optional)
│   │
│   ├── interface/                 # Transport & journaling (PHASE 5)
│   │   ├── vcsd_hil_hook.h/.cc   # SimpleSSD HIL integration (controller-agnostic)
│   │   ├── vcsd_sdt.h/.cc        # Secure Deletion Transport abstraction
│   │   └── vcsd_journal.h/.cc    # Crash consistency journal
│   │
│   └── controller/                # Controller-specific adapters (PHASE 5)
│       ├── vcsd_nvme_admin.cc    # NVMe vendor admin commands (optional)
│       ├── vcsd_sata_ext.cc      # SATA extended commands (optional)
│       └── vcsd_ufs_cmd.cc       # UFS vendor commands (optional)
│
├── tools/                         # Verification & testing
│   ├── verify_pcd.py             # Python PCD verifier
│   ├── run_harness.py            # Benchmark harness
│   └── vcsd_verify.cc            # C++ verifier tool
│
├── configs/                       # Configuration files
│   ├── vcsd_ppk_only.yml         # PPK-only mode (Phase 1)
│   ├── vcsd_ppk_erase.yml        # PPK + erasure (Phase 4)
│   └── vcsd_full.yml             # Full with BDS (Phase 5)
│
└── tests/                         # Unit & integration tests
    ├── test_ppk.cc               # PPK derivation & puncture tests
    ├── test_merkle.cc            # Merkle tree tests
    ├── test_planner.cc           # Planner heuristic tests
    └── integration/              # End-to-end tests
```

---

## Implementation Phases (Git Workflow)

### **PHASE 0: Pre-Implementation Verification & Analysis**
**Branch**: `feature/phase0-verification`  
**Duration**: Week 7 Day 1-2 (March 8-9, 2026)  
**Goal**: Understand SimpleSSD architecture and prepare integration points

#### Submodules:

1. **0.1: SimpleSSD Build Verification** (`feature/phase0.1-build`)
   - Build SimpleSSD-Standalone on vcsd branch
   - Document build time, warnings, dependency versions
   - Verify all SimpleSSD tests pass: `make -j$(nproc) && ctest`
   - **Test**: SimpleSSD baseline functionality intact
   - **Commit**: "[PHASE0.1] Verify SimpleSSD-Standalone builds on vcsd branch"

2. **0.2: Dependency Audit** (`feature/phase0.2-deps`)
   - Check OpenSSL version: `openssl version` (need ≥3.0)
   - Verify yaml-cpp available: `find /usr -name "yaml-cpp*"`
   - Check nlohmann/json: `ldconfig -p | grep json` or header files
   - Install GoogleTest if missing
   - Document installation paths in `docs/DEPENDENCIES.md`
   - **Test**: All dependencies found or installed
   - **Commit**: "[PHASE0.2] Document and verify V-CSD dependencies"

3. **0.3: SimpleSSD Architecture Analysis** (`feature/phase0.3-analysis`)
   - **Study HIL (Host Interface Layer)**:
     - Locate HIL base class: `simplessd/hil/hil.cc` and `hil.hh`
     - Identify I/O command entry points (READ/WRITE/TRIM)
     - Document command flow: HIL → ICL (cache) → FTL → PAL
   - **Study FTL Interface**:
     - Examine `simplessd/ftl/ftl.cc` for page mapping
     - Identify where to intercept page writes for PPK encryption
     - Document block allocation and garbage collection hooks
   - **Study Configuration System**:
     - Examine `simplessd/sim/config_reader.cc`
     - Plan how to add V-CSD config sections
   - **Deliverable**: Create `docs/SIMPLESSD_INTEGRATION.md` documenting:
     - V-CSD hook points (where to intercept I/O commands)
     - Modified files needed for integration
     - Mock strategy for Phase 1-4 (V-CSD standalone testing)
   - **Test**: Documentation reviewed
   - **Commit**: "[PHASE0.3] Analyze SimpleSSD architecture for V-CSD integration"

4. **0.4: Baseline Performance Capture** (`feature/phase0.4-baseline`)
   - Run SimpleSSD with standard workload (sequential writes, random I/O)
   - Capture metrics:
     - I/O latency (avg, p50, p99)
     - Throughput (MB/s)
     - Memory usage
     - GC frequency
   - Save to `tools/baseline_metrics.txt`
   - **Purpose**: Compare against V-CSD overhead in Phase 6
   - **Test**: Metrics captured
   - **Commit**: "[PHASE0.4] Capture SimpleSSD baseline performance metrics"

5. **0.5: Integration Planning** (`feature/phase0.5-integration`)
   - **Create CMake Integration Plan**:
     - Identify where to add `add_subdirectory(vcsd)` in root CMakeLists.txt
     - Plan library linking: SimpleSSD-Standalone → libvcsd.a
   - **Create Mock Strategy**:
     - Phase 1-4: V-CSD modules test standalone (no SimpleSSD calls)
     - Phase 5: Real SimpleSSD integration with HIL hooks
   - **Document Controller-Agnostic Design**:
     - V-CSD works with any controller (NVMe/SATA/UFS)
     - Interface through HIL layer, not controller-specific
   - **Deliverable**: Update `docs/SIMPLESSD_INTEGRATION.md` with:
     - CMakeLists.txt modification plan
     - Mock vs real integration timeline
     - Controller-agnostic interface design
   - **Test**: Plan reviewed
   - **Commit**: "[PHASE0.5] Complete V-CSD integration planning"

6. **0.6: Memory Safety Setup** (`feature/phase0.6-sanitizers`)
   - Add CMake options for sanitizers:
     ```cmake
     option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
     option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
     ```
   - Add compiler flags in vcsd/CMakeLists.txt
   - Create `scripts/test_with_sanitizers.sh` helper
   - **Test**: Build with `-DENABLE_ASAN=ON`, run sample test
   - **Commit**: "[PHASE0.6] Add memory safety testing infrastructure"

7. **0.7: Backup & Recovery Setup** (`feature/phase0.7-backup`)
   - Create `/scratch/me/work/vcsd-backups/` directory
   - Write `scripts/backup_vcsd.sh` (conservative: only after phase merges)
   - Add backup reminder to `docs/DAILY_CHECKLIST.md`
   - **Test**: Run manual backup, verify tarball created
   - **Commit**: "[PHASE0.7] Setup local backup strategy"

**Merge to**: `vcsd`  
**Tag**: `v0.0-pre-implementation`  
**Deliverable**: SimpleSSD understanding documented, dependencies verified, integration plan ready

---

### **PHASE 1: Core Infrastructure & PPK Foundation**
**Branch**: `feature/phase1-ppk-foundation`  
**Duration**: Week 7 Day 3 - Week 8 (March 10-14, 2026)  
**Goal**: Implement PPK derivation and puncture - prove no-migration deletion works

#### Submodules:
1. **1.1: Project Setup** (`feature/phase1.1-setup`)
   - CMakeLists.txt with vcsd library target
   - Directory structure creation
   - Dependencies: OpenSSL (SHA256, HKDF), yaml-cpp
   - Basic header stubs
   - **Doxygen Setup**:
     - Create `Doxyfile` for API documentation
     - Document header format (brief, params, returns)
     - Add `make docs` target to CMakeLists.txt
   - **Test**: Build succeeds, `make docs` generates HTML
   - **Commit**: "Initial V-CSD project structure, build system, and Doxygen"

2. **1.2: Merkle Tree Utilities** (`feature/phase1.2-merkle`)
   - Implement `vcsd_merkle.h` (header-only)
   - SHA256-based Merkle root computation
   - **Test**: Unit test with known vectors
   - **Commit**: "Add Merkle tree utilities for state commitment"

3. **1.3: Chunk Manager - Data Structures** (`feature/phase1.3-chunk-ds`)
   - `vcsd_chunk_manager.h`: Define structs (PageStateBitmap, ChunkMapEntry, Snapshot)
   - Basic initialization, no functionality yet
   - **Test**: Instantiate and verify structure sizes
   - **Commit**: "Define chunk manager data structures"

4. **1.4: Chunk Manager - Core Logic** (`feature/phase1.4-chunk-logic`)
   - Implement `SnapshotOf()`, `UpdateOnWrite()`, `Meta()`
   - Maintain chunk↔block mapping
   - Track page state bitmaps (free/valid/invalid)
   - **Test**: Synthetic write patterns, verify bitmap updates
   - **Commit**: "Implement chunk manager core logic"

5. **1.5: PPK Manager - HKDF Tree** (`feature/phase1.5-ppk-hkdf`)
   - Implement `vcsd_ppk.cc`: HKDF-based key derivation tree
   - `DeriveLeafKey()`: Generate per-page keys from master seed
   - Binary tree structure (height = log2(pages_per_chunk))
   - **Test**: Derive keys, verify determinism and uniqueness
   - **Commit**: "Implement PPK HKDF key derivation tree"

6. **1.6: PPK Manager - Puncture Logic** (`feature/phase1.6-ppk-puncture`)
   - Implement `Puncture()`: Mark leaves as deleted, rotate ancestor seeds
   - `CommitDigests()`: Compact representation for key page storage
   - **Test**: Puncture patterns, verify keys become unrecoverable
   - **Commit**: "Implement PPK puncture and digest commitment"

7. **1.7: Config Manager** (`feature/phase1.7-config`)
   - Implement `vcsd_config.h/.cc`: YAML config loader (simplified, no TSD)
   - Parameters: PPK tree height, chunk size, quotas
   - **Test**: Load sample config, verify parsing
   - **Commit**: "Add V-CSD configuration manager"

**Merge to**: `vcsd`  
**Tag**: `v0.1-ppk-core`  
**Deliverable**: PPK can derive and puncture keys; chunk manager tracks state

---

### **PHASE 2: PCD Receipt Generation**
**Branch**: `feature/phase2-pcd-receipts`  
**Goal**: Generate and verify PCD receipts - prove verifiability works

#### Submodules:
1. **2.1: Crypto Stubs** (`feature/phase2.1-crypto-stubs`)
   - `vcsd_crypto_stub.h`: Ed25519 sign/verify stubs for simulator
   - Use OpenSSL EVP interface (real signatures)
   - **Test**: Sign and verify test payloads
   - **Commit**: "Add Ed25519 crypto stub for PCD signing"

2. **2.2: PCD Receipt Structure** (`feature/phase2.2-pcd-struct`)
   - `vcsd_pcd.h`: Define Receipt struct, JSON schema
   - Fields: req_id, scope, pre_state (Merkle), actions, post_state, signature
   - **Test**: Serialize to/from JSON
   - **Commit**: "Define PCD receipt structure and JSON schema"

3. **2.3: PCD Generation** (`feature/phase2.3-pcd-gen`)
   - Implement `Pcd::Sign()`: Build receipt from snapshot + plan + health
   - Compute pre_state Merkle root from chunk bitmaps
   - Embed PPK puncture list, erased blocks, SLO timing
   - **Test**: Generate receipt, verify JSON structure
   - **Commit**: "Implement PCD receipt generation"

4. **2.4: PCD Signing** (`feature/phase2.4-pcd-sign`)
   - Sign receipt payload with Ed25519
   - Store signature in receipt
   - **Test**: Generate signed receipt, verify signature
   - **Commit**: "Add Ed25519 signing to PCD receipts"

5. **2.5: Python Verifier** (`feature/phase2.5-verifier`)
   - `tools/verify_pcd.py`: Parse receipt, verify Merkle root
   - Verify Ed25519 signature against public key
   - Validate pre→post state transition
   - **Test**: Verify valid and invalid receipts
   - **Commit**: "Add Python PCD receipt verifier"

**Merge to**: `vcsd`  
**Tag**: `v0.2-pcd-generation`  
**Deliverable**: PCD receipts can be generated and cryptographically verified

---

### **PHASE 3: Planner - PPK-Only Mode**
**Branch**: `feature/phase3-planner-ppk`  
**Goal**: Implement planner with PPK-only strategy - prove optimization works

#### Submodules:
1. **3.1: Planner Data Structures** (`feature/phase3.1-planner-ds`)
   - `vcsd_planner.h`: Define Plan, Health, Quotas structs
   - **Test**: Instantiate structures
   - **Commit**: "Define planner data structures"

2. **3.2: PPK-Only Planner** (`feature/phase3.2-ppk-planner`)
   - Implement `Planner::Build()`: PPK-only mode (no erasures)
   - Select invalid pages for PPK puncture
   - Compute cost model (latency estimate)
   - **Test**: Plan generation for various invalid distributions
   - **Commit**: "Implement PPK-only deletion planner"

3. **3.3: Deletion Engine - PPK Execution** (`feature/phase3.3-deletion-ppk`)
   - `vcsd_deletion.cc`: Implement `ExecPpk()`
   - Call PPK puncture for planned leaves
   - Update chunk metadata (Merkle roots)
   - **Test**: Execute plan, verify keys are punctured
   - **Commit**: "Implement PPK deletion execution"

4. **3.4: End-to-End PPK Flow** (`feature/phase3.4-e2e-ppk`)
   - Wire together: Snapshot → Plan → Execute → Receipt
   - **Test**: Full deletion request through PPK-only path
   - **Commit**: "Wire end-to-end PPK deletion flow"

**Merge to**: `vcsd`  
**Tag**: `v0.3-ppk-planner`  
**Deliverable**: Complete PPK-only deletion with verifiable receipts (no migrations!)

---

### **PHASE 4: Hybrid Planner - Add Erasure**
**Branch**: `feature/phase4-hybrid-planner`  
**Goal**: Add block erasure with minimal migrations - prove hybrid approach works

#### Submodules:
1. **4.1: Erasure Cost Model** (`feature/phase4.1-erase-cost`)
   - Extend planner with erasure cost computation
   - Cost = migrations + erase_latency
   - **Test**: Cost estimation for various block states
   - **Commit**: "Add erasure cost model to planner"

2. **4.2: Warm-Start Biclique Heuristic** (`feature/phase4.2-warmstart`)
   - Implement ErasuCrypto-style warm-start
   - Biclique covering: rows (PPK) vs columns (erasure)
   - **Test**: Warm-start on synthetic invalid patterns
   - **Commit**: "Add warm-start biclique heuristic to planner"

3. **4.3: Hybrid Refinement** (`feature/phase4.3-hybrid-refine`)
   - Refine warm-start: high valid density → PPK; high invalid density → erase
   - Threshold-based decision
   - **Test**: Compare PPK-only vs hybrid plans
   - **Commit**: "Implement hybrid PPK+erasure refinement"

4. **4.4: Deletion Engine - Erasure Execution** (`feature/phase4.4-deletion-erase`)
   - Implement `ExecErase()`: Migrate valid pages, erase block
   - Update FTL and chunk metadata
   - **Test**: Execute erasure, verify migrations
   - **Commit**: "Implement erasure deletion execution"

5. **4.5: ECC Monitor - Basic** (`feature/phase4.5-ecc-basic`)
   - `vcsd_ecc.cc`: Basic ECC margins tracking
   - `Measure()`: Return dummy margins for now
   - **Test**: Query margins for blocks
   - **Commit**: "Add basic ECC monitor"

**Merge to**: `vcsd`  
**Tag**: `v0.4-hybrid-planner`  
**Deliverable**: Hybrid planner optimizes between PPK and erasure

---

### **PHASE 5: Full System Integration**
**Branch**: `feature/phase5-integration`  
**Goal**: Integrate V-CSD with SimpleSSD HIL layer (controller-agnostic) + optional controller extensions

#### Submodules:
1. **5.1: HIL Hook Integration** (`feature/phase5.1-hil`)
   - `vcsd_hil_hook.cc`: Integrate V-CSD with SimpleSSD HIL base class
   - Hook into I/O command path: intercept WRITE/TRIM/DELETE commands
   - **Controller-Agnostic**: Works with any SimpleSSD controller (NVMe/SATA/UFS)
   - Modify SimpleSSD HIL to call V-CSD hooks
   - **Test**: V-CSD intercepts commands from different controller types
   - **Commit**: "[PHASE5.1] Integrate V-CSD with SimpleSSD HIL (controller-agnostic)"

2. **5.2: BDS Engine** (`feature/phase5.2-bds`)
   - `vcsd_bds.cc`: Bounded Disturb Sanitization simulation
   - ECC-gated: only if margins permit
   - **Test**: BDS execution with margin checks
   - **Commit**: "Add BDS engine with ECC gating"

3. **5.3: Crash Journal** (`feature/phase5.3-journal`)
   - `vcsd_journal.cc`: Append-only log with markers
   - `Replay()`: Recover from crashes
   - **Test**: Crash/replay scenarios
   - **Commit**: "Add crash-consistent journal"

4. **5.4: SDT Interface** (`feature/phase5.4-sdt`)
   - `vcsd_sdt.h/.cc`: Secure Deletion Transport abstraction
   - Generic command interface (not controller-specific)
   - **Test**: Send commands via SDT to different controllers
   - **Commit**: "Add Secure Deletion Transport interface"

5. **5.5: Controller Extensions (Optional)** (`feature/phase5.5-controllers`)
   - `vcsd_nvme_admin.cc`: NVMe vendor commands (DELETE_SELECTIVE, GET_PCD, GET_HEALTH)
   - `vcsd_sata_ext.cc`: SATA extended commands (if needed)
   - `vcsd_ufs_cmd.cc`: UFS vendor commands (if needed)
   - **Note**: Core V-CSD works without these; extensions for protocol compliance
   - **Test**: Issue controller-specific admin commands
   - **Commit**: "Add optional controller-specific admin extensions"

6. **5.6: Integration Testing** (`feature/phase5.6-integration`)
   - Full end-to-end tests: Host → HIL → V-CSD → Receipt → Verify
   - Test with multiple controller types (if available)
   - **Test**: Complete deletion workflows across controllers
   - **Commit**: "Add full integration tests"

**Merge to**: `vcsd`  
**Tag**: `v0.5-full-system`  
**Deliverable**: Complete V-CSD system integrated with SimpleSSD, controller-agnostic design

---

### **PHASE 6: Benchmarking & Evaluation**
**Branch**: `feature/phase6-evaluation`  
**Goal**: Benchmark, prove performance characteristics, and provide verification tools

#### Submodules:
1. **6.1: Baseline Implementations** (`feature/phase6.1-baselines`)
   - Pure erasure baseline
   - Pure crypto baseline (shared keys)
   - **Test**: Run baselines, collect stats
   - **Commit**: "Add baseline deletion schemes for comparison"

2. **6.2: Test Harness** (`feature/phase6.2-harness`)
   - `tools/run_harness.py`: Automated benchmark runner
   - Workload traces, multiple configs
   - **Test**: Run harness on all schemes
   - **Commit**: "Add automated benchmark harness"

3. **6.3: Metrics & Plotting** (`feature/phase6.3-metrics`)
   - KPIs: erasures, migrations, latency, P/E variance, SLO hit rate
   - **Performance Regression Tests**:
     - Compare against Phase 0 baseline metrics
     - Alert if V-CSD overhead exceeds threshold (e.g., >10% latency increase)
     - Automated checks in harness
   - Plotting scripts
   - **Test**: Generate plots from benchmark data, verify regression checks
   - **Commit**: "Add metrics collection, plotting, and performance regression tests"

4. **6.4: Python Verification Tools** (`feature/phase6.4-verification`)
   - `tools/verify_pcd.py`: Standalone PCD receipt verifier
     - Parse JSON receipts
     - Verify Ed25519 signatures
     - Verify Merkle proofs
   - `tools/vcsd_cli.py`: Command-line interface for V-CSD operations
     - Generate deletion requests
     - Query deletion status
     - Validate receipts
   - Dependencies: `pip install cryptography pyyaml`
   - **Test**: Verify real PCD receipts from Phase 2-5 tests
   - **Commit**: "Add Python PCD verification tools and CLI"

**Merge to**: `vcsd`  
**Tag**: `v1.0-complete`  
**Deliverable**: Reproducible evaluation proving V-CSD effectiveness + standalone verification tools

---

## Git Branching Strategy

```
main (stable releases only)
  │
  ├── v0.1-ppk-core
  ├── v0.2-pcd-generation
  ├── v0.3-ppk-planner
  ├── v0.4-hybrid-planner
  ├── v0.5-full-system
  └── v1.0-complete
  
vcsd (V-CSD main branch - local only)
  │
  ├── feature/phase1-ppk-foundation
  │   ├── feature/phase1.1-setup
  │   ├── feature/phase1.2-merkle
  │   ├── feature/phase1.3-chunk-ds
  │   ├── feature/phase1.4-chunk-logic
  │   ├── feature/phase1.5-ppk-hkdf
  │   ├── feature/phase1.6-ppk-puncture
  │   └── feature/phase1.7-config
  │
  ├── feature/phase2-pcd-receipts
  │   ├── ...
  │
  └── ... (similar for other phases)
```

### Workflow (Local-Only):
1. Create phase branch from `vcsd`
2. Create submodule branch from phase branch
3. Implement → Test → Commit with descriptive message
4. Merge submodule → phase branch (after testing)
5. Merge phase → `vcsd` (after all submodules complete)
6. Tag milestone on `vcsd` (local only, no push)

### Commit Message Format:
```
[PHASE] Brief description

Detailed explanation:
- What: Specific changes made
- Why: Rationale for implementation choice
- Testing: How it was verified

Related: #issue-number (if applicable)
```

Example:
```
[PHASE1.5] Implement PPK HKDF key derivation tree

Detailed explanation:
- What: Added HKDF-based binary tree for per-page key derivation
- Why: Enables puncturable encryption without key storage overhead
- Testing: Unit tests with 1000 keys, verified uniqueness and determinism

Tree height configurable via config (default: 6 levels = 64 pages/chunk)
```

---

## Testing Strategy

### Unit Tests (GoogleTest)
**Purpose**: Test individual module functionality in isolation  
**When**: Required for every submodule commit

- **test_merkle.cc**: Merkle root computation, known vectors
- **test_ppk.cc**: Key derivation, puncture, recovery failure
- **test_chunk_manager.cc**: Bitmap updates, snapshot consistency
- **test_planner.cc**: Cost model, plan optimality
- **test_ecc.cc**: Margin thresholds, BDS gating

**Coverage Target**: ≥80% line coverage per module

### Integration Tests
**Purpose**: Test cross-module interactions  
**When**: After merging 2+ related submodules to phase branch

#### Phase 1-2 Integration:
- **test_ppk_chunk_integration.cc**: 
  - PPK derives keys → ChunkManager tracks pages → Keys punctured → Verify bitmap updates
  - Test: Write 1000 pages, delete 500, verify 500 keys unrecoverable

#### Phase 2-3 Integration:
- **test_pcd_ppk_integration.cc**:
  - Complete flow: Snapshot → Plan → Execute → Receipt → Verify
  - Test: Multiple deletion scopes, verify all PCD fields valid

#### Phase 3-4 Integration:
- **test_planner_hybrid_integration.cc**:
  - Planner selects PPK vs erasure → Execution engine uses correct method
  - Test: Mixed workload, verify cost-optimal decisions

#### Phase 5 Integration:
- **test_hil_vcsd_integration.cc**:
  - SimpleSSD HIL → V-CSD hooks → Deletion → SimpleSSD FTL updated
  - Test: I/O commands from different controllers (NVMe/SATA/UFS if available)

**Integration Test Timing**:
- Run after every phase branch merge
- Must pass before merging phase → vcsd
- Part of phase completion checklist

### Memory Safety Tests
**Purpose**: Detect memory leaks, buffer overflows, undefined behavior  
**Tools**: AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), Valgrind

#### ASan/UBSan Builds:
```bash
cmake -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON -DENABLE_UBSAN=ON ..
make -j$(nproc)
ctest -V  # Run all tests with sanitizers
```

#### Valgrind Checks:
```bash
valgrind --leak-check=full --show-leak-kinds=all ./test_ppk
```

**When to Run**:
- Required before merging any submodule with pointer manipulation
- Mandatory for all Phase 1-2 core modules (PPK, ChunkManager, Merkle)
- Weekly full suite run with sanitizers

### End-to-End Tests
**Purpose**: Test complete system workflows  
**When**: Phase 5-6 only (after full integration)

- **test_e2e_complete.cc**: Host → HIL → V-CSD → Receipt → Python Verify
- **test_workload_replay.cc**: Trace replay with mixed read/write/delete
- **test_controller_variants.cc**: Test with multiple controller types

### Performance Regression Tests
**Purpose**: Ensure V-CSD overhead stays within acceptable bounds  
**Baseline**: Phase 0.4 captured SimpleSSD metrics

#### Regression Criteria:
- V-CSD latency overhead ≤ 10% vs baseline SimpleSSD
- V-CSD throughput reduction ≤ 5%
- Memory overhead ≤ 50MB for 100GB SSD simulation
- GC frequency increase ≤ 15%

#### Automated Checks:
- Run in Phase 6.3 harness
- Compare against `tools/baseline_metrics.txt`
- FAIL build if thresholds exceeded without justification

### Verification
- **verify_pcd.py**: Automated receipt verification (Phase 6.4)
- **verify_suite.sh**: Batch verification of all test receipts
- **valgrind_check.sh**: Automated valgrind runs on critical modules

---

## Rollback Points

| Tag | Branch | Functionality | Use Case |
|-----|--------|---------------|----------|
| `v0.0-pre-implementation` | `vcsd` | SimpleSSD analysis complete, dependencies verified | Baseline before V-CSD implementation |
| `v0.1-ppk-core` | `vcsd` | PPK-only, no planner | Prove no-migration crypto deletion |
| `v0.2-pcd-generation` | `vcsd` | + PCD receipts | Prove verifiability |
| `v0.3-ppk-planner` | `vcsd` | + PPK planner | Prove basic optimization |
| `v0.4-hybrid-planner` | `vcsd` | + Erasure hybrid | Prove cost reduction |
| `v0.5-full-system` | `vcsd` | + BDS + HIL integration | Full prototype |
| `v1.0-complete` | `vcsd` | + Evaluation + Verification tools | Paper-ready |

**Recovery**: `git checkout <tag>` to rollback to stable milestone

---

## Success Criteria

### Phase 0 Success:
- [ ] SimpleSSD-Standalone builds successfully on vcsd branch
- [ ] All SimpleSSD tests pass (baseline regression check)
- [ ] OpenSSL ≥3.0, yaml-cpp, nlohmann/json, GoogleTest installed
- [ ] SimpleSSD architecture documented in `docs/SIMPLESSD_INTEGRATION.md`
- [ ] V-CSD integration points identified (HIL hooks, FTL interactions)
- [ ] Baseline performance metrics captured in `tools/baseline_metrics.txt`
- [ ] CMake integration plan documented
- [ ] Mock testing strategy defined for Phase 1-4
- [ ] Sanitizer builds configured and tested
- [ ] Backup script created and tested

### Phase 1 Success:
- [ ] PPK can derive unique keys per page
- [ ] Puncture makes keys unrecoverable
- [ ] Chunk manager tracks page states correctly
- [ ] Unit tests pass
- [ ] Doxygen generates API documentation
- [ ] ASan/UBSan tests pass with no errors

### Phase 2 Success:
- [ ] PCD receipts generated with valid JSON
- [ ] Merkle roots match chunk states
- [ ] Ed25519 signatures verify
- [ ] Python verifier validates receipts

### Phase 3 Success:
- [ ] PPK-only planner generates valid plans
- [ ] Deletion executes without data migration
- [ ] End-to-end flow produces verifiable receipt
- [ ] Latency < 10ms for small scopes

### Phase 4 Success:
- [ ] Hybrid planner reduces cost vs PPK-only
- [ ] Erasure executes with minimal migrations
- [ ] Cost model predicts actual latency within 20%

### Phase 6 Success:
- [ ] V-CSD beats pure erasure on migrations (>50% reduction)
- [ ] V-CSD matches pure crypto on security
- [ ] SLO hit rate >95% for D10s class
- [ ] PCD verify time <1ms

---

## Dependencies

### Required Libraries:
- **OpenSSL** (>= 3.0): SHA256, HKDF, Ed25519
- **yaml-cpp**: Config file parsing
- **nlohmann/json**: PCD receipt serialization
- **GoogleTest**: Unit testing
- **Python 3.10+**: cryptography, numpy, pandas, matplotlib (for verifier/harness)

### SimpleSSD Integration:
- Link against SimpleSSD FTL/NAND layers
- Hook into NVMe admin command path
- Access PAL for block erase/program
- Read ECC statistics from PAL

---

## Next Steps

1. **Review this plan** - Confirm phase priorities and scope
2. **Setup development environment** - Install dependencies, verify build
3. **Create git branches** - Initialize branching structure
4. **Begin Phase 1.1** - Project setup and build system
5. **Implement → Test → Commit** - Follow submodule sequence
6. **Track progress** - Update checkboxes in this document

**First Implementation**: Start with Phase 1.1 - Project Setup after approval.

---

## Notes

- **No multi-tenant**: Simplified vs V-S3D (no TSD token validation, single namespace)
- **Simulator focus**: Real hardware integration deferred
- **Incremental verification**: Test after each submodule, not just at phase end
- **Prototype first**: Optimize later (Phase 6), focus on correctness in early phases
- **Document as you go**: Update README and inline comments during implementation

---

**Author**: AI Assistant (based on V-S3D documentation)  
**Date**: 2026-03-08  
**Version**: 1.0
