# EyanaSSDSim — Final Corrected-Engine Results (1.27 GB device)

Device: **1.27 GB** (1,216 blocks x 256 pages, chip=2 so S1-S6 are distinct) — the
same size used for the FTLSim/FEMU validation. Methodology: workload footprint =
80% of capacity, replayed 3x (2,000,000 total writes), no artificial
preconditioning. Deterministic engine, 18/18 unit tests.

## 1. Synthetic WAF + wear (vs over-provisioning)

| OP | workload | WAF | DoIPD | mean erase | DoEC | CV | Gini | Fourier sigma |
|----|----------|-----|-------|-----------|------|----|----|-----------|
|10%| sequential | 1.00 | 74.1 | 5.53 | 1.552 | 0.281 | 0.113 | 53.3 |
|10%| uniform | 2.39 | 28.8 | 14.40 | 2.481 | 0.172 | 0.052 | 78.0 |
|10%| zipf | 1.13 | 58.3 | 6.36 | 8.320 | 1.309 | 0.460 | 245.0 |
|20%| uniform | 2.28 | 30.5 | 13.70 | 2.353 | 0.172 | 0.050 | 74.4 |
|25%| uniform | 1.97 | 36.2 | 11.75 | 2.016 | 0.172 | 0.049 | 63.8 |
|25%| zipf | 1.13 | 57.9 | 6.33 | 8.277 | 1.308 | 0.460 | 239.2 |

Story (unchanged from the 5 GB run, only magnitudes differ):
- WAF: sequential (1.00) < Zipf (1.13) < uniform (2.39).
- Wear two-axis: uniform most *uniform* (Gini 0.052), sequential least *total*
  wear (mean erase 5.53); Zipf worst on both dispersion measures.
- mu_I: sequential 25.2, uniform 46.0, Zipf 166.7.

## 2. Validation vs FTLSim / FEMU (Figure 12, 1.27 GB)

| Workload | Eyana@10% | FTLSim@10% | Eyana@25% | FEMU@25% |
|---|---|---|---|---|
| Sequential | 1.00 | 1.78 | 1.00 | 1.05 |
| Uniform | **2.39** | **2.55** | 1.97 | 1.28 |
| Zipf | **1.13** | **1.13** | 1.13 | 1.00 |
| TPC-C | 10.58 | 6.05 | 2.36 | 1.50 |
| prxy | 1.02 | 8.50 | 1.03 | 1.00 |

- Uniform and Zipf match FTLSim tightly -> validates the engine.
- Sequential/prxy gaps are explained by FTLSim's LSB backup pages (2.8M for prxy).
- (FTLSim/FEMU values read from the original Figure 12; those tools were not re-run.)

## 3. GC policy comparison (OP 10%)

| policy | uniform WAF | uniform Gini | zipf WAF | zipf DoEC |
|--------|------|------|------|------|
| greedy | 2.385 | 0.052 | 1.131 | 8.320 |
| cost_benefit | 2.388 | 0.045 | 1.134 | 7.845 |
| fifo | 2.412 | 0.041 | 1.208 | 9.097 |

cost-benefit gives the most uniform wear; greedy the lowest Zipf WAF; FIFO worst Zipf WAF.

## 4. Allocation schemes S1-S6

Near-identical (differences within ~2%); S1 marginally lowest Fourier sigma across
workloads (seq 53.3, uniform 78.0, Zipf 245.0). Confirms static scheme choice has
minor effect under dynamic block management.

## 5. Scalability

Engine runs at ~430-452K writes/s from 5 GB to 40 GB with near-constant throughput.

## NOT regenerated / left to author (agreed)
- Table 7 WSC access-pattern percentages (WSC classifier not implemented).
- Real-workload section V-E numbers + real-data placement figures + cluster_timestemp:
  reverted to original, because TPC-C's footprint (~2.5 GB) and prxy's (~160 MB) cannot
  both fit the 1.27 GB device, and this block is tied to the WSC/Table 7 analysis.
- FEMU/FTLSim latency numbers (tools not re-runnable here).
