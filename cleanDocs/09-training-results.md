# Training results

## Scope and evidence

This document describes the result produced by each LPDDR5 PMU training run and the exact serialized representation consumed by quickboot. Byte offsets in the message are relative to the start of its `0x140`-byte structure. PHY indices denote 16-bit CSR/DMEM locations, not byte addresses.

The layouts and collection order below are **observed** in the loader, firmware interface definitions, and captured board data. Statements about robustness or implementation policy are explicitly marked as inferred recommendations; unresolved meanings are listed separately.

## Observed: result containers

A complete training result has three components. The message preserves both configuration and results, `phypara` preserves selected PHY state in a defined serialization order, and ACSM preserves a contiguous SRAM region.

| Component | Elements | Bytes | Runtime source |
|---|---:|---:|---|
| Message block | `0xa0` halfwords | `0x140` | DMEM index `0x58000` |
| `phypara` | 5170 halfwords | `0x2864` | 69 ordered PHY segments |
| ACSM | 4096 halfwords | `0x2000` | contiguous SRAM at index `0x41000` |

The complete message must be retained, including fields whose meaning is not yet known. Saving only the documented outputs is insufficient because quickboot restores the whole structure.

The host-visible message index is `0x58000`. For a 16-bit-aligned message field:

`DMEM index = 0x58000 + byte_offset / 2`

Eight-bit fields share a little-endian halfword. For example, `DRAMFreq` at byte `0x06` is at index `0x58003`, and `SequenceCtrl` at byte `0x10` is at index `0x58008`. The message occupies `0xa0` halfwords within a reserved `0x200`-halfword area.

## Observed: message outputs

### Status and general trained values

| Result | Byte offset(s) | Representation |
|---|---|---|
| `PmuRevision` | `0x02` | 16-bit firmware revision written by the PMU |
| `CsTestFail` | `0x0f` | zero means no rank failure; nonzero indicates a failure |
| CDD, channel A | `0x26`–`0x31` | 12 signed bytes |
| CDD, channel B | `0x41`–`0x4c` | 12 signed bytes |
| `TrainedVREFCA` | A0 `0x33`, A1 `0x34`, B0 `0x4e`, B1 `0x4f` | one byte per rank |
| `TrainedVREFDQ` | A0 `0x35`, A1 `0x36`, B0 `0x50`, B1 `0x51` | one byte per rank |

`InternalStatus` at `0x14` is a 16-bit output field documented as RFU; no additional meaning is established here.

### Margins

Each rank has four one-byte margin results, in this order: `RxClkDly`, `VrefDac`, `TxDqDly`, and `DeviceVref`.

| Rank | Byte range |
|---|---|
| A0 | `0x37`–`0x3a` |
| A1 | `0x3b`–`0x3e` |
| B0 | `0x52`–`0x55` |
| B1 | `0x56`–`0x59` |

Each value is the smallest distance from the trained center to the failing region in that dimension. A small value therefore indicates little margin, but these documents do not establish a universal pass/fail threshold.

### Additional LPDDR5 results

| Rank | `TrainedVREFDQU` | `TrainedDRAMDFE` | `TrainedDRAMDCA` |
|---|---:|---:|---:|
| A0 | `0xd0` | `0xd1` | `0xd2` |
| A1 | `0xd5` | `0xd6` | `0xd7` |
| B0 | `0xda` | `0xdb` | `0xdc` |
| B1 | `0xdf` | `0xe0` | `0xe1` |

`TrainedDRAMDFE` uses bit fields `[6:4]` and `[2:0]`; `TrainedDRAMDCA` uses `[7:4]` and `[3:0]`. Quickboot copies these values, and the CA/DQ VREF results, into the message's mode-register shadow entries.

The quickboot-related PLL outputs are `QBPllUPllProg0..3` at `0x130`, `0x132`, `0x134`, and `0x136`, followed by `QBPllCtrl1`, `QBPllCtrl4`, and `QBPllCtrl5` at `0x138`, `0x13a`, and `0x13c`.

## Observed: exact `phypara` serialization

`phypara` is **not** a linear CSR dump. It consists of exactly **69 segments and 5170 halfwords**. A segment is either:

- **scattered:** read each explicitly listed offset relative to the segment base, in list order; or
- **sequential:** read `count` consecutive halfwords beginning at the base, with stride one.

Values are appended to the output in segment order. Duplicate bases and repeated CSRs are intentional: position in the serialized stream is significant. Segments must not be sorted, merged, deduplicated, or removed. Segment 48 is the largest contiguous group, reading 1536 halfwords from base `0x90000`; segments 54 and 55 each read 96 sequential halfwords from base `0x20000`.

The authoritative complete list of bases, counts, and scattered offsets is [appendix-phypara-order.md](appendix-phypara-order.md). The count column totals 5170. Collection and restoration must use the same order.

During quickboot, the saved `phypara` byte stream is copied sequentially to DMEM index `0x58200`. The PMU consumes that serialized representation; the host does not individually scatter the values back to PHY CSRs in this path.

## Observed: collection sequence

After the training table completes, the loader:

1. sets bit 6 of PHY CSR `0x20037`;
2. selects host access and the required clocks;
3. reads all `0x140` message bytes from DMEM `0x58000`;
4. serializes the 5170 `phypara` halfwords using the 69-segment table;
5. reads 4096 contiguous ACSM halfwords from `0x41000`; and
6. reduces the clocks after collection.

Collection is per controller. DDR0 and DDR1 each produce an independent `0x49a4`-byte result record, even when their bytes happen to match.

The PMU publishes major message `0x07` on normal completion. `CsTestFail` and a completed mailbox exchange are useful checks, but persistence policy should additionally validate the resulting memory before replacing a known-good saved result.

## Observed: reference-board sample

The captured LPDDR5 envelope from the two-rank, 6400 MT/s reference board contains the following per-controller values:

| Property | DDR0 | DDR1 |
|---|---|---|
| `MsgMisc` | `0x01` | `0x01` |
| `HdtCtrl` | `0x04` | `0x04` |
| Nonzero `phypara` entries | 3926 of 5170 | 3926 of 5170 |
| Nonzero ACSM entries | 1845 of 4096 | 1845 of 4096 |

The first twelve `phypara` values in each record are:

`2867, 0, 156, 0, 1, 2, 3, 4, 5, 6, 7, 2879`

The two captured records are byte-identical. This is a sample observation, **not** a format requirement or a reason to alias the controllers' storage.

## Inferred: practical sanity checks

A result with almost all-zero `phypara` or ACSM data is suspicious when compared with the reference sample, but nonzero counts alone do not prove that training succeeded. Sensible validation includes completion status, `CsTestFail`, structural sizes, and a real memory test. Values and margins can legitimately vary across boards, memory parts, runs, voltage, and temperature.

## Unknown or not established

- The detailed meaning of many message fields and PHY CSRs remains unknown; unknown bytes must still be preserved.
- No universal numerical acceptance thresholds are established for the margin fields.
- The captured equality of DDR0 and DDR1 does not prove that all boards or training runs produce identical records.
- A successful training protocol exchange does not, by itself, prove long-term memory stability.
