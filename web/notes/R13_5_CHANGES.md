# R13.5 Inlier Reacquisition / Anchor Quarantine

R13.5 is based on R13.4 Unified Information-Adaptive + Vibration Consensus Guard.
It specifically addresses the real test where TAG-01 was moved about +60 mm while one or more UWB links became corrupt, and TAG-02 was dragged by a persistently bad A1 link.

## Main changes

1. **Inlier-subset displacement consensus**
   - Each current Anchor measurement is converted into a normal-displacement candidate using the model Jacobian H = dr/dΔn.
   - A robust median/MAD cluster is formed.
   - A bad link no longer has to agree with the good links for a real displacement to be accepted.
   - Large/vibration events require at least 3 mutually consistent links when 3+ links are available.

2. **Hard outlier rejection**
   - A current link with a displacement candidate or innovation far outside the consensus is given zero influence for that update.
   - Soft covariance weighting is still used for moderate quality differences; hard rejection is reserved for severe outliers.

3. **Single-link quarantine**
   - A link that is a severe outlier for repeated cycles is temporarily quarantined from the estimator.
   - It is still allowed to recover through later healthy measurements/health sweeps.
   - This is intended to stop cases like TAG-02/A1 slowly dragging the entire estimate.

4. **Automatic large-step reacquisition**
   - During a vibration/large-motion event, if at least 3 independent good links repeatedly agree on a 25–90 mm displacement, the old sliding window is cleared and the state is reacquired at the new Δn.
   - This fixes the R13.4 false rejection where a real ~60 mm motion could be held at the old position forever.

5. **Common-mode packet burst rejection**
   - A packet-wide range shift that looks more like a common radio/timestamp offset than H·Δn is rejected.
   - It is not used to falsely quarantine all Anchors.

6. **All-Tag diagnostic CSV retained and expanded**
   - Inlier/outlier Anchor IDs
   - quarantine state
   - reacquisition state/target
   - per-link hard_rejected / quarantined flags and robust weights
   - existing raw/corrected range, residual, sigma, H, information, CIR/NLOS diagnostics remain.

## Firmware

R13.5 does **not** change the R13.4 Tag RF protocol. Use the included Tag firmware (same R13.4 behavior) and the existing Anchor responders.
The estimator/scheduler/CSV changes are in the PC UI/server package.

## Important validation limitation

R13.5 is designed to stop catastrophic/fault-driven state corruption and to reacquire genuine large steps with bad links present. It does **not** mathematically guarantee that every ordinary stationary synthetic noise seed remains inside ±5 mm at every sample. Some legacy stationary synthetic seeds still show 7–14 mm instantaneous peaks and bias. Actual BU03 logs are still needed to tune per-link calibration/noise models for the final ±5 mm objective.
