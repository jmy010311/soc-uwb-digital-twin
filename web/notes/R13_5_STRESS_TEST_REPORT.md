# R13.5 Stress / Fault Injection Report

## Why these tests were added

The latest real test exposed two different failure modes:
- TAG-01: a real ~60 mm displacement occurred while one or more UWB links became grossly inconsistent. R13.4 rejected the real displacement because it required overly conservative consensus.
- TAG-02: the structure remained fixed but a bad Anchor link (especially A1 in the captured CSV) drifted and gradually pulled the estimator away from the correct state.

R13.5 therefore tests real motion and sensor faults separately, and then mixes them randomly.

## Deterministic fault cases

- Real +60 mm step + one Anchor corrupted by +600 mm: **PASS**. Final ~61.57 mm; bad Anchor quarantined; automatic reacquisition fired.
- Stationary + one Anchor slowly drifting by -140 mm: **PASS**. Maximum false displacement ~0.86 mm; drifting Anchor quarantined.
- 2-vs-2 contradictory large displacement during vibration: **PASS**. No reacquisition; maximum false displacement ~4.42 mm.
- Permanent one-Anchor dropout + real +25 mm displacement: **PASS**. Final ~24.94 mm.
- Three-Tag shared event: TAG-01 real +60 mm + catastrophic A2, TAG-02 stationary + bad A1, TAG-03 healthy but same vibration: **PASS**.
  - TAG-01 final ~60.86 mm, reacquired, A2 quarantined.
  - TAG-02 peak false displacement ~3.66 mm, A1 quarantined.
  - TAG-03 peak false displacement ~0.69 mm.

## Randomized matrices

### Stationary-only 48-seed / 3-minute-per-seed fault matrix
The true displacement is always 0. Faults are injected irregularly:
- slowly drifting single links
- catastrophic single-link spikes
- packet-wide common-mode bursts
- one/two Anchor dropouts
- random shared vibration/footsteps

Result: **PASS under the defined safety bound**.
- Worst false displacement: ~17.72 mm in the harshest synthetic seed.
- Worst final recovery after clean data: ~0.68 mm.
- The point of this matrix is to ensure no unbounded runaway and eventual recovery, not to claim ±5 mm under every extreme artificial fault combination.

### Dynamic 32-seed reacquisition matrix
Sequence of true structural states: +60, +20, -35, +45, 0 mm. Each step is combined with:
- one heavily corrupted Anchor for ~2 s
- random extra spikes
- random Anchor dropout
- vibration around the real motion

Result: **PASS**.
- Worst settled error: ~0.78 mm.
- Worst tracking error in the measured post-reacquisition interval: ~1.04 mm.
- 159 automatic reacquisitions observed across the matrix.

### Mixed 24-seed long random fault matrix
Includes real structural steps, persistent bad links, random spikes, common bursts, dropouts and vibration.
- Worst final recovery error: ~0.02 mm.
- Large peak error in this metric can occur immediately after a deliberately discontinuous true step because the metric counts estimator tracking latency as “fault error”; therefore stationary and dynamic matrices above are the more meaningful separated metrics.

## Regression tests

PASS:
- Surface-normal solver
- Information-ranging planner
- Robust sliding window
- Vibration consensus guard
- Stationary drift guard
- Runaway guard
- 3-Tag TDMA fairness / STOP loss / ACK loss / rejoin
- Multi-Tag calibration
- Calibration UI
- Unified estimator continuity
- Tracking / weak geometry handling

## Known remaining issue

The legacy stationary ±5 mm synthetic suite still fails on several noise seeds. This is not hidden in this release. R13.5 fixes the large-step/fault-handling logic; final millimeter-level stationary accuracy must be tuned using the actual all-Tag BU03 CSV data, especially per-link bias/noise/CIR distributions and real geometry.
