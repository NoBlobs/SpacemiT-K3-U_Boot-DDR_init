# Supported parts, topologies, and timing scope

## Evidence and scope

**Interface** facts describe the loader's tables and accepted inputs. **Observation** identifies the reference board and captured results. **Inference** and **unknown** are used where coverage or interpretation is not proven. This chapter is primarily about LPDDR5; LPDDR4X entries elsewhere in the tree do not extend the LPDDR5 timing or PMU contract described here.

## Supported LPDDR5 part table

| Part number | CRC-32 used for lookup | Ranks | Device organization flag | Listed density |
|---|---|---:|---|---:|
| `MT62F1G32D2DS` | `0x0fd38dd9` | 1 | x16 (`x8_mode = 0`) | 4096 MiB |
| `MT62F2G32D4DS` | `0x85d1f688` | 2 | x16 (`x8_mode = 0`) | 8192 MiB |
| `RS3G32LG5D8FDB` | `0xf74c6bfc` | 2 | x8 (`x8_mode = 1`) | 12288 MiB |
| `MT62F4G32D8DV` | `0x3acef2e4` | 2 | x8 (`x8_mode = 1`) | 16384 MiB |

These are table capacities, not proof of a particular board's total capacity across both DDRC instances. Physical channel population and system address interleaving are not fully established by the part table.

### LPDDR4X part table (for lookup reference only)

The same part-lookup table also lists three LPDDR4X entries so the loader can identify generation (`type = 0` for LPDDR4X, `type = 1` for LPDDR5):

| Part number | CRC-32 used for lookup | Ranks | Device organization flag | Listed density |
|---|---|---:|---|---:|
| `MT53E1G32D2FW` | `0x75251ab8` | 1 | x16 (`x8_mode = 0`) | 4096 MiB |
| `MT53E2G32D4DE` | `0x3ea87223` | 2 | x16 (`x8_mode = 0`) | 8192 MiB |
| `MT53E4G32D8CY` | `0xaa9d4848` | 2 | x8 (`x8_mode = 1`) | 16384 MiB |

**Interface.** The part string is obtained from board identification data, with a device-tree property as fallback. Lookup uses CRC as a filter and requires a string match. If no entry matches, the current selection behavior falls back to the first table entry. That fallback is not automatic detection of an unknown memory device; a product may instead choose to reject it.

**Observation.** The reference board uses `MT62F2G32D4DS`, two ranks, x16 organization, and 6400 MT/s. Its persisted training envelope agrees with those settings.

## Supported rates and topology matrix

The LPDDR5 initialization path accepts only:

- **4800 MT/s**, with 3000 MHz DPLL programming, CFG selector `7 << 19`, and `DRAMFreq = 0x12c0`;
- **6400 MT/s**, with 3200 MHz DPLL programming, CFG selector `2 << 19`, and `DRAMFreq = 0x1900`.

**Interface.** The pre-training base table is for dual-rank x8 at 6400 MT/s. Overrides cover:

| Topology | 4800 MT/s | 6400 MT/s |
|---|---|---|
| Single-rank x16 | PHY override present | PHY override present |
| Dual-rank x16 | PHY override present | PHY override present; reference timing configuration available |
| Dual-rank x8 | PHY override present | Base PHY configuration; reference timing configuration available |

The message block carries `EnabledDQsChA/B` and `CsPresentChA/B` masks. These masks, the x8/x16 overrides, and DDRC rank/address-map settings must describe the same topology. A part match alone is not sufficient.

**Unknown.** The existence of table entries and overrides is not equivalent to board-level validation of every part at both rates. No universal training-result compatibility should be assumed across topology, rate, part, or PMU-program changes.

## PHY configuration precedence

For a given PHY CSR write, priority from lowest to highest is:

1. base pre-training/training table value;
2. rate/topology override;
3. I/O override.

A topology override value of `0xdeadbeef` suppresses that write. The host table representation carries 16-bit CSR values even though it addresses the DPHY through 32-bit words.

Default LPDDR5 I/O policy in the interface is:

| Setting | Configured policy |
|---|---|
| PHY write drive | R30 equivalent |
| PHY receive ODT | R60 equivalent |
| DQ ODT | R60 equivalent |
| CA ODT | R80 equivalent |
| NT/SoC ODT | Disabled |
| PDDS | R40 equivalent |
| 2D training | Enabled |

These resistance labels are configuration choices, not raw register encodings. The settings affect both lane CSRs and DRAM mode-register fields.

## Host DDRC timing dependencies

Host-side DDRC timing selection is not a simple frequency scaling exercise:

- At 6400 MT/s, changing from one to two x16 ranks changes DRAM timing entries, DFI values, rank timing, `mstr0` (`0x01080008` for 1 rank vs. `0x03080008` for 2 ranks), `rfshmod0` (`0x010003f3` vs. `0x00000361`), and derating controls.
- At two x16 ranks, changing from 6400 to 4800 MT/s changes rate-dependent timing and refresh/ZQ values.
- At 6400 MT/s and two ranks, changing x16 to x8 changes rank and operational timing fields.

**Interface.** A nanosecond-to-cycle conversion commonly needs upward rounding, representable as `ceil(time_ns * data_rate_MT_per_s / 2000)`. **Unknown.** That arithmetic alone is insufficient to reconstruct the full host DDRC timing table: register-specific rounding, minimum/maximum constraints, mode-register rules, rank interactions, and DFI/PHY timing requirements also apply.

## Persistence and timing compatibility

Each controller's quickboot result contains a 320-byte message block, 5170 PHY halfwords, and 4096 ACSM halfwords. The 5170-halfword PHY serialization has a fixed 69-segment order; contiguous and sparse segments must remain in that exact order.

**Interface.** Quickboot acceptance checks the envelope identity, DDR type, ranks, data rate, and CRC. This ties a result to selected configuration properties but does not turn training output into a universal timing database. P-state SRAM restoration is not active in the documented quickboot path.

## Coverage gaps to preserve

- Full DDRC bitfield definitions and all initialization-phase transitions are not established.
- PHY lane timing, trained delay values, and most streaming-opcode meanings are outside host DDRC timing configuration.
- Support in a lookup or override table does not establish electrical validation for another board or memory package.

## PHY table descriptor formats

PHY configuration tables applied by the host use two descriptor representations:

- **Discrete table**: an ordered list of explicit `(csr_index, halfword_value)` pairs.
- **Linear table**: a descriptor specifying `base` (starting CSR word index), `count` (number of entries), `linear_increase` (`1` when indices increment by 1), and a pointer to the contiguous halfword values.

## Summary of size and sentinel constants

| Constant | Value | Scope / Usage |
|---|---|---|
| Message block useful / reserved size | `0xa0` / `0x200` halfwords (`0x140` / `0x400` bytes) | DMEM at index `0x58000` |
| Serialized `phypara` length | `5170` halfwords (`0x2864` bytes across 69 segments) | Result collection & quickboot (`0x58200`) |
| ACSM SRAM snapshot length | `4096` halfwords (`0x2000` bytes) | Result collection & quickboot (`0x41000`) |
| P-state SRAM size | `16384` halfwords (`1 << 14`) | Index `0xa0000` (optional / inactive in current quickboot) |
| Per-controller saved record | `0x49a4` bytes | Persistent envelope (`0x140 + 0x2864 + 0x2000`) |
| Reserved two-controller envelope | `0x9800` bytes | Persistent storage (`bootinfo` offset `0x10000`) |
| Host memory smoke-test block | `0x4000` bytes | Post-activation verification |
| Envelope magic | `0x54524444` | ASCII `"DDRT"` |
| Topology override skip sentinel | `0xdeadbeef` | Suppresses the corresponding CSR write |
| Mailbox major completion | `0x07` | Normal PMU training/quickboot completion |
| Mailbox streaming introducer | `0x08` | Followed by word count and 32-bit framed words |
| Mailbox fatal error | `0x000000ff` | Terminal PMU error notification |
