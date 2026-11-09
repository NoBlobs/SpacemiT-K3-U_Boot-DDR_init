# Full LPDDR5 initialization

## Scope and execution model

This document specifies the full-training path for one SpacemiT K3 LPDDR5 controller and the ordering across the two controllers. Quickboot is a different path and is covered only where it affects result ownership.

There are two execution agents:

- The **host/SPL** discovers the memory, configures clocks and DDRC, loads the PHY/PMU images and tables, services the mailbox, collects results, activates the controller, and tests memory.
- The **PMU**, an ARCv2/EM microcontroller inside each DDR PHY, executes device initialization and training. The host CPU does not perform the per-lane training sweeps.

The two DDRC instances are initialized **sequentially**, each with its own PHY, PMU execution, mailbox stream, and saved result. A controller-local failure must not be hidden by continuing as though both controllers were usable.

**Observed reference behavior** and **recommended new-implementation policy** are separated below. The reference contains unbounded waits and inconsistent PMU-error propagation; those are not desirable interface requirements.

## Bases and access conventions

For each controller, obtain `DDRC_BASE` from the hardware description rather than assuming a single fixed address:

```text
CFG_BASE  = DDRC_BASE + 0x600000
DPHY_BASE = DDRC_BASE + 0x800000
```

PHY resources use CSR indices: host byte address = `DPHY_BASE + 4 * index`. DDRC and CFG offsets in this document are byte offsets from their bases. PHY/mailbox values are 16-bit resources accessed through the indexed host window; preserve neighboring halfwords when only one is intended.

Important windows are:

| Resource | PHY CSR index | Role |
|---|---:|---|
| ACSM SRAM | `0x41000` | 4096 halfwords collected after training |
| PMU IMEM | `0x50000` | Training firmware loaded as halfwords |
| DMEM/message | `0x58000` | 320-byte message block |
| Serialized PHY parameters | `0x58200` | Quickboot restore format, 5170 halfwords |
| DMEM clear start | `0x584d2` | Clear through `0x5ffff` in full-training preparation |

Do not access the PHY/PMU MMIO from an operating system after boot: on the reference board this stalled the SoC. Debug collection belongs in the loader while the relevant clocks and PMU are active, or on a suitable debug interface.

## Ownership and ordering at a glance

| Stage | Owner | Required result before advancing |
|---:|---|---|
| 1 | Host | Part, ranks, organization, density, rate, electrical parameters, and firmware selected |
| 2 | Host | DPLL locked, clock switch complete, DCLK enabled |
| 3 | Host | DDRC timings, DFI, refresh, ZQ, ranks, address map, and configuration-mode state programmed |
| 4 | Host | IMEM/DMEM, PHY tables, overrides, message block, and DMEM clear complete |
| 5 | PMU, serviced by host | PMU completion `0x07` received, or a declared error returned |
| 6 | Host | PMU stopped, host ownership restored, post-run training table applied |
| 7 | Host | Message, 5170 PHY halfwords, and 4096 ACSM halfwords collected |
| 8 | Host | DFI operational, DDRC in normal state, self-refresh off, ports enabled |
| 9 | Host | Non-destructive-after-restore memory smoke test passed |

The host must not access PHY CSRs while the PMU owns the PHY, except for the defined mailbox interface. Conversely, the host must own the PHY before table application and result collection.

## Stage 1 — discovery and parameter selection

1. Read the DRAM part identifier from the board identification source, with the hardware description as fallback.
2. Resolve it to memory type, rank count, x8/x16 organization, density, and supported data rate.
3. Reject unsupported memory types and rates. The observed LP5 DDRC path accepts **4800 and 6400 MT/s**; other rates must not silently select an approximation.
4. Select the electrical configuration: PHY write drive, PHY RX ODT, DRAM DQ/CA ODT, non-target/SOC ODT, PDDS, and whether 2D training is enabled.
5. Select topology/rate-specific PHY overrides and prepare the normal-training IMEM and initial DMEM content.
6. Allocate a per-controller result area. The reference result size is `0x49a4` bytes.

If a valid persisted result matches the current memory type, ranks, rate, integrity check, and format version, a loader may choose quickboot instead. That decision occurs before this full-training sequence; an invalid or mismatched result must not be partially reused.

## Stage 2 — clock initialization

1. Program the CFG DPLL for the target rate. The observed configuration uses a 3000 MHz DPLL path for 4800 MT/s and a 3200 MHz path for 6400 MT/s.
2. Poll `CFG+0x1c` bit 0 until lock.
3. Clear the prior divider selection in `CFG+0x18[21:16]` and select the observed source encoding: `7 << 19` for 4800 or `2 << 19` for 6400.
4. Start the frequency change with `CFG+0x18[25]` and wait for that bit to clear.
5. Set `CFG+0x18[0]`.
6. Before PHY pre-training, release DCLK using `CFG+0x18[22]`. The reference also sets bit 1 in this phase.

**Recommended:** bound every lock/change wait and fail with a clock-specific diagnostic. Do not start the PMU after a clock timeout.

## Stage 3 — DDRC configuration

Configure the DDRC while the host owns the PHY:

1. Program the master/DFI configuration and DFI update timings.
2. Apply the rate- and organization-specific DRAM timing set.
3. Program refresh, derating, ZQ timing, rank turnaround timings, and rank presence.
4. Construct the bank/row/column/rank address map and system address regions from density and topology.
5. Place the controller in its training-preparation state using `opctrl1`, `inittmg0`, `pwrctl`, and `rfshctl0`.
6. Configure ZQ controls and perform the required quasi-static register transactions with `swctl`/`swstat`.
7. Set `dfimisc=0x00010014` before PMU execution, again using the DDRC configuration transaction mechanism.

The relevant DDRC offsets are byte offsets. In particular, `swctl=0x10c80`, `swstat=0x10c84`, `dfimisc=0x10510`, `dfistat=0x10514`, `pwrctl=0x10180`, and controller status `stat=0x10014`.

Address-map invariants observed in the reference are:

- Four system address regions begin at 2 GiB and are 256 MiB aligned.
- In dual-rank configurations, the chip-select address derives from the memory MSB; in single-rank configurations CS mapping is disabled (`ADDRMAP1=0x3f`).
- Non-power-of-two density requires the non-binary row-mapping extension rather than simply truncating capacity.

## Stage 4 — PHY pre-training preparation

This stage establishes state; it does not itself measure training eyes.

1. Load normal-training firmware into IMEM at `0x50000`, in halfwords, using the actual decompressed/image length.
2. Load initial DMEM content when supplied separately.
3. Apply the base PHY pre-training tables. A table may be a linear halfword sequence or a list of discrete CSR/value pairs.
4. Apply overrides with this precedence, lowest to highest:
   1. base table value;
   2. topology/rate override;
   3. board electrical/I/O override.
5. A topology override value `0xdeadbeef` means **skip the write**, not write that literal value.
6. Install the complete 320-byte message block at `0x58000`, including frequency, topology, MR inputs, and `SequenceCtrl`.
7. Clear DMEM indices `0x584d2` through `0x5ffff`.

For a host 32-bit store covering two 16-bit CSR entries, both halfwords must be intentional. Use a read-modify-write or a paired write when only one halfword is known; do not accidentally zero its neighbor.

Immediately before releasing the PMU, the reference sets `pwrctl` bit 11.

## Stage 5 — PMU execution and mailbox servicing

1. Set `MICRO_CONT_MUX_SEL` (`0xd0000`) to 1, giving the PMU ownership.
2. Write `MICRO_RESET` (`0xd0099`) in the exact order **9, 1, 0**. Zero releases execution.
3. Continuously consume and acknowledge mailbox items until major completion `0x07`. Streaming messages can substantially outnumber major messages and must not block the producer.
4. On major `0xff`, follow the error semantics in `08-mailbox-and-streaming.md`; do not interpret a PLL-status value as a generic success/failure return.

On normal completion, the reference acknowledges `0x07`, returns success from its mailbox routine, writes `MICRO_RESET=1`, performs a short busy wait, and sets the mux to 0.

**Recommended:** use bounded mailbox waits, a single explicit completion/error result, and the same orderly PMU-stop/host-mux restoration on every exit where hardware permits. Do not proceed to collection or activation after PMU error or timeout.

## Stage 6 — post-run training table

After the first mailbox routine has completed and host ownership has been restored, the reference applies a second, rate/topology-specific PHY table. The electrical I/O override list is not reapplied in this phase.

The table's observed terminal state is:

- `UCCLK_HCLK_ENABLES` (`0xc0080`) = 2;
- CSR `0xd0003` = 0;
- `MICRO_CONT_MUX_SEL` = 1.

There is **no second explicit host mailbox-drain call** between this table and result collection. A new implementation must preserve the observed hardware sequencing unless a revised PMU/host protocol is designed and validated as a whole. It must not invent a second training completion event.

## Stage 7 — result collection

The reference collection sequence is:

1. Set bit 6 of PHY CSR `0x20037` for ACSM capture.
2. Select host access (mux 0) and clocks value 3.
3. Read exactly `0x140` bytes of message data beginning at `0x58000`.
4. Serialize exactly **5170 halfwords** from the PHY parameter groups.
5. Read exactly **4096 halfwords** from ACSM SRAM at `0x41000`.
6. Reduce the clock-enable value to 1.

The 5170 values are **not a linear CSR dump**. They comprise 69 ordered scattered/sequential segments. Their serialization order is part of the quickboot interface and must be treated as a versioned format.

Per-controller saved layout:

| Saved offset | Component | Size |
|---:|---|---:|
| `0x0000` | Complete message block | `0x140` bytes |
| `0x0140` | Serialized PHY parameters | 5170 halfwords = `0x2864` bytes |
| `0x29a4` | ACSM SRAM | 4096 halfwords = `0x2000` bytes |
| Total |  | **`0x49a4` bytes** |

The reference stores two such results consecutively. Persist a result only after all components and the configuration identity have been validated. Do not publish a partially collected result as quickboot-valid.

## Stage 8 — controller activation

1. Using `swctl`/`swstat` transactions, change `dfimisc` to `0x00010034` and wait for `dfistat` bit 0.
2. Using another transaction, change `dfimisc` to `0x00010015`.
3. Clear `pwrctl` and wait for `stat[1:0] == 01`.
4. Restore operational ZQ and refresh settings (`zqctl0=0`, `zqctl2=0`, `rfshctl0=0` in the reference), with the required configuration transactions.
5. Set `pwrctl=0x10` to disable self-refresh, restore the operational DFI update timing, and clear `opctrl1`.
6. Enable all five host ports by writing 1 to DDRC byte offsets `0x20090`, `0x21090`, `0x22090`, `0x23090`, and `0x24090`.

Only after these steps is the DRAM array available for normal CPU accesses.

## Stage 9 — memory smoke test

The reference smoke test operates on `0x4000`-byte blocks with address-derived patterns and their complements. Required behavior is:

1. Save the original block.
2. Write the first pattern and verify every location.
3. Write the complement and verify every location.
4. Restore the original contents even when verification fails.
5. Report the failing address/pattern and do not declare the controller operational.

This host-side test is distinct from PMU training. Passing it does not prove adequate training margin, and PMU completion alone does not prove end-to-end addressing works.

## Error and robustness policy for a new implementation

The following are recommendations, not claims about the reference:

- Return typed errors for unsupported configuration, clock timeout, DDRC timeout, mailbox timeout/protocol violation, explicit PMU `0xff`, `CsTestFail`, collection failure, and memory-test failure.
- Propagate errors through both normal training and quickboot. The reference's mailbox routine returns PLL status after `0xff`, while one caller compares that value to `0xff`; quickboot ignores the return. This inconsistency is observed behavior, not a contract to reproduce.
- Add timeouts to all hardware polls, preserve the last major/streaming diagnostic where practical, and leave ports disabled on failure.
- Use per-controller state and result validity; never let one controller's success validate the other's result.

For scale only, observed reference-board full-training times were about **5.6–5.8 seconds**, versus about **9 ms** for quickboot. These are measurements, not timeout limits.
