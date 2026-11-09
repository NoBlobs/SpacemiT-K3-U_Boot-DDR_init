# DDRC, DPLL, and host-port control

## Scope and evidence

This chapter covers the host-programmed clock and controller path for each of the two K3 LPDDR5 instances. **Interface** facts come from loader behavior and extracted configuration outputs; **observations** come from board execution. Phase interpretation is marked **inference**, and missing hardware definitions are **unknown**.

All offsets in this chapter are **byte offsets**, not DPHY word indices. Use DDRC bases `0xcb000000` and `0xcc000000`; CFG bases are respectively `0xcb600000` and `0xcc600000`.

## DPLL and clock selection

**Interface.** The supported LPDDR5 paths program the following values:

| DRAM data rate | DPLL setting | Source selection in `CFG+0x18` | PMU `DRAMFreq` |
|---:|---:|---:|---:|
| 4800 MT/s | 3000 MHz | `7 << 19` | `0x12c0` |
| 6400 MT/s | 3200 MHz | `2 << 19` | `0x1900` |

`DRAMFreq` is the rate in MT/s represented in hexadecimal (`4800 = 0x12c0`, `6400 = 0x1900`), not the DPLL frequency in MHz.

| CFG byte offset | Established role |
|---:|---|
| `0x08` | Part of the 3000 MHz DPLL programming path |
| `0x0c` | DPLL configuration; 3200 MHz is programmed here on the default 6400 path |
| `0x10` | Additional 3000 MHz DPLL programming |
| `0x18` | Divisor/source selection, clock-switch kick, DCLK gate, and final enable |
| `0x1c` | Bit 0 polled for DPLL lock |

The observed sequence is:

1. program the DPLL;
2. wait for `CFG+0x1c[0]` to indicate lock;
3. clear `CFG+0x18[21:16]` and install the rate-specific source selection;
4. set bit 25 to start the switch and wait for it to clear;
5. set bit 0 for the final enable;
6. before PHY execution, set bit 22 to release DCLK.

**Unknown.** The DPLL multiplier/divider field meanings and the relationship between every internal clock are not fully documented here. In particular, `DfiFreqRatio` must be preserved as the interface value expected by the PMU; the sources do not justify redefining it as a generic WCK/MemClk ratio. Rates other than 4800 and 6400 MT/s are rejected by this LPDDR5 path rather than approximated.

## DDRC configuration stages

**Interface.** Before training, the loader configures:

- DRAM timing sets for the selected part, rate, rank count, and x8/x16 organization;
- DFI frequency, update, PHY-master, and low-power timing controls;
- refresh and derating;
- ZQ calibration controls;
- rank presence and CS/address mapping;
- power and initialization state.

After training, it collects PHY state, transitions DFI to normal operation, waits for DDRC operational status, adjusts refresh/ZQ state, exits self-refresh, and only then enables host ports.

The `swctl`/`swstat` handshake brackets applicable configuration updates. `stat[1:0] == 01` is the final operational-state condition used by the loader.

## Principal DDRC registers

The table records roles established by the interface. It does not assign undocumented bitfields.

| Register/group | Byte offset(s) | Role in this flow |
|---|---|---|
| DRAM timing sets | `0x00000`–`0x00080` | Rate/topology-specific DRAM timing values |
| DFI timing/update | `0x00580`–`0x00594`, `0x005a4`–`0x005b8` | DFI frequency, low-power, and update timings |
| Refresh | `0x00600`–`0x00608`, `0x00650` | Refresh timing and related controls |
| ZQ timing/configuration | `0x00800`–`0x00804` | Rate/topology-specific ZQ values in the timing output |
| Derating | `0x00b04`–`0x00b08` | Temperature/rate-related derating configuration |
| Rank timing/config | `0x00d00`, `0x00d04`, `0x00d08` | Timing configuration and rank-dependent values |
| `mstr0` | `0x10000` | Master DDR configuration, including rank-dependent state |
| `deratectl1/2` | `0x10104`, `0x10108` | Derating controls |
| `pwrctl` | `0x10180` | Power/self-refresh sequencing |
| `rfshmod0` | `0x10200` | Refresh mode, with rank-dependent values |
| `sched0` | `0x10380` | Scheduler configuration |
| `dfiupd0` | `0x10508` | DFI update control |
| `dfimisc` | `0x10510` | DFI initialization and operational transitions |
| `dfiphymstr` | `0x10518` | DFI/PHY-master interface |
| `opctrl1` | `0x10b84` | Operational control during initialization |
| `inittmg0` | `0x10d00` | Initialization timing |
| `stat` | `0x10014` | Controller status; final wait for bits `[1:0] == 01` |

Additional ZQ writes occur in the high control area, including `0x10280` and `0x10288`. **Unknown.** The sources do not provide a complete authoritative register-name map proving how every low-offset ZQ timing register and high-offset ZQ control register corresponds to Synopsys architectural names. Preserve their offsets and sequence instead of treating them as aliases.

## DFI and power-state values are phase-dependent

**Interface.** Different snapshots contain different values for the same register. For example:

- `dfimisc` has an extracted timing-stage value of `0x00010005`, while the wider initialization/activation flow also uses values ending in `0x14`, `0x34`, and `0x15`.
- `opctrl1` is `3` in the extracted timing output; another preparation description records `2` at a different point.
- `pwrctl` is `0x11` in the extracted timing output, while activation uses the documented `0x811`/`0x800` sequence, bit 11 handling, and final `0x10` self-refresh exit.

**Inference.** These are stage differences, not evidence that one arbitrary value can replace the whole sequence. A reimplementation should place writes in their original phase and perform the associated polling. **Unknown.** The available reconciled sources do not establish a complete cycle-accurate ordering for every intermediate write, so the numeric examples must not be collapsed into a single flat register image.

## Address map

**Interface.** Four system-address regions are programmed with `SARBASE0..3` and `SARSIZE0..3` at `0x200c0`–`0x200dc`. They are aligned to 256 MiB and based at 2 GiB. `ADDRMAP1` at `0x30004` differs between single- and dual-rank configurations, and `ADDRMAP3..12` at `0x3000c`–`0x30030` complete the implemented map.

**Unknown.** The sources do not completely specify channel interleaving or every row/bank/column bit. Capacity and address mapping must therefore be derived and validated for the actual part, not inferred only from the rank field.

## Five host ports

Each DDRC instance has five port-control registers in the analyzed flow:

| Port | Byte offset | Final enable value |
|---:|---:|---:|
| 0 | `0x20090` | `1` |
| 1 | `0x21090` | `1` |
| 2 | `0x22090` | `1` |
| 3 | `0x23090` | `1` |
| 4 | `0x24090` | `1` |

**Interface.** These writes occur after the controller reaches its operational state and leaves self-refresh. Enabling a port earlier would expose traffic to an uninitialized or training-owned interface.

**Unknown.** The evidence establishes five software-controlled ports and their enable sequence, but does not identify each port's SoC master, QoS capabilities, or bandwidth allocation. Those should not be inferred from port number alone.
