# Open questions and specification gaps

## Scope

This chapter catalogs the structural, behavioral, and hardware details that are **not** established by this specification. An independent implementation team must treat these items as open engineering questions to be resolved through controlled hardware experimentation or memory datasheets—never by consulting proprietary vendor firmware source code or binaries.

---

## Resolved structural facts (for historical clarity)

Two questions that were open in early hardware bring-up are now **fully resolved**:

1. **Second DDR controller base address (Resolved)**: The K3 SoC exposes two complete controller instances: Instance 0 at `DDRC = 0xcb000000`, `CFG = 0xcb600000`, `DPHY = 0xcb800000` and Instance 1 at `DDRC = 0xcc000000`, `CFG = 0xcc600000`, `DPHY = 0xcc800000` (see [02-addressing-and-access.md](02-addressing-and-access.md)).
2. **Exact `phypara` serialization order (Resolved)**: The 5170 halfwords collected after training and restored during quickboot follow the exact 69-segment table documented in [appendix-phypara-order.md](appendix-phypara-order.md).

---

## Unresolved structural and protocol questions

### 1. DDRC timing derivation for non-reference parts (host side)

Host-side DDRC timing registers depend on the selected LPDDR5 part, rank count, x8/x16 organization, and data rate (`4800` or `6400` MT/s). For parts other than the reference `MT62F2G32D4DS`, host timing values must be derived from the LPDDR5 device datasheet (`ceil(ns * rate / 2000)`) and validated on hardware.

### 2. Full payload schemas for the 124 streaming message IDs

The streaming wire framing (`header = (id << 16) | len` followed by `len` 32-bit payload words) is established (see [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md) and [appendix-streaming-catalog.md](appendix-streaming-catalog.md)), and selected patterns are identified (`0x007e` 31-word array, `0x0104` 128-byte sweep, `0x00b1`/`0x00b4`/`0x011a` packed 8-bit lane masks, `0x01e0`/`0x01e9` frequency reports). However, the exact field-by-field schema, signedness, and physical units of the remaining streaming IDs remain unknown.

### 3. Major stage byte to training-phase mapping

A full-training capture emits the major message sequence `0d 00 01 02 fd 06 01 02 fd 04 03 09 07`. Aside from `0x07` (normal completion), `0x08` (streaming introducer), and `0x000000ff` (fatal error), the exact one-to-one mapping between intermediate stage codes (such as `0x0d`, `0xfd`, `0x06`, `0x04`, `0x03`, `0x09`) and `SequenceCtrl` algorithm steps is not confirmed.

---

## Unresolved training and behavioral questions

### 4. Internal eye-acceptance and retry criteria

How the PMU training algorithm decides that a candidate delay/VREF point or an entire 1D/2D eye passes—including the base comparison count for `BitTimeControl = 0`, allowed bit-error count (if any), minimum acceptable eye width/margin (`RxClkDly_Margin`, `VrefDac_Margin`, `TxDqDly_Margin`, `DeviceVref_Margin`), and maximum retraining loop iterations—is not established.

### 5. Full DQ read/write eye sweep vs. abbreviated initialization

Hardware tests demonstrated that an abbreviated PMU initialization (~5 ms, completing `DevInit` without the multi-second per-lane DQ eye sweeps) can still emit `0x07` and boot Linux on the reference board at nominal room temperature, with single-thread `membench` bandwidth/latency within ~2% of the ~5.6 s full-training path. Which exact PHY delay/VREF CSRs differ between an uncalibrated default state and a fully swept 5.6 s run, and how much voltage/temperature margin is lost when DQ sweeps are omitted, must be quantified by register-diff and corner testing.

### 6. Host post-activation smoke-test behavior in source-built loaders

When building the unsigned first-stage loader from source, the post-activation host memory smoke test (`0x4000`-byte pattern/complement check) was observed to hang even when paired with the unmodified reference PMU firmware, whereas the prebuilt signed factory FSBL completes its post-training flow. Resolving whether this discrepancy stems from compiler code generation, 64-bit pointer/addressing above 4 GiB, cache/MMU state, or DDRC port/region setup requires a controlled host-side investigation.

### 7. Multi-P-state training and P-state SRAM (`0xa0000`)

The message block supports `Pstate` selection and a 15-P-state DMA mode (`Pstate[7]`), and the DPHY exposes a 16,384-halfword P-state SRAM window at `0xa0000`. However, only single-P-state operation was observed on the reference board, and P-state SRAM restoration is inactive in the observed quickboot sequence.

---

## Unresolved hardware questions

### 8. Bit-level definitions of internal PHY CSRs (`0x90xxxxxx` / DPHY indices)

While CSR access widths (16-bit), lane base addresses (`0x90020158`, `0x9002016e`, `0x9003e126`, `0x90060002`), and the 69 `phypara` serialization segments are known, the bit-level functional descriptions and read/write side effects of most internal PHY CSRs are undocumented.

### 9. Expected PHY hardware signature/version value

The PMU firmware checks a hardware signature/version register early in execution and aborts on mismatch, and writes `PmuRevision` (`0x002`) to the message block. The complete table of accepted silicon revision codes is not cataloged.

### 10. Temperature derating curves and 4800 MT/s hardware validation

DDRC derating registers (`0x00b04`–`0x00b08`, `0x10104`–`0x10108`) and 4800 MT/s clock/timing tables exist in the interface, but hardware validation in this study was performed at 6400 MT/s under ambient bench temperature.

---

## Policy for handling open questions in a clean-room implementation

- **Separate measured facts from assumptions**: Document clearly in code comments whenever a timeout, margin floor, or retry count is an implementation choice rather than a hardware requirement.
- **Treat unknown message codes and reserved bytes as opaque**: Log unknown major/streaming IDs without aborting, and preserve all 320 message-block bytes and 5170 `phypara` halfwords without filtering or reordering.
- **Close gaps by hardware measurement**: Validate any new timing calculation, eye-margin threshold, or host memory test through controlled bench experiments (recording artifact hashes, topology, and serial logs) rather than assuming Linux login or benchmark speed implies full calibration equivalence.

## Related chapters

- [04-parts-topologies-and-timings.md](04-parts-topologies-and-timings.md)
- [07-training-behavior.md](07-training-behavior.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
