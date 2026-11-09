# Quickboot

## Scope and evidence

Quickboot reuses a previous training result instead of requesting the normal calibration sequence. The flow, offsets, and mappings here are **observed** in the loader and its message layout. Recommendations and unresolved behavior are separated at the end.

Quickboot is valid only after the complete persistent envelope passes the magic, memory-type, rank-count, data-rate, and full-envelope CRC checks described in `10-persistence-and-selection.md`. An invalid or incomplete result must lead to full training.

On the reference board, accepted quickboot took approximately **9 ms**, compared with approximately **5.6 seconds** for full training. These timings are measurements from that board, not guaranteed constants.

## Observed: prerequisites and high-level behavior

Before quickboot-specific work begins, DDR clock and DDRC configuration follow the common initialization path. The quickboot IMEM and additional DMEM images must also be available.

Quickboot does **not** apply the normal pre-training and training tables and does not request a new training sequence. Each controller is initialized sequentially using its own saved `0x49a4` result record.

The host modifies a working copy of the saved message:

- `SequenceCtrl = 1`, requesting only device initialization (`DevInit`); and
- `Quickboot = 1`, selecting result reuse.

The normal training bits are therefore not requested. The host also copies selected trained outputs into mode-register shadow entries before restoring DMEM.

## Observed: source-result to MR-shadow mapping

All offsets below are byte offsets within the `0x140`-byte message. The left columns are the **trained result sources**; the right columns are the **MR-shadow destinations** written by the host before DMEM restoration. Rank order is always **A0, A1, B0, B1**.

| Trained source | Source offsets, A0/A1/B0/B1 | MR shadow | Destination offsets, A0/A1/B0/B1 |
|---|---|---|---|
| `TrainedVREFCA` | `0x33`, `0x34`, `0x4e`, `0x4f` | MR12 | `0x6e`, `0x6f`, `0x70`, `0x71` |
| `TrainedVREFDQ` | `0x35`, `0x36`, `0x50`, `0x51` | MR14 | `0x76`, `0x77`, `0x78`, `0x79` |
| `TrainedVREFDQU` | `0xd0`, `0xd5`, `0xda`, `0xdf` | MR15 | `0x7a`, `0x7b`, `0x7c`, `0x7d` |
| `TrainedDRAMDFE` | `0xd1`, `0xd6`, `0xdb`, `0xe0` | MR24 | `0x9a`, `0x9b`, `0x9c`, `0x9d` |
| `TrainedDRAMDCA` | `0xd2`, `0xd7`, `0xdc`, `0xe1` | MR30 | `0xae`, `0xaf`, `0xb0`, `0xb1` |

The copy is byte-for-byte; the host does not recalculate the training values. `TrainedDRAMDFE` uses fields `[6:4]` and `[2:0]`, and `TrainedDRAMDCA` uses `[7:4]` and `[3:0]`.

These destinations are message-resident shadow values. They are not five immediate host writes to the DRAM. The restored quickboot firmware subsequently consumes the message and applies the saved configuration.

## Observed: DMEM and SRAM restore semantics

The saved result is restored into different runtime regions at different points in the sequence:

| Saved component | Runtime destination | Restore semantics |
|---|---|---|
| Modified message, `0x140` bytes | DMEM index `0x58000` | Copy the complete message after setting quickboot controls and MR shadows |
| `phypara`, `0x2864` bytes | DMEM index `0x58200` | Copy all 5170 halfwords sequentially, preserving the 69-segment serialized order |
| Additional quickboot DMEM image | DMEM index `0x59622` | Load the quickboot-specific image after message and `phypara` restoration |
| ACSM, `0x2000` bytes | ACSM SRAM index `0x41000` | Restore 4096 contiguous halfwords after stopping the PMU |

The start of the additional DMEM image follows directly from the indices:

`0x59622 = 0x58000 + 0x200 + 5170`

Here `0x200` is the reserved message area in halfwords, which is larger than the actual `0xa0`-halfword message. The `phypara` stream is not expanded or scattered by the host: quickboot firmware consumes its serialized DMEM representation.

The quickboot path clears the relevant remaining DMEM through index `0x60000`. In particular, the area preceding the extra image is cleared before loading it, and the rest after the image is cleared up to the DMEM limit. This clearing is part of the runtime environment; an implementation must not leave stale training data in those ranges.

PSTATE SRAM at index `0xa0000` has restoration support in the architecture, but that restoration is commented out in the observed current flow. It is **not** restored by this quickboot sequence.

ACSM restoration is deliberately separate from DMEM restoration. The loader stops the PMU and clears bit 6 of `0x20037` before restoring the contiguous ACSM contents. It does not write ACSM to DMEM `0x58200` or include it in the extra DMEM image.

## Observed: ordered quickboot flow

1. Set `SequenceCtrl = 1` and `Quickboot = 1` in the working message.
2. Populate MR12, MR14, MR15, MR24, and MR30 shadows from the trained results using the exact table above.
3. Run MemReset and wait **30 microseconds plus 2 milliseconds**.
4. Set clocks to 7, select APB-only operation, and disable ECC.
5. Load quickboot firmware into IMEM at PHY index `0x50000`.
6. Restore the complete modified message at DMEM `0x58000` and serialized `phypara` at `0x58200`.
7. Load the additional DMEM image at `0x59622`, with the surrounding clearing through `0x60000` described above.
8. Release the PMU with `MICRO_RESET = 9, 1, 0` and consume/acknowledge its mailbox messages.
9. Stop the PMU, clear `0x20037` bit 6, and restore ACSM at `0x41000`.
10. Write `0xd00e7 = 0x0600`, set clocks to 2, and set the microcontroller mux to 1.

MemReset uses PHY indices `0x20090` and `0x20060`, plus timing values at `0x3f0a2`, `0x3f042`, `0x3f043`, and `0x3f0a3` (`0xf00`, `0xf0f`, `0x0f0f`, and `0x800`, respectively). APB-only and ECC-disable operations use `0xd0036 = 0` and `0xc0086 = 1`.

The normal completion message is major message `0x07`. Mailbox waits in the reference flow have no useful timeout guarantee, so a stalled PMU can stall boot.

## Observed: what quickboot does not do

- It does not execute normal pre-training/training tables.
- It does not perform a fresh result collection.
- It does not repair an incompatible or corrupt saved result.
- It does not restore PSTATE SRAM in the current flow.
- It does not inherently rewrite the persistent envelope.

The saved message returns through the reuse flow with its trained values; the quickboot control changes belong to the runtime working copy and are not evidence of a new training run.

## Inferred: implementation safeguards

A replacement implementation should validate the envelope before touching the PHY, use a separate mutable message copy, and preserve the exact source/destination rank order. It should restore all three saved components rather than treating the MR mappings as a substitute for `phypara` or ACSM.

Adding bounded mailbox/PMU waits and a fallback to full training is prudent, but that is new policy. A fallback after arbitrary PMU corruption may require a full hardware reset; it must not be assumed to recover every halted state.

After quickboot, memory should still be validated when diagnosing instability. An intact CRC proves consistency of stored bytes, not that those coefficients remain suitable after a board, memory-device, firmware, voltage, or temperature change outside the checked metadata.

## Unknown or not established

- CRC and metadata validation cannot establish the physical suitability of old coefficients under every changed operating condition.
- The documents do not define a universal quickboot completion deadline or a guaranteed in-place recovery sequence after a PMU hang.
- The semantic details of many restored DMEM and ACSM locations remain unknown; their complete opaque preservation is nevertheless required.
- PSTATE SRAM restoration is absent from the observed flow, and its necessity for other firmware/configurations is not established.
- The reference timing does not guarantee the same latency on other boards or software stacks.
