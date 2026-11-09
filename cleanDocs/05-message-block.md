# LPDDR5 PMU message block

## Purpose and scope

The message block is the data interface between the boot loader (host) and the PHY's PMU firmware. The host supplies topology, clocking, training options, and initial DRAM mode-register values. The PMU returns status and trained values. This document specifies the LPDDR5 1D layout used on the SpacemiT K3; it is not a C structure definition and does not prescribe compiler padding.

**Observed** below means established from the K3 interface description, firmware behavior, or board captures. **Recommended** denotes policy for a new implementation, not behavior guaranteed by the reference implementation.

## Location, size, and byte order

| Property | Value |
|---|---|
| Host DMEM base index | `0x58000` |
| Message size | **320 bytes = `0x140` bytes = 160 halfwords** |
| Valid byte offsets | `0x000` through `0x13f` |
| Reserved DMEM stride | `0x200` halfwords; only the first `0xa0` halfwords are the message |
| PMU ARC address of byte 0 | `0x80000000` |
| Byte order | Little-endian; even byte offsets occupy the low byte of a halfword |

For an even byte offset, the containing host CSR index is:

```text
0x58000 + byte_offset / 2
```

Odd offsets share that same CSR index. Host MMIO addresses use `DPHY_BASE + 4 * csr_index`, even though the indexed PHY resource is 16 bits wide. Do not confuse these indices with ARC byte addresses or DDRC byte offsets.

Examples: `DRAMFreq` at byte `0x006` uses CSR `0x58003`; `SequenceCtrl` at `0x010` uses `0x58008`; `HdtCtrl` at `0x012` is the low byte of CSR `0x58009`.

The host prepares the entire block before starting the PMU and must not modify it while the PMU owns the PHY. Reserved and RFU fields are zero unless a board-specific contract explicitly says otherwise. Preserve all 320 bytes when saving results, including fields whose purpose is unknown.

A post-run dump taken at CSR `0x58000` in one instrumented reference boot was zeroed. Therefore, do not assume that the live DMEM block remains readable as an output block after the first PMU phase. The reference result-collection path supplies the authoritative saved message.

## Direction and lifetime

- **Input:** written by the host before PMU execution.
- **Output:** produced or updated by the PMU; the host must initialize its storage deterministically but must not treat an output as an input.
- **N/A or reserved:** zero on input and preserved opaque on persistence.
- `A0`, `A1`, `B0`, and `B1` mean channel A/B and rank 0/1 respectively.

The message alone is not a complete quickboot result. A complete per-controller result also contains the serialized PHY parameters and ACSM SRAM contents described in `06-full-initialization.md`.

## Core control fields

| Byte offset | Size | Dir. | Field | Behavioral role |
|---:|---:|:---:|---|---|
| `0x001` | 1 | In | `MsgMisc` | Global execution and reporting options; see below. |
| `0x002` | 2 | Out | `PmuRevision` | PMU firmware revision. |
| `0x004` | 1 | In | `Pstate` | Target P-state. Bit 7 selects the 15-P-state DMA-transfer mode; its interaction with the numeric P-state value is not fully characterized. |
| `0x005` | 1 | In | `PllBypassEn` | 0 uses the PHY PLL; 1 selects PLL bypass for the target P-state. |
| `0x006` | 2 | In | `DRAMFreq` | SDRAM data rate in MT/s, not a clock in MHz. Example: 6400 is `0x1900`. |
| `0x008` | 1 | In | `DfiFreqRatio` | **MemClk/SDRAM-WCK ratio.** `0x2` means 1:2 and `0x4` means 1:4. It is not a DFI-controller-clock ratio. |
| `0x009` | 1 | In | `BitTimeControl` | Bits [2:0] select `2^n` times the base number of data comparisons per candidate point. More comparisons increase confidence and run time. Bits [7:3] are zero. |
| `0x00a` | 2 | In | `Train2DMisc` | 2D optimization, contour reporting, iteration, seed, and VREF runtime options. |
| `0x00d` | 1 | In | `Misc` | Reset-pin ownership and read-training search options. Bit 0=0 leaves `BP_MEMRESET_L` under PHY control through training; bit 0=1 permits DFI reset control and requires the controller to hold reset deasserted throughout training. |
| `0x00e` | 1 | In | `SIFriendlyDlyOffset` | Signed eye-placement offset in bits [7:1]. Bit 0 controls treatment of an eye truncated at low voltage. |
| `0x00f` | 1 | Out | `CsTestFail` | Zero means no reported rank failure; nonzero means at least one rank failed. |
| `0x010` | 2 | In | `SequenceCtrl` | Training-stage enable mask; see below. |
| `0x012` | 1 | In | `HdtCtrl` | Streaming verbosity threshold. |
| `0x014` | 2 | Out | `InternalStatus` | RFU; no dependable meaning is assigned. |
| `0x016` | 1 | In | `DFIMRLMargin` | Margin in DFI clocks added to the smallest passing maximum-read-latency result. It must cover expected positive `tDQSCK` drift; the interface recommends at least 1. |
| `0x017` | 1 | In | `TX2D_Delay_Weight` | TX 2D delay weight, bits [4:0]. |
| `0x018` | 1 | In | `TX2D_Voltage_Weight` | TX 2D voltage weight, bits [4:0]. |
| `0x019` | 1 | In | `Quickboot` | Nonzero selects quickboot operation in the saved-message flow. |
| `0x01b` | 1 | In | `CATrainOpt` | CA VREF and LPDDR5 CS-training enables, clock choice, and CA scan increment. |
| `0x01c` | 1 | In | `X8Mode` | Per-channel/rank organization bits in A0,A1,B0,B1 order: 0=x16, 1=two x8 devices. Documented valid LP5 values are `0x0`, `0xa`, and `0xf`. |
| `0x01d` | 1 | In | `RX2D_TrainOpt` | Bit 0 suppresses the complementary `RxClkCDly` scan when set; zero performs the default scan. Other bits are zero. |
| `0x020` | 1 | In | `RX2D_Delay_Weight` | RX 2D delay weight, bits [4:0]. |
| `0x021` | 1 | In | `RX2D_Voltage_Weight` | RX 2D voltage weight, bits [4:0]. |
| `0x024` | 1 | In | `EnabledDQsChA` | **Number** of enabled DQ bits in channel A, not a bit mask. |
| `0x025` | 1 | In | `CsPresentChA` | Channel A populated chip selects: `0x1`=CS0, `0x3`=CS0+CS1. |
| `0x032` | 1 | In | `CATerminatingRankChA` | Channel A CA terminating rank: 0 or 1. |
| `0x03f` | 1 | In | `EnabledDQsChB` | Number of enabled DQ bits in channel B. |
| `0x040` | 1 | In | `CsPresentChB` | Channel B populated chip selects: `0x0`=none, `0x1`=CS0, `0x3`=CS0+CS1. |
| `0x04d` | 1 | In | `CATerminatingRankChB` | Channel B CA terminating rank: 0 or 1. |

### `MsgMisc` bits

| Bit | Meaning |
|---:|---|
| 0 | Pulse the digital test output at major training-stage boundaries. This does not itself select a training algorithm. |
| 1 | Simulation-only shortened reset; must be 0 on silicon. |
| 2 | Simulation-only reduced training; must be 0 on silicon. |
| 3 | Disable the boot-frequency clock during DRAM initialization. Normal operation leaves this 0. |
| 4 | Suppress all streaming messages, including streamed assertions, regardless of `HdtCtrl`. Major stage/completion/error messages remain subject to their own filtering. |
| 5 | RFU; zero. |
| 6 | Average write-leveling and applicable PHY DCA results across ranks/stages rather than retaining independent values. |
| 7 | RFU; zero. |

A persisted reference-board block had `MsgMisc=0x01` and `HdtCtrl=0x04`; these are observations, not universal required defaults.

### `SequenceCtrl`

Bit 0 is required for normal device/PHY initialization. Reserved bits 5, 11, and 13–15 are zero.

| Bit | Stage |
|---:|---|
| 0 | Device and PHY initialization (`DevInit`) |
| 1 | Write leveling (`WrLvl`) |
| 2 | Read gate / receive enable (`RxEn`) |
| 3 | Read DQS training (`RdDQS`) |
| 4 | Write DQ training (`WrDq`) |
| 6 | DRAM duty-cycle adjustment |
| 7 | PHY read duty-cycle adjustment |
| 8 | PHY write duty-cycle adjustment |
| 9 | Maximum read latency (`MxRdLat`) |
| 10 | DRAM transmit DFE |
| 12 | Low-power CA training (`LPCA`) |

For quickboot, the reference saved-message flow sets `SequenceCtrl=1` and `Quickboot=1`, requesting initialization while reusing prior training results.

### `HdtCtrl`

The PMU emits a streamed message when its associated level is **less than** `HdtCtrl`; increasing the value reduces output. Named thresholds are `0x05` (detailed eye-delay debug), `0x0a` (coarse/rank debug), `0xc8` (stage completion), `0xc9` (assertions), and `0xff` (firmware completion only). `MsgMisc[4]` is an independent hard suppression of streaming. See `08-mailbox-and-streaming.md` for transport semantics.

## Advanced input groups

These fields tune the algorithm but do not replace the board's validated parameter set.

| Range | Role |
|---|---|
| `0x0ce` | `Disable2D`: nonzero disables 2D training. |
| `0x0cf` | `VrefSamples`: number of VREFs to scan; zero selects the default scan. |
| `0x0e4`–`0x0e7` | Custom read/write patterns, inversion, and LDFF/DBI pattern behavior. All-zero pattern controls select defaults. |
| `0x0e8`–`0x0f3` | ACSM/DMA start addresses for frequency-change operation. |
| `0x0f4`–`0x0ff` | Optional low-speed AC transmit slew and impedance settings, enabled by `EnCustomSettings`. |
| `0x100` | `VrefInc`: 2D VREF increment; the interface specifies 1. |
| `0x101` | `UpperLowerByte`: per-A0/A1/B0/B1 partner-byte swap bits. |
| `0x102` | `DisableTrainingLoop`: independently suppress retraining loops for write leveling, read gate/read DQ, and write training. This is an optimization/skip control, not an acceptance criterion. |
| `0x103`–`0x104` | Alternate and main read-latency inputs. |
| `0x105` | `CSBACKOFF`: values 1, 2, 3 select -0.125, -0.25, -0.375 tCK; default is -0.5 tCK. |
| `0x106` | `WrLvlTrainOpt[0]`: skip coarse WCK-to-CK leveling. |
| `0x108`–`0x10f` | Outputs describing DCCM CSR/MR DMA locations and lengths. |
| `0x110` | Signed `MRLCalcAdj`, added to intermediate maximum-read-latency calculations. |
| `0x111` | `PPT2OffsetMargin`; zero disables it, nonzero extends left/right eye offsets to avoid underflow/overflow. |
| `0x118`–`0x126` | RX/TX VREF start, end, and step controls for pattern/PRBS and DFE modes. `Train2DMisc[13]` changes the PRBS RX controls from absolute bounds to counts around the SI-friendly result. |
| `0x127`–`0x12e` | Signed RX delay scan shifts for rank 0/1 and byte 0–3. |
| `0x130`–`0x13d` | Seven 16-bit quickboot PLL outputs: four `QBPllUPllProg` values followed by `QBPllCtrl1`, `QBPllCtrl4`, and `QBPllCtrl5`. |

## DRAM mode-register inputs

Every entry is one byte and the four entries for each MR are in **A0, A1, B0, B1** order. The host supplies these values; the PMU consumes them during initialization/training.

| MR | A0 offset | Four-byte range |
|---:|---:|---|
| 1 | `0x05a` | `0x05a`–`0x05d` |
| 2 | `0x05e` | `0x05e`–`0x061` |
| 3 | `0x062` | `0x062`–`0x065` |
| 10 | `0x066` | `0x066`–`0x069` |
| 11 | `0x06a` | `0x06a`–`0x06d` |
| 12 | `0x06e` | `0x06e`–`0x071` |
| 13 | `0x072` | `0x072`–`0x075` |
| 14 | `0x076` | `0x076`–`0x079` |
| 15 | `0x07a` | `0x07a`–`0x07d` |
| 16 | `0x07e` | `0x07e`–`0x081` |
| 17 | `0x082` | `0x082`–`0x085` |
| 18 | `0x086` | `0x086`–`0x089` |
| 19 | `0x08a` | `0x08a`–`0x08d` |
| 20 | `0x08e` | `0x08e`–`0x091` |
| 21 | `0x092` | `0x092`–`0x095` |
| 22 | `0x096` | `0x096`–`0x099` |
| 24 | `0x09a` | `0x09a`–`0x09d` |
| 25 | `0x09e` | `0x09e`–`0x0a1` |
| 26 | `0x0a2` | `0x0a2`–`0x0a5` |
| 27 | `0x0a6` | `0x0a6`–`0x0a9` |
| 28 | `0x0aa` | `0x0aa`–`0x0ad` |
| 30 | `0x0ae` | `0x0ae`–`0x0b1` |
| 31–34 | `0x0b2` | Reserved blocks through `0x0c1`; zero |
| 37 | `0x0c2` | `0x0c2`–`0x0c5` |
| 40 | `0x0c6` | `0x0c6`–`0x0c9` |
| 41 | `0x0ca` | `0x0ca`–`0x0cd` |

The MR bit meanings and legal values come from the selected DRAM's LPDDR5 specification and board configuration; they are not inferred from the message layout.

## Trained outputs

### Delay differences, VREFs, and margins

- Channel A signed critical delay differences (CDDs): `0x026`–`0x031`.
- Channel B signed CDDs: `0x041`–`0x04c`.
- The twelve CDDs in each channel are ordered `RR_1_0`, `RR_0_1`, `RW_1_1`, `RW_1_0`, `RW_0_1`, `RW_0_0`, `WR_1_1`, `WR_1_0`, `WR_0_1`, `WR_0_0`, `WW_1_0`, `WW_0_1`.
- `TrainedVREFCA`: A0/A1/B0/B1 at `0x033`, `0x034`, `0x04e`, `0x04f`.
- `TrainedVREFDQ`: A0/A1/B0/B1 at `0x035`, `0x036`, `0x050`, `0x051`.

Each rank has four one-byte margins in the order below:

| Margin | Units | A0 | A1 | B0 | B1 |
|---|---|---:|---:|---:|---:|
| `RxClkDly_Margin` | PHY DLL/tap steps | `0x037` | `0x03b` | `0x052` | `0x056` |
| `VrefDac_Margin` | PHY DAC steps | `0x038` | `0x03c` | `0x053` | `0x057` |
| `TxDqDly_Margin` | PHY DLL/tap steps | `0x039` | `0x03d` | `0x054` | `0x058` |
| `DeviceVref_Margin` | DRAM DAC steps | `0x03a` | `0x03e` | `0x055` | `0x059` |

Each value is the minimum, over all eyes in that timing group, of the distance from the selected center to the nearest failing region. It is not an eye width and values in different units are not directly interchangeable.

### Additional LPDDR5 results

| Rank | `TrainedVREFDQU` | `TrainedDRAMDFE` | `TrainedDRAMDCA` |
|---|---:|---:|---:|
| A0 | `0x0d0` | `0x0d1` | `0x0d2` |
| A1 | `0x0d5` | `0x0d6` | `0x0d7` |
| B0 | `0x0da` | `0x0db` | `0x0dc` |
| B1 | `0x0df` | `0x0e0` | `0x0e1` |

`TrainedDRAMDFE` carries upper/lower fields in bits [6:4] and [2:0]. `TrainedDRAMDCA` carries them in [7:4] and [3:0]. Intervening bytes are reserved.

### Mapping trained results back to MR inputs for quickboot

Before restoring saved DMEM, the reference quickboot flow copies each trained result to the same channel/rank position in the corresponding MR input array:

| Trained output | Destination MR input | Destination A0 offset |
|---|---|---:|
| `TrainedVREFCA` | MR12 | `0x06e` |
| `TrainedVREFDQ` | MR14 | `0x076` |
| `TrainedVREFDQU` | MR15 | `0x07a` |
| `TrainedDRAMDFE` | MR24 | `0x09a` |
| `TrainedDRAMDCA` | MR30 | `0x0ae` |

The A0/A1/B0/B1 ordering and packed bit positions are preserved. This is an explicit result-to-input transformation for reuse, not an alias in the original 320-byte block.

## New-implementation requirements

1. Construct the block using byte offsets and explicit little-endian halfword accesses; do not rely on host structure packing.
2. Validate topology counts, `CsPresent`, `X8Mode`, data rate, and MR values before releasing the PMU.
3. Keep input and output ownership separate and do not write the block during training.
4. Persist all 320 bytes and treat unknown fields as opaque.
5. Treat `CsTestFail`, completion/error messages, margins, and post-training memory tests as separate evidence; no universal numeric margin threshold is known.
