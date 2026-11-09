# Persistence and boot-time selection

## Scope and evidence

The persistent envelope stores training results for two controllers. Its binary layout, CRC range, reference-board location, and selection behavior are **observed**. The update procedure at the end is an **inferred safe design** derived from those constraints; it is not a claim that the reference loader already implements atomic updates.

All offsets and sizes are in bytes. Multi-byte fields use the loader's native little-endian representation.

## Observed: per-controller record

Each controller record is exactly **`0x49a4` bytes**:

| Record offset | Size | Contents |
|---:|---:|---|
| `0x0000` | `0x0140` | Complete message block (`0xa0` halfwords) |
| `0x0140` | `0x2864` | Serialized `phypara` (5170 halfwords) |
| `0x29a4` | `0x2000` | ACSM (4096 contiguous halfwords) |
| End | `0x49a4` | Next record or end of controller data |

The ACSM offset is therefore exactly **`0x140 + 0x2864 = 0x29a4`**.

`phypara` uses the fixed **69-segment, 5170-halfword** serialization order documented in [09-training-results.md](09-training-results.md) and fully enumerated in [appendix-phypara-order.md](appendix-phypara-order.md). It is not a linear PHY dump. ACSM is contiguous. Preserve every message byte and every reserved/unknown part of the record.

## Observed: complete `0x9800` envelope

The reserved persistent object is **`0x9800` bytes (38,912 bytes)**, even though the defined header and two result records occupy only `0x93c8` bytes.

| Envelope offset | Field | Size / value |
|---:|---|---|
| `0x0000` | `magic` | `u32`, `0x54524444` (`DDRT`) |
| `0x0004` | `crc32` | `u32`, CRC-32 of `[0x08, 0x9800)` |
| `0x0008` | `chipid` | `u64` |
| `0x0010` | `mac_addr` | `u64`, may be zero |
| `0x0018` | `version` | `u32`; reference sample is zero |
| `0x001c` | `type` | `u32`; 0 = LPDDR4X, 1 = LPDDR5 |
| `0x0020` | `cs_num` | `u32`, number of ranks |
| `0x0024` | `data_rate` | `u32`, MT/s |
| `0x0028` | reserved header | through `0x007f` (88 bytes) |
| `0x0080` | DDR0 result | `0x49a4` bytes |
| `0x4a24` | DDR1 result | `0x49a4` bytes |
| `0x93c8` | remaining reserved result area | through `0x97ff` |
| `0x9800` | end of envelope | total reserved size |

The populated size calculation is:

`0x80 + 2 * 0x49a4 = 0x93c8`

DDR1 begins at `0x80 + 0x49a4 = 0x4a24`. The bytes from `0x93c8` through `0x97ff` are still part of the persistent object and still participate in its CRC.

### CRC coverage

The stored checksum is the CRC-32 used by the reference implementation, confirmed with `zlib.crc32`, over the **full reserved envelope except the magic and CRC fields themselves**:

**CRC input = bytes `[0x08, 0x9800)`**

Thus the input starts with `chipid` and includes the defined header, both controller records, and all trailing reserved bytes. Its length is `0x97f8` bytes. It does **not** stop at `0x93c8`, and it does not include `magic` or the stored CRC field. When modifying any covered byte, including padding, recompute the checksum over exactly this range.

## Observed: storage and invalidation

On the reference board, the envelope resides in SPI flash in the `bootinfo` partition at partition-relative offset `0x10000`. The reference flash sector size is 4 KiB. This location is board-specific rather than an intrinsic property of the format.

A storage implementation must accommodate the entire `0x9800` object and the flash erase geometry. Destroying `magic` invalidates the result and forces full training; erasing its first sector is sufficient for deliberate invalidation. A failed or interrupted training run must not silently destroy the last usable result.

## Observed: selection rules

The loader chooses quickboot only when the envelope passes all of these checks:

1. `magic` equals `0x54524444`;
2. `type` matches the detected memory generation;
3. `cs_num` matches the detected rank count;
4. `data_rate` matches the configured rate; and
5. the stored CRC matches the CRC computed over `[0x08, 0x9800)`.

Any failure selects full training. The loader structure performs the inexpensive metadata comparisons before calculating CRC; that ordering is an optimization, not a relaxation of any condition.

Hardware experiments on the reference board confirmed the following isolated cases:

| Envelope variant | Result |
|---|---|
| Invalid CRC, other fields correct | Full training, 5630 ms |
| `cs_num = 1` instead of 2, valid recomputed CRC | Full training, 5630 ms |
| `data_rate = 4800` instead of 6400, valid recomputed CRC | Full training, 5635 ms |
| Original valid envelope | Quickboot, 9 ms |

These measurements establish that each tested mismatch independently forces training. They do not experimentally establish the relative check order; that order comes from the loader structure. Erasing the sector had previously been observed to force training through invalid magic, but invalid magic was not part of the controlled, CRC-recomputed variant set.

`version` exists for format evolution, but version checking is not among the five observed acceptance conditions above. `chipid` and `mac_addr` are stored, but the cited selection evidence does not establish that they are compared during boot.

## Inferred: safe update policy

A robust implementation should treat the saved envelope as a write-on-success snapshot, not as a work buffer.

1. **Train into RAM.** Collect complete DDR0 and DDR1 records without modifying flash. Verify exact sizes, completion status, and `CsTestFail`.
2. **Validate memory.** Exercise DRAM before replacing a known-good result. Successful PMU messaging alone is not sufficient evidence of stability.
3. **Construct the entire envelope in RAM.** Set metadata from the actual detected configuration, populate both independent controller records, and initialize every reserved/padding byte deterministically. Do not leave stale bytes in the CRC-covered tail.
4. **Finish with the CRC.** Compute CRC-32 over `[0x08, 0x9800)` only after all other bytes are final, then store it at `0x0004`.
5. **Commit deliberately.** Use an A/B envelope scheme where possible, keeping the previous valid copy until the new full object has been written and read back successfully. If only one slot exists, invalidate the old magic before erase/write and accept that interruption will cause retraining, never quickboot from a partial object.
6. **Verify the committed object.** Read back all `0x9800` bytes, check metadata, recompute the CRC over the full range, and only then consider the new snapshot usable.
7. **Fall back conservatively.** Any read, erase, write, verification, compatibility, or integrity failure must select full training rather than attempting partial restoration.

The reference full-training path did not automatically rewrite the envelope; writing was a separate later-loader action. A replacement implementation must therefore schedule persistence explicitly after successful training and validation.

For format changes, increment `version`, reject unsupported layouts, and retrain. Never reinterpret an existing field with different semantics. Because current observed selection does not check `version`, adding a version check is a necessary implementation hardening step when multiple layouts are possible, not a description of the reference behavior.

## Unknown or not established

- The required values and future semantics of `version`, `chipid`, `mac_addr`, and all reserved bytes are not fully specified beyond the captured sample and layout.
- The reference board's single-slot update is not proven power-fail atomic.
- The evidence does not establish a board-independent storage location or erase size.
- Equality of the two controller records in the captured sample is incidental, not an acceptance rule.
- No evidence supports selecting quickboot from only one valid controller record when the other is absent or corrupt: the envelope CRC covers both.
