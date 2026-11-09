# DDR system architecture

## Scope and evidence

This document describes the K3 LPDDR5 initialization path implemented by the first-stage loader. It reconciles the software interface, the PMU firmware interface, and board observations; it is not a complete Synopsys controller or PHY hardware manual.

Evidence labels used throughout these documents:

- **Interface**: a fact derived from loader interfaces, data structures, or device-tree data.
- **Instruction-level**: a fact derived from PMU instruction-level behavior.
- **Observation**: a result captured from a board or persisted training data.
- **Inference**: a conclusion that follows from those facts but is not independently guaranteed by hardware documentation.
- **Unknown**: behavior that the available evidence does not establish.

## Two complete controller instances

**Interface.** The reference board declares two 4 MiB DDRC regions. Two independent DTBs agree on both addresses:

| Instance | DDRC base | Clock/CFG base | DPHY base |
|---|---:|---:|---:|
| 0 | `0xcb000000` | `0xcb600000` | `0xcb800000` |
| 1 | `0xcc000000` | `0xcc600000` | `0xcc800000` |

For each instance, `CFG = DDRC + 0x600000` and `DPHY = DDRC + 0x800000`. The CFG and DPHY areas are derived windows, not part of the 4 MiB DDRC `reg` extent. The generic SoC address `0xc0000000` and the older DPHY address `0xc0800000` do not describe this board's instance 0 and must not be used.

**Interface.** The loader initializes instance 0 and instance 1 sequentially. Each receives its own PHY setup, PMU execution, and training result. This sequencing does not by itself prove that both instances are electrically identical or that their trained values may be interchanged.

## Functional blocks and ownership

| Block | Responsibility | Programming view |
|---|---|---|
| Host loader | Selects the memory part and rate, programs clocks and DDRC, loads and supervises the PMU, saves results, and enables traffic | Host MMIO |
| CFG/DPLL | Produces and selects DDR clocks, reports PLL lock, and gates DCLK | Byte offsets from CFG base; host 32-bit accesses |
| DDRC | Owns DRAM timings, DFI, refresh, ZQ, rank/address mapping, power state, and host ports | Byte offsets from DDRC base; host 32-bit accesses |
| DPHY | Provides lanes, delay and drive controls, training engines, PMU memories, and host/PMU register windows | Host word indices, normally carrying 16-bit CSR values |
| PMU | Embedded little-endian ARCv2/EM processor that executes training and exchanges messages with the loader | ARC DMEM and CSR addresses |
| LPDDR5 array | Stores data and receives mode, calibration, and training traffic through the PHY | One or two ranks; x16 or x8 device organization in the supported table |

**Interface + instruction-level.** Training runs on the PMU, not on the main RISC-V processor. The host configures the environment and starts the PMU; the PMU performs the PHY training algorithms. The later host memory-pattern test is a separate check against an already operational DRAM interface.

**Instruction-level.** The K3 PMU implements the reduced ARC register file: general registers `r16`–`r25` are absent. Its firmware interface uses 16-bit DMEM and PMU/PHY peripheral accesses, with byte accesses for packed tables whose stride is not naturally aligned. These are ARC-side rules and differ from the host's 32-bit DPHY bus transactions.

## Full-training sequence

The following is the reconciled functional order. Individual register writes within a stage can be more numerous and phase-dependent.

1. **Select configuration — interface.** Resolve the part number, rank count, x8/x16 organization, data rate, and I/O parameters. The supported LPDDR5 rate set is 4800 and 6400 MT/s.
2. **Start clocks — interface.** Program the DPLL, wait for lock, switch divisors/source, enable the resulting clock, and later release DCLK to the PHY.
3. **Configure the DDRC — interface.** Program DFI, part/rate-specific timings, refresh, derating, ZQ, rank presence, and the address map. Keep host traffic disabled.
4. **Prepare the PHY and PMU — interface.** Apply pre-training tables with topology and I/O overrides, load ARC firmware into IMEM, construct the DMEM message block, and clear the specified unused DMEM range.
5. **Run the PMU — interface.** Give PHY control to the PMU and apply the `MICRO_RESET` sequence `9 -> 1 -> 0`. Consume major and streaming mailbox messages until completion or an error indication.
6. **Run the training phase — interface.** Return register access to the host, apply the training table and topology overrides, then restore the clock/mux state expected for result collection.
7. **Collect results — interface.** Save the 320-byte message block, exactly 5170 serialized PHY halfwords, and 4096 ACSM SRAM halfwords for the instance.
8. **Activate the DDRC — interface.** Transition DFI and power-control state, wait for the operational status, adjust ZQ/refresh state, leave self-refresh, and enable all five DDRC ports.
9. **Check memory — observation/interface.** The host tests and restores `0x4000`-byte blocks with address-based and complemented patterns. Passing this test is useful evidence, not a exhaustive DRAM validation.

The PMU owns the PHY while the control mux selects it. The host must not concurrently manipulate the same PHY window.

## Quickboot and persistence

**Interface.** Quickboot reuses a previous result when the persisted envelope passes magic, DDR generation, rank-count, data-rate, and CRC checks. Otherwise the loader performs full training. Each controller has a separate `0x49a4`-byte result; the reserved two-controller envelope is `0x9800` bytes.

Quickboot loads a different PMU program, restores the saved message and serialized PHY state, restores ACSM state, and follows its own PMU and clock sequence. It does not run the ordinary pre-training and training tables. P-state SRAM restoration is not active in the documented path.

**Observation.** One reference-board comparison measured approximately 5.6 seconds for full training and 9 ms for quickboot. These figures characterize that boot only; they are not architectural timing limits.

## Training-result boundaries

The saved result contains:

- a `0x140`-byte used message block, within a `0x200`-halfword DMEM reservation;
- 5170 PHY halfwords serialized in a fixed order of 69 segments;
- 4096 ACSM halfwords.

**Interface.** Serialization order is part of the quickboot ABI. Sorting the PHY indices or compacting the segments would produce a different, incompatible image.

**Observation.** Identical PHY/ACSM snapshots were captured for both controllers in one same-part reference-board envelope. **Unknown.** This single observation does not establish that controller results are always identical or safely portable between boards, parts, rates, or firmware versions.

## Deliberate limits

The available material does not provide a complete bit-level DDRC/DPHY specification, authoritative meanings for all PMU streaming opcodes, or proof of interoperability for every listed part/rate/topology combination. Those limits must remain visible rather than being filled with inferred register names or universal timing claims.
