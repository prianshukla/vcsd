# VCSD Comprehensive Evaluation Plan
**Date**: April 26, 2026  
**Status**: Planning Phase  
**Goal**: Rigorous, versatile, and trustable evaluation of VCSD using real-world MSR-Cambridge traces

---

## 1. Overview

This evaluation will demonstrate VCSD's performance using real-world I/O traces from MSR-Cambridge, comparing against three baseline secure deletion schemes:
- **Erasure-based**: Physical block erasure
- **Crypto-based**: Per-page encryption with key deletion
- **ErasuCrypto**: Hybrid erasure + crypto (Liu et al., 2017)
- **VCSD**: Our PPK + PCD + Hybrid Planner approach

---

## 2. Trace Processing Pipeline

### 2.1 Input: MSR-Cambridge Traces
**Location**: `trace_generator/traces/MSR-Cambridge/`

**Original Format**: 
```
<timestamp> <hostname> <disk> <operation> <lba> <size> <response_time>
```

**Characteristics**:
- Real enterprise storage workloads
- No TRIM commands (pre-TRIM era traces)
- No file-level metadata
- Pure block-level I/O patterns

### 2.2 Augmentation Requirements

Transform MSR traces to VCSD-compatible format:

**Target Format**:
```
<timestamp_ns> <op> <lba> <size_sectors> <file_id> <secure_delete_flag>
```

**Augmentation Steps**:

1. **File-ID Assignment**
   - Cluster LBAs into logical files
   - Use realistic file size distributions
   - Maintain temporal locality (sequential writes → same file)

2. **TRIM Command Injection**
   - Two variants per trace:
     - **End-deletion**: TRIMs at end of trace (batch file deletion)
     - **Mid-deletion**: TRIMs scattered throughout (incremental deletion)
   - TRIM granularity: aligned with file boundaries

3. **Secure Delete Flag**
   - Mark a percentage of files for secure deletion
   - Vary ratios: 25%, 50%, 75%, 100% secure deletion

---

## 3. File Size Distribution Strategies

### 3.1 Distribution Types

| Distribution | Parameters | Use Case |
|:-------------|:-----------|:---------|
| **Log-Normal** | μ=10, σ=2 (median ~148KB) | Web servers, general file systems |
| **Power-Law** | α=1.5, min=4KB, max=1GB | Large-scale storage (Zipf-like) |
| **Bimodal** | Small: N(16KB, 4KB²)<br>Large: N(10MB, 1MB²)<br>Ratio: 80:20 | Mixed workload (metadata + data) |
| **Uniform** | 4KB - 10MB | Baseline, controlled experiments |

### 3.2 File Size Distribution Implementation

```python
import numpy as np

def generate_file_sizes(distribution_type, num_files):
    if distribution_type == "lognormal":
        sizes = np.random.lognormal(mean=10, sigma=2, size=num_files)
    elif distribution_type == "powerlaw":
        sizes = (np.random.pareto(a=1.5, size=num_files) + 1) * 4096
    elif distribution_type == "bimodal":
        small = np.random.normal(16384, 4096, int(num_files * 0.8))
        large = np.random.normal(10*1024*1024, 1024*1024, int(num_files * 0.2))
        sizes = np.concatenate([small, large])
    elif distribution_type == "uniform":
        sizes = np.random.uniform(4096, 10*1024*1024, num_files)
    
    return np.clip(sizes, 4096, 1024*1024*1024).astype(int)
```

---

## 4. TRIM Injection Strategies

### 4.1 End-Deletion Pattern
```
[Original I/O operations - unchanged]
...
[End of trace]
<TRIM commands for selected files>
```

**Characteristics**:
- Simulates batch file deletion (e.g., log rotation, cache cleanup)
- Tests deletion throughput under idle system
- Best-case latency scenario

### 4.2 Mid-Deletion Pattern
```
[10% of I/O operations]
<TRIM for file_1 to file_k>
[20% of I/O operations]
<TRIM for file_(k+1) to file_(k+m)>
...
```

**Characteristics**:
- Simulates incremental deletion during normal operation
- Tests deletion latency under active I/O load
- Realistic production scenario
- Interleaving ratio: TRIM every 10-15% of workload progress

---

## 5. Evaluation Matrix

### 5.1 Trace Variants

Total configurations: **4 distributions × 2 TRIM patterns × 3 secure ratios = 24 variants**

| Trace Variant | File Distribution | TRIM Pattern | Secure % |
|:--------------|:------------------|:-------------|:---------|
| lognormal_end_25 | Log-Normal | End | 25% |
| lognormal_end_50 | Log-Normal | End | 50% |
| lognormal_end_75 | Log-Normal | End | 75% |
| lognormal_mid_25 | Log-Normal | Mid | 25% |
| lognormal_mid_50 | Log-Normal | Mid | 50% |
| lognormal_mid_75 | Log-Normal | Mid | 75% |
| powerlaw_end_25 | Power-Law | End | 25% |
| ... | ... | ... | ... |

### 5.2 Evaluation Per Variant

For each trace variant, run **4 schemes** → 24 × 4 = **96 simulation runs**

---

## 6. Performance Metrics

### 6.1 Primary Metrics

| Metric | Description | Unit |
|:-------|:------------|:-----|
| **Deletion Latency** | Time to complete TRIM operations | µs (avg, p50, p95, p99, max) |
| **I/O Latency** | Read/Write operation latency | µs (avg, p50, p95, p99) |
| **Throughput** | Operations per second | IOPS |
| **Data Migration** | Volume of data copied | GiB |
| **Block Erasures** | Number of physical block erases | count |

### 6.2 VCSD-Specific Metrics

| Metric | Description |
|:-------|:------------|
| **PPK Puncture Time** | Time spent puncturing keys |
| **PCD Generation Time** | Receipt creation latency |
| **Hybrid Decision Overhead** | Planner computation time |
| **BDS Sanitization** | Bounded disturb scan operations |

---

## 7. Visualization Plan

### 7.1 Performance Comparison Graphs

#### Graph 1: Deletion Latency vs File Distribution
```
X-axis: File Size Distribution (LogNormal, PowerLaw, Bimodal, Uniform)
Y-axis: Average Deletion Latency (µs, log scale)
Series: erasure, crypto, erasucrypto, vcsd
```

#### Graph 2: Deletion Latency vs Secure Deletion Ratio
```
X-axis: Secure Deletion Percentage (25%, 50%, 75%)
Y-axis: Average Deletion Latency (µs)
Series: 4 schemes × 4 distributions (16 lines)
Facets: TRIM pattern (end / mid)
```

#### Graph 3: Latency CDFs (Cumulative Distribution)
```
X-axis: Latency (µs, log scale)
Y-axis: Cumulative Probability (0-1)
Series: 4 schemes
Per-distribution subplots
```

#### Graph 4: Throughput Comparison
```
X-axis: Workload Type (lognormal, powerlaw, bimodal, uniform)
Y-axis: IOPS
Series: 4 schemes
Grouped bar chart
```

#### Graph 5: Data Migration Cost
```
X-axis: Scheme (erasure, crypto, erasucrypto, vcsd)
Y-axis: Data Migrated (GiB)
Facets: Distribution type
Stacked bar (total data + breakdown by reason)
```

### 7.2 VCSD Component Breakdown

#### Pie Chart: VCSD Deletion Latency Composition
```
Slices:
- PPK Puncture (%)
- PCD Generation (%)
- Block Erasure (%)
- BDS Sanitization (%)
- Planner Overhead (%)
```

**Data Source**: Parse VCSD debug logs and aggregate time spent in each operation.

### 7.3 Heatmap: Scheme Performance by Distribution × TRIM Pattern

```
Rows: File distributions
Columns: TRIM patterns
Cell color: Speedup vs baseline (erasure)
Annotations: Absolute latency
```

---

## 8. Implementation Scripts

### 8.1 Script Structure

```
trace_generator/scripts/
├── msr_to_vcsd_advanced.py          # Main augmentation script
├── file_size_distributions.py       # Distribution generators
├── trim_injector.py                 # TRIM insertion logic
├── generate_all_variants.sh         # Batch variant generation
└── analysis/
    ├── parse_vcsd_logs.py           # Extract VCSD component timing
    ├── compare_schemes.py           # Multi-scheme comparison
    ├── plot_performance.py          # Generate graphs
    └── plot_vcsd_breakdown.py       # Pie chart generator
```

### 8.2 Workflow Script (`generate_all_variants.sh`)

```bash
#!/bin/bash
# Generate all 24 trace variants from MSR-Cambridge traces

DISTRIBUTIONS=("lognormal" "powerlaw" "bimodal" "uniform")
TRIM_PATTERNS=("end" "mid")
SECURE_RATIOS=(25 50 75)

for dist in "${DISTRIBUTIONS[@]}"; do
  for pattern in "${TRIM_PATTERNS[@]}"; do
    for ratio in "${SECURE_RATIOS[@]}"; do
      python3 msr_to_vcsd_advanced.py \
        --input traces/MSR-Cambridge/hm_0.csv \
        --output traces/variants/${dist}_${pattern}_${ratio}.trace \
        --distribution $dist \
        --trim-pattern $pattern \
        --secure-ratio $ratio
    done
  done
done
```

---

## 9. Evaluation Execution Plan

### Phase 1: Trace Preparation (1-2 days)
1. Analyze MSR-Cambridge trace characteristics
2. Implement augmentation scripts
3. Generate all 24 trace variants
4. Validate trace format and statistics

### Phase 2: Simulation Runs (3-5 days)
1. Run 96 simulations (24 variants × 4 schemes)
2. Parallel execution on multiple cores
3. Monitor for failures, re-run if needed
4. Collect latency logs and stats

### Phase 3: Analysis & Visualization (2-3 days)
1. Parse all latency logs
2. Extract VCSD component breakdowns
3. Generate comparative graphs
4. Create summary tables

### Phase 4: Validation & Reporting (1 day)
1. Sanity-check results
2. Identify anomalies
3. Write final evaluation report

**Total Timeline**: ~7-10 days

---

## 10. Expected Outcomes

### 10.1 Hypotheses

1. **VCSD will show lower deletion latency than erasure-based** due to no data migration
2. **VCSD will outperform crypto-based** due to puncturable keys (no re-encryption)
3. **VCSD vs ErasuCrypto**: Comparable or better, with verifiability advantage (PCD)
4. **File distribution impact**: Power-law (large files) favors VCSD more than uniform small files
5. **TRIM pattern**: Mid-deletion will show VCSD's advantage under load

### 10.2 Key Findings to Extract

- Optimal file size range for VCSD effectiveness
- Sensitivity to secure deletion ratio
- Hybrid planner decision quality (PPK vs erasure vs BDS mix)
- PCD overhead (quantify verifiability cost)

---

## 11. Rigor & Trustability Measures

### 11.1 Reproducibility
- Fixed random seeds for file-ID assignment
- Deterministic TRIM injection
- Version-controlled config files
- Documented simulator parameters

### 11.2 Validation Steps
- Cross-check deletion counts (TRIM count = deleted file count)
- Verify LBA coverage (all LBAs belong to exactly one file)
- Sanity test: crypto-based latency ≈ constant (per-page overhead)
- Smoke test: run small trace end-to-end before full batch

### 11.3 Statistical Significance
- Multiple MSR traces (if available: hm, proj, rsrch, stg)
- Confidence intervals on latency metrics
- Avoid cherry-picking: report all results

---

## 12. Next Steps

1. **Immediate**: Check MSR-Cambridge trace availability and format
2. **Week 14 (April 26-30)**: Implement augmentation scripts
3. **Verify**: Test one trace variant end-to-end first
4. **Scale**: Generate all 24 variants, run evaluation
5. **Deliver**: Final presentation with graphs and insights

---

## Appendix: File-ID Assignment Algorithm

### Temporal Clustering Approach

```python
def assign_file_ids(trace_ops, file_sizes, distribution):
    """
    Cluster LBA accesses into files based on temporal locality.
    
    Heuristic:
    - Sequential writes within 1 second → same file
    - LBA range continuity → same file
    - File boundary when cumulative size ≈ target file size
    """
    file_id = 0
    current_file_lbas = []
    current_file_bytes = 0
    target_size = file_sizes[file_id]
    last_timestamp = 0
    
    for op in trace_ops:
        if op['type'] == 'W':  # Write operation
            # Check file boundary conditions
            if (op['timestamp'] - last_timestamp > 1e9 or  # 1 second gap
                current_file_bytes >= target_size * 0.9):  # 90% of target size
                
                file_id += 1
                current_file_bytes = 0
                if file_id < len(file_sizes):
                    target_size = file_sizes[file_id]
            
            op['file_id'] = file_id
            current_file_bytes += op['size']
            last_timestamp = op['timestamp']
        
        elif op['type'] == 'R':
            # Reads inherit file-ID from written LBA
            op['file_id'] = lookup_lba_file_id(op['lba'])
    
    return trace_ops
```

---

**End of Evaluation Plan**
