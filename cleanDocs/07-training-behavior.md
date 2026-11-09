# PHY training behavior

## Scope and confidence

This document describes the behavioral model required to understand and replace LPDDR5 PHY training: what is swept, what depends on what, how centers and margins are represented, and how success should be decided.

The stage names, message-block controls, output units, and selected diagnostics are observed interface facts. The detailed candidate ordering, exact pass/fail statistics, internal retry decisions, and numeric acceptance thresholds of the reference firmware are **not fully known**. Statements about a general sweep/center algorithm are a functional model, not a promise that every reference stage uses the same implementation.

Training runs on the PHY PMU. The host supplies a validated message block and initial PHY state, then services the mailbox without accessing training CSRs.

## Training domain

Training is constrained by:

- enabled P-state and data rate (`Pstate`, `DRAMFreq`, `PllBypassEn`, `DfiFreqRatio`);
- channel/rank population (`CsPresentChA/B`), device organization (`X8Mode`), and DQ counts (`EnabledDQsChA/B`);
- initial mode-register values and electrical overrides;
- enabled stages (`SequenceCtrl`) and CA/CS options (`CATrainOpt`);
- per-point comparison count (`BitTimeControl`);
- pattern selection, VREF ranges/steps, delay-scan shifts, and 2D optimization controls;
- initial calibrated state supplied by the pre-training tables.

DQ-related work is per channel, rank, byte/lane, and potentially bit. CA/CS work is per command group/channel/rank. Absent ranks and disabled DQs must not contribute results or failures.

The main adjustable dimensions are:

| Dimension | Typical target | Result units |
|---|---|---|
| Write-clock/write-data timing | PHY transmit path and DRAM sampling | PHY delay taps |
| Read-gate/read-DQS timing | PHY receive sampling | PHY delay taps |
| RX voltage reference | PHY receiver | PHY DAC codes/steps |
| TX/device voltage reference | DRAM receiver | DRAM DAC codes/steps |
| Duty cycle | DRAM, PHY RX, or PHY TX | DCA codes |
| DRAM transmit equalization | DRAM transmitter | DFE fields/coefficients |
| Command/address and CS timing | CA/CS groups | PHY delay taps and MR values |
| Maximum read latency | DFI read path | DFI clocks |

Tap-to-time and DAC-code-to-voltage conversions are PHY/DRAM-specific and are not established by this specification.

## Dependency model

The following is a **functional dependency order**, not a claim that bit numbers or every observed major-message ID directly encode execution order.

1. **Device/PHY initialization (`SequenceCtrl[0]`)** establishes reset, clocking, basic MR state, and a working command interface. All later stages depend on it.
2. **Write leveling (`[1]`)** establishes coarse WCK/CK and per-lane write timing. Reliable subsequent write-based tests depend on it.
3. **Read gate / receive enable (`[2]`)** finds a usable read-data capture window.
4. **Read DQS (`[3]`)** refines read sampling; read-eye/VREF optimization depends on a working gate/strobe.
5. **Write DQ (`[4]`)** refines the write eye after basic write alignment and a usable readback/test mechanism exist.
6. **DCA (`[6]`, `[7]`, `[8]`)** refines duty cycle in the DRAM and PHY. These stages consume already functional read/write paths and can change the timing results that later stages use.
7. **Maximum read latency (`[9]`)** measures the passing DFI latency and adds `DFIMRLMargin`; it depends on functional read training.
8. **DRAM TX DFE (`[10]`)** optimizes the DRAM transmit path and therefore depends on an operational read path. Later read-margin evaluation must use the final DFE setting.
9. **CA/CS (`[12]`)** requires a functioning command interface and may include CA VREF and LPDDR5 CS training. The exact placement of every CA/CS substep relative to all DQ/DCA substeps is not established by the available evidence.

The firmware also contains retraining loops. `DisableTrainingLoop` can suppress loops involving write leveling, read gate/read-DQ calibration, and write training. Consequently, a one-pass dependency graph is insufficient as an execution trace: later calibration may revisit earlier stages. A replacement should iterate only with explicit bounds and convergence checks.

## Candidate evaluation and eye construction

A behavioral training stage performs the following operations:

1. Select a candidate delay, VREF, DCA, DFE, or latency configuration.
2. Apply it to the correct channel/rank/lane or group.
3. Transfer the selected training pattern and compare the returned data.
4. Repeat comparisons for the configured number of bit times.
5. Classify the candidate and construct passing/failing regions.
6. Select an interior operating point rather than simply the first passing point.
7. Program the selected value and publish applicable result/margin fields.

`BitTimeControl[2:0]=n` selects `2^n` times the base comparison count per candidate. The base count, mismatch tolerance, PRBS/pattern lengths, and whether any mismatches are allowed are unknown. Thus, this document cannot define a bit-error-rate threshold for reference compatibility.

For 2D training, candidates span delay and voltage. The selected point optimizes a combined margin:

```text
score = delay_margin * delay_weight + voltage_margin * voltage_weight
```

Separate RX and TX weight pairs occupy message offsets `0x20/0x21` and `0x17/0x18`. Each weight is 0–31. One may be zero; if both are zero, each is treated as 1.

`Train2DMisc` additionally controls iteration count, seeds, contour output, optimization method, and RX VREF runtime range selection. The exact optimizer need not be copied to satisfy the external contract, but its resulting CSR/MR state and reported margins must be coherent.

## Margins and result interpretation

Each populated channel/rank reports four one-byte margins:

| Field | Meaning | Unit |
|---|---|---|
| `RxClkDly_Margin` | Selected RX timing center to nearest failing region | PHY DLL/tap steps |
| `VrefDac_Margin` | Selected RX VREF center to nearest failing region | PHY DAC steps |
| `TxDqDly_Margin` | Selected TX DQ timing center to nearest failing region | PHY DLL/tap steps |
| `DeviceVref_Margin` | Selected device VREF center to nearest failing region | DRAM DAC steps |

Each is the **minimum across all eyes in that timing group**, not an average and not the full eye width. For one dimension with selected coordinate `c`, nearest failing coordinates `L < c` and `R > c`, the conceptual value is:

```text
margin = min(c - L, R - c)
```

The exact treatment of scan boundaries, unobserved failures, truncated eyes, rounding, and saturated values is not established. Do not infer `2 * margin` as a standards-compliant eye width without resolving those cases.

`DFIMRLMargin` is different: it is an input safety increment in DFI clocks added to the smallest passing maximum-read-latency result. It must account for maximum expected positive `tDQSCK` drift over operating voltage and temperature; the interface recommends at least 1. This is not one of the four output eye margins.

Trained VREF/DFE/DCA outputs are copied to MR12/14/15/24/30 inputs for quickboot as specified in `05-message-block.md`. The final live CSR and MR state, not the streamed debug text alone, is authoritative.

## Observability and diagnostics

Streaming messages expose internal measurements but do not replace result collection. A reference full-training capture across two controllers contained 9,153 streamed messages and 124 distinct IDs. Confirmed framing is specified in `08-mailbox-and-streaming.md`.

Useful but incomplete observations include:

- ID `0x007e`, payload length 31: a packed per-bit DQ-related array; older notes describing “32 values” should not be read as 32 payload words.
- ID `0x0104`, length 1: emitted in loops over a 128-byte array; 1,024 occurrences were captured.
- IDs `0x00b1` and `0x011a`: pack eight bit results into a byte.
- ID `0x0042`, length 7: per-lane DQ report with an index and VREF/delay-related values.
- IDs `0x01e0` and `0x01e9`: frequency/PLL-related reports containing `0x1900` (6400).

Payload schemas for most IDs remain unknown. Do not use these inferred meanings as acceptance inputs.

Observed firmware safeguards include a PHY signature/version check, a diagnostic when a useful window is smaller than 12 (not necessarily fatal), and a diagnostic after 64 intervals without valid samples. Their presence does not establish a universal margin threshold or retry policy.

## Completion, failure, and acceptance

### Observed reference behavior

- Enabled stages are selected by `SequenceCtrl`; the PMU emits major `0x07` on normal completion and `0xff` on its fatal path.
- `CsTestFail=0` means no rank failure was reported; nonzero means at least one rank failed.
- The host collects message, PHY-parameter, and ACSM results after the PMU phase.
- The exact rule by which a candidate or complete stage is accepted is unknown. In particular, no minimum values for the four output margins, no universal mismatch allowance, and no complete convergence/retry threshold have been recovered.
- The reference error-return path is inconsistent and is not reliable acceptance evidence; see `08-mailbox-and-streaming.md`.

### Recommended acceptance for a new implementation

This is new policy, not recovered reference behavior:

1. Require normal completion, no PMU timeout/protocol error, and `CsTestFail=0` for every populated rank.
2. Verify that every enabled stage produced all required live CSR, MR, message, PHY-parameter, and ACSM results.
3. Choose explicit per-margin minima from PHY/DRAM documentation and board signal-integrity validation. Record the actual margins with the result. Until those minima are established, report margins as diagnostic data rather than inventing a numeric pass threshold.
4. Bound retraining by iteration count and time. Fail if no passing region exists, results do not converge, or a platform margin minimum is not met.
5. Revalidate when data rate, topology, electrical parameters, firmware/result format, or operating corners change.
6. Run the post-activation memory smoke test independently. A wide reported eye does not prove DDRC address mapping is correct; a smoke-test pass does not prove adequate signal margin.

## Unresolved details

A fully cycle-compatible independent trainer still requires:

- exact pass/fail comparison statistics and base bit-time count;
- scan ordering, step rounding, truncated-eye handling, and center-selection details for each stage;
- numeric margin/acceptance thresholds and convergence/retry criteria;
- full streamed payload schemas and major-stage-to-algorithm mapping;
- precise tap/DAC units and CSR programming sequences for every result.
