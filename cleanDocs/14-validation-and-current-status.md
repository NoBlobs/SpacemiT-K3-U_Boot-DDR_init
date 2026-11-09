# Validation and hardware measurements

## Scope

This chapter records the hardware measurements obtained on the SpacemiT K3 reference platform, defines the validation criteria required to verify an independent LPDDR5 bootloader or PMU firmware implementation, and documents the bench safety constraints that must be respected during testing.

## Reference platform

| Property | Value |
|---|---|
| Board | SpacemiT K3 Pico-ITX |
| SoC | SpacemiT X100, 16 RISC-V cores (8 + 8), up to 2.2 GHz |
| DRAM | LPDDR5 `MT62F2G32D4DS`, 2 ranks, x16 (`x8_mode = 0`), 6400 MT/s, 16 GiB total (8 GiB × 2 controllers) |
| Operating system | Fedora Linux (kernel 6.18.3+), 16 GiB visible |
| Serial console | 115200 8N1 |
| Boot chain | BootROM → FSBL (SPL + DDR init) → OpenSBI / UEFI (EDK2) → GRUB → Linux |

## Observed boot and initialization timings

| Boot path | Measured duration | Observation method |
|---|---:|---|
| Persisted **quickboot** | **9 ms** | Loader console timestamp with valid `DDRT` envelope in SPI flash |
| **Full training** (both controllers) | **5611 ms** – **5803 ms** (~5.6 s) | Loader console timestamp after invalidating the `DDRT` envelope |
| Partial / abbreviated initialization (DQ sweep omitted) | **~5 ms** | Experimental PMU firmware run that completed `DevInit` and mailbox handshake without full DQ eye sweeps |
| Total cold boot to Linux login prompt | **~14 s** | Serial console wall-clock measurement (quickboot path) |

Quickboot is approximately **620× faster** than full training. Conversely, if a supposed full-training run completes in only a few milliseconds (such as ~5 ms), that duration is diagnostic evidence that heavy per-lane DQ/eye calibration loops were skipped, even if the PMU still emits major completion code `0x07` and the board boots into Linux under nominal room-temperature conditions.

## Reference training envelope and trace baselines

### Captured reference `DDRT` envelope (`MT62F2G32D4DS`, 6400 MT/s, 2 ranks)

| Field / Metric | Observed value (per controller) |
|---|---|
| Envelope `magic` | `0x54524444` (`"DDRT"`) |
| `type` | `1` (LPDDR5) |
| `cs_num` | `2` |
| `data_rate` | `6400` |
| `MsgMisc` | `0x01` |
| `HdtCtrl` | `0x04` (detailed debug threshold enabled in captured sample) |
| Non-zero `phypara` halfwords | `3926` / `5170` |
| Non-zero `acsm` halfwords | `1845` / `4096` |

### Captured full-training mailbox traffic (`HdtCtrl = 0x04`, two controllers)

- **22 major stage messages** across the two controllers (`0d 00 01 02 fd 06 01 02 fd 04 03 09 07` on controller 0);
- **9,153 streaming messages** (`0x08` frames);
- **124 distinct streaming message IDs**.

### Reference firmware blob sizes and SHA-256 prefixes

When validating the host loader's decompression pipeline against the reference images, the decompressed payloads match these sizes and SHA-256 prefixes:

| Image | Decompressed size | SHA-256 prefix |
|---|---:|---|
| Full-training PMU firmware (IMEM) | 64,372 bytes | `523777e3f4d765b9...` |
| Quickboot PMU firmware (IMEM) | 25,756 bytes | `3ee755834b26caf9...` |
| Quickboot additional DMEM image | 1,420 bytes | `13f5974a4ae799ed...` |

## Memory bandwidth and latency baseline

Single-thread memory benchmark (`membench`, 256 MiB working set, 3 repetitions, compiled with `-O2`) measured under Linux on the reference board:

| Metric | Reference full training (~5.6 s) | Abbreviated PMU run (~5 ms) |
|---|---:|---:|
| Sequential read bandwidth | 8.516 GB/s | 8.599 GB/s |
| Sequential write bandwidth | 6.104 GB/s | 6.179 GB/s |
| Memory copy bandwidth | 3.191 GB/s | 3.250 GB/s |
| Pointer-chase latency (16 MiB) | 133.15 ns | 129.03 ns |
| Pointer-chase latency (64 MiB) | 166.80 ns | 167.07 ns |
| Pointer-chase latency (256 MiB) | 176.15 ns | 176.69 ns |

> **Important interpretation note:** Steady-state bandwidth and latency depend primarily on DDRC clock frequency and DRAM timing registers (`dramset1tmg*`, `dfitmg*`), not on analog PHY eye centering. Therefore, matching benchmark numbers (within ~2%) and booting Linux **do not prove** that PHY DQ training was executed or that adequate voltage/timing eye margins were achieved.

## Host-chain vs. PMU-firmware isolation during validation

When testing a locally built unsigned SPL/FSBL chain on the K3:

1. **Signed prebuilt FSBL vs. unsigned source-built FSBL**: The factory prebuilt FSBL is cryptographically signed; modifying bytes inside it causes BootROM signature verification to reject the image. An unsigned source-built FSBL is accepted by the BootROM unsigned path and serves as the test harness for custom DDR loader or PMU firmware code.
2. **Isolating host smoke-test issues from PMU firmware**: In the source-built SPL chain, an unresolved hang was observed in the post-activation host memory smoke-test routine even when loading the **unmodified reference PMU firmware**. Bypassing that specific host routine allowed both the reference PMU firmware and custom PMU firmware to boot into Linux.
3. **Validation rule**: Always test any host-loader modification with the reference PMU firmware first so host-side toolchain or addressing regressions are never misattributed to PMU training firmware, and never claim full memory verification passed if the post-activation smoke test was bypassed.

## Validation criteria for an independent implementation

An implementation claiming full compliance with this specification must verify:

1. **Stage and work coverage**: Confirm that all stages enabled in `SequenceCtrl` execute to completion, with training duration, major stage transitions, and per-lane calibration sweeps consistent with full 1D/2D training rather than an early bypass.
2. **Trained output completeness**: Verify `CsTestFail == 0`, non-zero trained VREF/DFE/DCA/CDD outputs in the 320-byte message block, non-zero eye margins (`RxClkDly_Margin`, `VrefDac_Margin`, `TxDqDly_Margin`, `DeviceVref_Margin`), and populated `phypara` (5170 halfwords) and ACSM (4096 halfwords) snapshots on **both** DDR0 and DDR1.
3. **Post-activation memory verification**: Pass a non-destructive `0x4000`-byte address-pattern and complement smoke test across the mapped address ranges of both controllers before enabling normal boot continuation.
4. **Persistence and quickboot cycle**: Save the `0x9800`-byte `DDRT` envelope, reboot, verify that quickboot accepts the CRC-32 and restores DRAM in ~9 ms, and re-verify memory stability.
5. **Stress and margin validation**: Run multi-threaded memory stress tests across cold/warm reboots, voltage/temperature corners, and all supported topologies (`4800`/`6400` MT/s, 1/2 ranks, x8/x16).
6. **Error path containment**: Verify that clock lock timeouts, mailbox timeouts, and PMU fatal code `0x000000ff` are caught and reported explicitly without hanging silently or enabling DDRC ports on an uncalibrated interface.
7. **Clean production binary**: Confirm all diagnostic progress markers and verbose UART trace hooks are disabled, and verify via `objdump` that no `r16`–`r25` instructions or broken `libgcc` helpers exist in the final ARC ELF.

## Safe bench observation policy

> **DANGER — Do not access PHY/PMU MMIO from Linux (`/dev/mem`):**
> On the reference board, reading the DPHY/PMU region from user space after Linux booted immediately stalled the SoC interconnect—causing total loss of serial console and network and requiring a physical power-plug disconnect. (An early attempt also used the wrong base `0xc0960000`, which is unmapped on K3 where Instance 0 DPHY is at `0xcb800000`, but the prohibition applies equally to the true `0xcb800000`/`0xcc800000` windows once the OS is running.)
>
> All DPHY, IMEM, DMEM, and PMU CSR reads/writes must be performed **inside the first-stage bootloader (SPL)** while the DPHY clocks and PMU power domain are active and `MICRO_CONT_MUX_SEL` (`0xd0000`) is set appropriately, or via JTAG.

Additional bench precautions:

- **Live DMEM message zeroing**: Reading `0x58000` directly after the PMU phase may return zeros because the message area is consumed/cleared; always inspect the 320-byte copy captured by the result-collection sequence.
- **UART baud-rate bottleneck**: Setting `HdtCtrl = 0x04` or `0x05` and printing all 9,153 streaming messages at 115200 bps emits ~1.4 MiB of text and adds tens of seconds to boot time. Use verbose streaming only during targeted debugging.
- **Post-experiment cleanup**: Always restore the known-good `FSBL.bin` and `bootinfo` `DDRT` envelope after bench experiments and verify that quickboot returns to ~9 ms.

## Reproducing the baseline measurements

1. **Quickboot timing**: Boot normally with a valid `DDRT` envelope in `bootinfo` (`+0x10000`) and record the loader's reported DDR init duration (`~9 ms`).
2. **Full-training timing**: Erase or invalidate the first 4 KiB sector of the `DDRT` envelope at `bootinfo+0x10000`, reboot, and record the full-training duration (`~5.6 s`).
3. **Mailbox trace**: Boot an instrumented loader with `HdtCtrl = 0x04` and an invalidated envelope to capture the major and streaming message sequence over UART.
4. **Envelope inspection**: Read back the `0x9800` bytes at `bootinfo+0x10000` and parse the header and two `0x49a4`-byte controller records according to [10-persistence-and-selection.md](10-persistence-and-selection.md).

## Related chapters

- [06-full-initialization.md](06-full-initialization.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [09-training-results.md](09-training-results.md)
- [10-persistence-and-selection.md](10-persistence-and-selection.md)
- [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md)
