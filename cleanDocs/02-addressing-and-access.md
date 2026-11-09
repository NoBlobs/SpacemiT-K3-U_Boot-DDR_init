# Addressing and access rules

## Evidence and notation

**Interface** identifies loader/device-tree behavior, **instruction-level** identifies ARC instruction-level behavior, and **observation** identifies captured board or saved-image data. Conclusions without independent hardware confirmation are marked **inference**; unresolved behavior is **unknown**.

There are three distinct address views:

1. host DDRC/CFG byte offsets;
2. host DPHY 32-bit word indices;
3. ARC virtual addresses for DMEM and PMU-visible peripherals.

They are not interchangeable.

## Host physical bases

**Interface.** The board's DDRC addresses are confirmed independently by the SPL DTB and the DTB embedded in the reference FSBL:

| Resource | Instance 0 | Instance 1 | Address rule |
|---|---:|---:|---|
| DDRC | `0xcb000000` | `0xcc000000` | 4 MiB declared region |
| CFG/DPLL | `0xcb600000` | `0xcc600000` | `DDRC_BASE + 0x600000` |
| DPHY | `0xcb800000` | `0xcc800000` | `DDRC_BASE + 0x800000` |

The generic `0xc0000000` node is overridden for this board. The older `0xc0800000` DPHY derivation is likewise superseded by the confirmed `0xcb800000` base.

## DDRC and CFG: byte offsets

**Interface.** DDRC and CFG constants are byte offsets, even when implementation code converts them to a 32-bit array subscript:

```text
DDRC address = DDRC_BASE + byte_offset
CFG address  = DDRC_BASE + 0x600000 + byte_offset
```

Thus DDRC `mstr0` at byte offset `0x10000` is at `0xcb010000` for instance 0 and `0xcc010000` for instance 1. Host accesses to these registers are 32-bit.

## DPHY: host word indices

**Interface.** A DPHY constant `i` is an index into a 32-bit host pointer, not a byte offset:

```text
DPHY address = DPHY_BASE + 4 * i
```

A normal host transaction is 32 bits wide and addresses one 16-bit PHY CSR value in that word. Tables store 16-bit values even though the host pointer and bus transaction are 32-bit. This is a host-window convention; it does not make the PMU's own peripheral accesses 32-bit.

Examples:

| Host index | Purpose | Instance 0 address | Instance 1 address |
|---:|---|---:|---:|
| `0x41000` | ACSM SRAM window | `0xcb940000` | `0xcc940000` |
| `0x50000` | PMU IMEM | `0xcbc00000` | `0xccc00000` |
| `0x58000` | DMEM/message block | `0xcbe00000` | `0xcce00000` |
| `0x58009` | `HdtCtrl` | `0xcbe00024` | `0xcce00024` |
| `0xd0000` | Host/PMU control mux | `0xcbc40000` | `0xccc40000` |
| `0xd0099` | `MICRO_RESET` | `0xcbc40264` | `0xccc40264` |

**Unknown.** The source interface establishes the low 16-bit CSR payload convention, but it is not a complete electrical specification for upper-lane behavior or for arbitrary unaligned/16-bit host bus writes. Reimplementations should preserve the demonstrated 32-bit host access pattern rather than assume other host widths are equivalent.

## DPHY memory windows

| Start index | Window | Documented use |
|---:|---|---|
| `0x41000` | ACSM SRAM | 4096 halfwords saved and restored |
| `0x50000` | IMEM | Normal or quickboot PMU program, transferred as halfword values |
| `0x58000` | DMEM/message | `0xa0` useful message halfwords; `0x200` halfwords reserved |
| `0x58200` | Serialized PHY state | 5170 halfwords for quickboot |
| `0x584d2` | DMEM clear start | Start of the post-pre-training clear range |
| `0x59622` | Additional quickboot DMEM | `0x58000 + 0x200 + 5170` |
| `0x5aa00` | Partial quickboot boundary | End of a cleared/restored region in that flow |
| `0x60000` | DMEM limit | Upper limit used by the clearing loops |
| `0xa0000` | P-state SRAM | 16384-halfword resource; restoration is not active in the current flow |

The 5170-halfword PHY image consists of 69 ordered contiguous or sparse segments. Its bases and relative offsets are host DPHY indices. Segment 48 is the largest single contiguous run: 1536 halfwords at base `0x90000`.

## ARC-side addresses and actual access widths

**Instruction-level.** The PMU sees its own little-endian ARC address space. Do not add these addresses to a host DPHY base.

| ARC range/address | Purpose | Access rule established by firmware |
|---|---|---|
| `0x80000000` onward | DMEM, message fields, firmware state | 16-bit `ldh`/`sth` for DMEM fields; 8-bit accesses for packed byte tables |
| `0x80000400` | PHY-parameter/global area | Preserve loader-written state; it is not disposable zero-initialized storage |
| `0x80010000` | Initial stack top | Stack grows downward |
| `0x8000e000`–`0x8000e0ff` | Firmware message area | Host may consume/clear it; not persistent scratch |
| `0x8000e100 + 4*v` | Firmware progress-marker slots | Marker layout used by the execution environment |
| `0x90180008` | Mailbox status/ACK | 16-bit |
| `0x90180064` | Mailbox command or low halfword | 16-bit |
| `0x90180066` | Mailbox doorbell/ACK | 16-bit |
| `0x90180068` | Mailbox high halfword | 16-bit |
| `0x90xxxxxx` | PMU-visible PHY/training peripherals | 16-bit accesses in the analyzed firmware |

**Instruction-level.** Using a 32-bit C object for a DMEM or `0x90xxxxxx` peripheral field does not model the observed interface and can read the wrong value or stall the core. Packed structures with a stride not divisible by four must be accessed bytewise where the firmware does so; a wider unaligned access is unsafe on this core.

The ARC timer at AUX registers `0x103`, `0x104`, and `0x105` is accessed with ARC `sr`/`lr` instructions, not as DMEM or a `0x90xxxxxx` peripheral. Its low/high counter reads support firmware delays.

### Lane, channel, and rank CSR addressing

Many per-lane and per-rank PHY CSR accesses compute their effective address by combining the channel selector, rank selector, and a family-specific aligned base constant using **bitwise OR** (`|`), never addition (`+`):

```text
address = (channel << N) | rank_bits | base_constant
```

Where the shift `N` and `rank_bits` depend on the CSR family. Observed PMU-side base constants include `0x90020158`, `0x9002016e`, `0x9003e126`, and `0x90060002`. An implementation must preserve bitwise OR composition so that selector bits never carry into adjacent address fields.

## Host PMU control and auxiliary CSRs

In addition to the mailbox indices below, the host controls PMU execution, clocks, and capture modes through the following DPHY word indices:

| Host DPHY index | Interface role | Observed values / behavior |
|---:|---|---|
| `0xc0080` | `UCCLK_HCLK_ENABLES` | Clock enables (`1`, `2`, `3`, `7` depending on stage) |
| `0xc0086` | ECC control | Written `1` in quickboot to disable ECC |
| `0xd0000` | `MICRO_CONT_MUX_SEL` | `0` = host/APB owns PHY CSRs; `1` = PMU owns PHY CSRs |
| `0xd0003` | Training phase auxiliary | Written `0` at the end of the post-run training table |
| `0xd0036` | APB access mode | Written `0` in quickboot (APB-only mode) |
| `0xd0099` | `MICRO_RESET` | `9 -> 1 -> 0` releases PMU execution; `1` stalls/resets PMU |
| `0xd00e7` | Quickboot finalization | Written `0x0600` at the end of quickboot |
| `0x20037` | ACSM capture toggle | Bit 6 set before ACSM collection and cleared after stopping PMU |
| `0x200c9` | PLL lock status | Read as diagnostic data after fatal message `0xff` |

## Mailbox views

The host and PMU use different addresses for the same protocol:

| Function | Host DPHY index | PMU ARC address |
|---|---:|---:|
| Status/busy | `0xd0004` | `0x90180008` |
| Acknowledge/protection | `0xd0031` | `0x90180066` |
| Major message or streaming low halfword/count | `0xd0032` | `0x90180064` |
| Streaming high halfword | `0xd0034` | `0x90180068` |

**Interface + instruction-level.** The PMU waits for a free mailbox, writes message halves, and signals the doorbell. The host acknowledges by writing 0, waiting for the consumed state, and writing 1. Before the next item it waits for the free state. Major value `0x07` denotes completion, `0x08` introduces streaming data, and `0xff` enters the documented error diagnostic path. These polls have no timeout in the analyzed interface.

A streaming header contains an opcode and payload length followed by that many 32-bit words, transported as halfword pairs. **Unknown.** The format is established, but most opcode semantics are not.

## DDRC address mapping and ports

**Interface.** The DDRC address map is separate from its MMIO address:

- `SARBASE0..3` and `SARSIZE0..3`, at byte offsets `0x200c0`–`0x200dc`, describe four 256 MiB-aligned regions beginning at the 2 GiB DRAM base.
- `ADDRMAP1` at `0x30004` controls rank/CS mapping; the single-rank configuration uses the documented inactive value `0x3f`.
- `ADDRMAP3..12` occupy `0x3000c`–`0x30030`.
- Five ports are enabled through `pctrl` at `0x20090`, `0x21090`, `0x22090`, `0x23090`, and `0x24090`.

**Unknown.** The available evidence does not fully decode every bank, row, column, channel-interleave, or system-address bit. Do not derive a complete physical-to-DRAM mapping solely from the region registers.

## Access-safety conclusion

**Inference supported by observations.** Direct post-boot DPHY access is not a safe debugging shortcut. One older attempted address was also based on the wrong DPHY base, which invalidates any explanation that attributed the resulting stall solely to PMU power-down. Even with corrected bases, access should remain within the demonstrated SPL ownership and clock/mux phases; concurrent host access while the PMU owns the PHY is outside the contract.
