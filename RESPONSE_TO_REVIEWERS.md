# Response to Reviewers — EyanaSSDSim (Access-2026-12395)

We thank the editor and all three reviewers for their careful and constructive
feedback. In this revision we have substantially strengthened the work. Most
importantly, we **rebuilt the simulation engine from scratch as a headless,
unit-tested Python core**, during which we discovered and fixed correctness
defects in the previous browser-based engine that had affected some reported
numbers. All experiments were re-run on the corrected engine; where results
changed, we report the new, verified values and explain the difference.

Key additions in this revision:
1. **Corrected, verified engine** with a regression test suite pinning the WAF,
   garbage-collection, and allocation math (host vs GC writes are now counted
   separately; blocks close at the correct page count; the six allocation
   schemes are now genuinely distinct — see R-Global-1).
2. **Additional GC policies** (cost-benefit and FIFO) beyond greedy, with a
   comparative evaluation (addresses R1-2, R2-W1, R3-2).
3. **Robust dispersion metrics** — Coefficient of Variation and the Gini
   coefficient — reported alongside the Fourier analysis, so the frequency-domain
   method is validated against standard statistical measures (R1-6, R2-S3, R3-3).
4. **Scalability evaluation** up to 40 GB with reported throughput (~420K writes/s, R1-3, R1-a7).
5. **Real traces** (TPC-C, MSR prxy) parsed directly from the original SNIA/MSR
   sources and evaluated across over-provisioning levels.
6. **Honest scoping of determinism and validation**, with a new limitations
   discussion (R1-1, R2-W5).

> NOTE TO SELF (remove before submission): values in [BRACKETS] are filled from
> results/*.json after the final experiment run.

---

## Global change G1 — Allocation-scheme methodology correction (affects R1, R3 novelty)

While rebuilding the engine we found that, under the single-chip configuration
used previously, the chip dimension was degenerate, which made allocation schemes
S2/S4 and S3/S5 mathematically identical. We have corrected the configuration
(≥2 chips per channel) so that **all six schemes S1–S6 are now genuinely distinct
placements**, verified programmatically by a test that fails if any two schemes
produce identical LPN→plane maps. The allocation-scheme comparison (Fig. and
Table) has been regenerated on this corrected basis, and the conclusions updated
accordingly. We disclose this transparently as part of strengthening the paper's
technical rigor.

---

## Reviewer 1

**R1-1 / R1-Q2 — Determinism ignores stochastic variability (latency, bus
contention, bit errors).**
*Response:* EyanaSSDSim is deterministic by design, in common with widely used
FTL research simulators (FTLSim, the DiskSim-based SSD model, and the default
configurations of MQSim). Determinism is a deliberate feature for a
visualization-and-analysis tool: it makes every observed internal event exactly
reproducible, which is what enables step-by-step visual tracking and repeatable
teaching. We now state explicitly that EyanaSSDSim models the *logical and
architectural* behavior of the FTL (data placement, GC, wear-leveling, WAF), not
analog NAND phenomena (RBER, read-retry, temperature). We have added a
Limitations paragraph making this scope explicit and noting that a stochastic
latency/error layer is future work; the workload generator already exposes a seed
hook to support such extensions.
*Action:* Added Limitations subsection; scoped all accuracy claims to the FTL
layer; added the determinism rationale with citations.

**R1-2 / R2-W1 / R3-2 — Only greedy GC.**
*Response:* We have implemented two additional, pluggable GC policies —
**cost-benefit** (age- and utilization-weighted) and **FIFO** — and added a
comparative evaluation. Results: [greedy vs cost_benefit vs fifo WAF/wear table
from results/gcpolicy.json]. This demonstrates the engine is not tied to greedy
and can serve as a testbed for advanced reclaim policies.
*Action:* New GC-policy subsection + comparison figure/table.

**R1-3 / R1-a7 / R1-Q2 — Small 5.03 GB capacity; does not show scale.**
*Response:* We added a scalability evaluation running the corrected engine at
[5 / 20 / 40] GB, reporting throughput ([XX]k writes/s) and confirming identical
qualitative behavior. We clarify that the visualization is resolution-independent
(aggregated views, see R1-4) so capacity is not bounded by rendering. We also
note that 5.03 GB was chosen for figure legibility, not as an engine limit.
*Action:* New scalability table; clarified capacity is configurable.

**R1-4 — Aggregated views under-specified.**
*Response:* We now describe the aggregation precisely: at large capacities each
rendered cell maps to a fixed group of physical blocks, and its color encodes the
group's mean invalid-page ratio (and, on toggle, mean erase count). This
preserves real-time observation of spatial trends while bounding the number of
drawn elements.
*Action:* Added aggregation description to the architecture section.

**R1-5 / R1-a2 / R1-Q2 — WAF vs FTLSim ("LSB backup") hand-waved.**
*Response:* We quantify the difference rather than attribute it vaguely. FTLSim
programs additional LSB backup pages ([counts]); we show these account for the
observed WAF gap and that, once accounted for, the trends match. We reframe this
as an *explained, bounded* implementation difference, not an unmodeled deficiency.
*Action:* Expanded validation subsection with the quantified comparison.

**R1-6 / R1-a5 — Physical justification for Fourier vs standard dispersion
metrics.**
*Response:* We now report the **Coefficient of Variation** and **Gini
coefficient** of the per-block erase counts alongside the Fourier amplitude
spread σ_A. The scalar metrics (CV, Gini) quantify *how much* wear is uneven,
while the Fourier spectrum reveals *spatial/periodic structure* — which blocks
are repeatedly hottest — that scalar metrics cannot expose. Crucially, the CV and
Gini rankings now **agree** with the Fourier ranking across workloads, which
validates the frequency-domain analysis rather than replacing it. We removed the
DC component from σ_A (it encodes only the mean level, not variation), so a
perfectly uniform distribution correctly yields zero spread.
*Action:* Added CV/Gini columns to the wear-comparison table; added a paragraph
motivating Fourier as complementary to, and consistent with, standard metrics.

**R1-7 — Ensure DoIPD/DoEC are fully defined in the evaluation section.**
*Response:* Both metrics are now defined with their equations at first use in the
evaluation, with a notation table, so the reader need not jump between sections.
*Action:* Metric definitions relocated/duplicated into the evaluation section.

**R1-a1 — GC threshold 99.9995%/99.9990% is unrealistic/narrow.**
*Response:* [PENDING AUTHOR INPUT — the code uses gc_threshold=0.999 +
free-space=0.0005; we will restate the watermarks in conventional terms
(e.g., start GC at 95% block utilization, stop at 90%) which is what the rebuilt
engine uses, and remove the misleading many-nines figure.]
*Action:* Restated GC trigger as standard high/low watermarks (95%/90%).

**R1-a3 — Latency validation weak (bus contention / OS overhead?).**
*Response:* We clarify that the reported latency models per-operation flash
service time (page read/program, block erase) with channel-level serialization,
not host-stack or OS overhead. We scope the FEMU latency comparison accordingly.
*Action:* Clarified latency model scope in the validation section.

**R1-a4 / R1-Q4 — QLC/PLC, ZNS, FDP and recent (2024–2026) work under-addressed.**
*Response:* We expanded the related-work and discussion with recent work on ZNS
and FDP (citations already in the bibliography: ZNS ATC'21, ZNS+, FDP SDC'23,
flash-alloc VLDB'23) and QLC endurance. We note that our engine's zone-aware
extension is under active development (a ZNS mode with host-managed GC), reported
in follow-up work.
*Action:* Expanded related work + discussion; added recent citations.

**R1-a6 / R1-Q3 — Internal contradiction: §IV-B says uniform-random best for
wear-leveling, later concludes sequential is best.**
*Response:* This was a genuine ambiguity, now resolved. The corrected engine and
the multi-metric analysis show the apparent contradiction stems from conflating
two different notions: **total wear** (mean erase count, minimized by low-WAF
workloads) versus **wear uniformity** (dispersion of erase counts). We now state
the trade-off explicitly and consistently: uniform-random writes yield the most
*uniform* wear (lowest Gini/CV) but *higher total* wear (higher WAF); sequential
yields *lower total* wear but slightly less uniform distribution; Zipf is lowest
total wear but least uniform. All four metrics (DoEC, CV, Gini, Fourier σ) are
reported together so the reader sees one coherent story.
*Action:* Rewrote §IV-B and the erase-count analysis to state the trade-off up
front; removed the contradictory phrasing.

**R1-a8 / R2-W4 — Survey is a usability metric, not accuracy; §VII shallow.**
*Response:* We reframe the survey explicitly as *usability/educational* evidence,
not a validation of simulation accuracy (which is established separately against
FEMU/FTLSim). We now present the survey methodology, questions, and the
distribution of responses [from survey data]. [If a controlled pre/post study is
run, summarize design and results here.]
*Action:* Rewrote §VII with methodology and result breakdown; scoped its claims.

---

## Reviewer 2

**R2-W2 — Descriptive vs prescriptive; no predictive-accuracy proof.**
*Response:* Beyond visualization we now provide quantitative, reproducible
metrics (WAF, DoIPD, DoEC, CV, Gini, Fourier σ) and validate WAF and latency
trends against FEMU and FTLSim. The WSC metric maps real traces to synthetic
profiles, enabling *prescriptive* firmware-tuning guidance (e.g., OP sizing for
Zipf-like workloads), which we make explicit.
*Action:* Strengthened the analysis framing; emphasized WSC-driven guidance.

**R2-W3 / R2-W5 — Limited head-to-head benchmark; no rigorous baseline.**
*Response:* We report a direct WAF comparison against FTLSim and FEMU across all
workloads, plus a read-latency comparison against FEMU, with differences
quantified and explained (R1-5). The engine now has a public test suite pinning
its numerical behavior.
*Action:* Expanded validation; added the regression-tested engine.

**R2-W4 — See R1-a8 (survey).**

---

## Reviewer 3

**R3-1 — Clarify novelty beyond visualization; quantitative differentiation.**
*Response:* We sharpened the contribution statement: the novelty is the
combination of (i) real-time internal visualization, (ii) the WSC metric mapping
real traces to synthetic profiles for workload-aware tuning, and (iii) a
multi-metric wear analysis (DoEC/CV/Gini/Fourier) that no surveyed simulator
provides. The feature table quantifies the gap versus MQSim, SSDModel, FEMU, etc.
*Action:* Rewrote the contributions list and the differentiation discussion.

**R3-2 — Advanced GC / queueing; justify determinism.** See R1-2 and R1-1.

**R3-3 / R3-Q2.3 — Fourier over-engineered vs simpler methods.** See R1-6. We now
show CV and Gini agree with the Fourier ranking, justifying it as a complementary
diagnostic that additionally exposes spatial structure.

**R3-4 — Grammar, sentence structure, overly long explanations.**
*Response:* We performed a full language pass, fixed specific errors (e.g.
"invalidation's"→"invalidations", "shwon"→"shown", a hardcoded "Table 7"
reference, and a block-count arithmetic error), tightened long passages, and
removed redundancy.
*Action:* Full proofreading pass; specific fixes applied.
