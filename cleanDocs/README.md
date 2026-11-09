# SpacemiT K3 DDR/LPDDR5 Clean-Room Specification

Independent interface and behavioral specification of the DDR/LPDDR5 memory subsystem on the SpacemiT **K3** SoC. This documentation is designed to allow an independent implementation team (**without access to vendor source code or binaries**) to build a compatible DDR initialization loader and/or PHY training microcontroller (**PMU**) firmware.

## Why "Clean Room"

This project follows the standard clean-room separation:

- The **analysis room** examined hardware behavior on the bench, bus transactions, boot measurements, and the public **ARCv2/EM** instruction set architecture to produce this interface specification.
- The **implementation room** must work **exclusively** from this specification.

By transferring only **interface facts, hardware constraints, memory maps, data formats, and required operational sequences**—and never vendor code expression, internal symbol names, or proprietary documentation—an implementation built from these documents remains an independent work.

> **Warning:** This is an **interface and hardware contract specification**, not source code. Do not consult vendor firmware sources, decompiled listings, or vendor binaries when implementing from this specification. Where behavior is not established here (see [15-open-questions.md](15-open-questions.md)), treat it as an open engineering question to be resolved by controlled hardware testing.

---

## Hardware Safety Warning

**Never read or write PHY/PMU MMIO registers from an operating system after boot (for example via `/dev/mem`).** Once the bootloader finishes and hands off to the OS, the PHY/PMU interface is placed in a state where host MMIO reads stall the SoC interconnect, causing an immediate system hang (loss of serial console and network) that requires a full physical power cycle. All PHY/PMU register accesses must take place inside the first-stage bootloader while the PMU clock and ownership state are properly configured, or via JTAG. See [14-validation-and-current-status.md](14-validation-and-current-status.md) and [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md).

---

## Scope

### Covered

- Dual-instance DDR controller (**DDRC**), clock/PLL (**CFG**), and **DPHY** architecture;
- Address maps, bus access conventions, and lane CSR addressing rules;
- Supported LPDDR5 memory parts, topologies, data rates (4800 and 6400 MT/s), and I/O electrical policies;
- PMU memory windows (**IMEM**, **DMEM**, **ACSM SRAM**, **P-state SRAM**) and the 320-byte **message block**;
- Complete 9-stage full-training initialization sequence and PHY training behavioral model;
- Host-to-PMU **mailbox** handshake, major stage codes, and **streaming** message transport;
- Training result collection, the 69-segment **5170-halfword `phypara`** serialization contract, and the **`0x9800`-byte persistent envelope**;
- **Quickboot** fast restoration sequence (9 ms vs. ~5.6 s full training);
- **ARCv2/EM execution environment**, reduced register file (`-mrf16`), auxiliary timer (`RTC`), memory layout, and toolchain/hardware behavioral requirements;
- Bench measurements, validation criteria, flash layout, and FDL recovery procedures.

### Out of Scope

- Full LPDDR4X initialization (LPDDR4X part-table entries are listed for lookup reference only);
- Unrelated SoC peripherals, CPU cores, or GPU subsystems;
- Proprietary internal register bitfield names not required for hardware interoperability.

---

## Reading Order and Contents

| Document | Topic | Summary |
|---|---|---|
| [01-system-architecture.md](01-system-architecture.md) | System Architecture | Dual controller instances, functional blocks, full-training vs. quickboot overview |
| [02-addressing-and-access.md](02-addressing-and-access.md) | Addressing & Access Rules | Byte offsets vs. 32-bit word indices, ARC virtual addresses, lane CSR addressing |
| [03-ddrc-and-clock.md](03-ddrc-and-clock.md) | DDRC, DPLL & Host Ports | Clock/PLL programming, DDRC configuration phases, address map, 5 host ports |
| [04-parts-topologies-and-timings.md](04-parts-topologies-and-timings.md) | Parts, Topologies & Timings | Part lookup tables, rate/rank/width matrices, PHY table formats, size/sentinel constants |
| [05-message-block.md](05-message-block.md) | PMU Message Block | 320-byte host/PMU data interface, control flags, mode registers, trained outputs |
| [06-full-initialization.md](06-full-initialization.md) | Full Initialization Sequence | Step-by-step 9-stage initialization, ownership transitions, collection, and activation |
| [07-training-behavior.md](07-training-behavior.md) | PHY Training Behavior | Swept dimensions, stage dependencies, 2D eye scoring, margins, and acceptance policy |
| [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md) | Mailbox & Streaming Protocol | Handshake, major messages (`0x07`, `0x08`, `0xff`), streaming framing, and error policy |
| [09-training-results.md](09-training-results.md) | Training Results | Per-controller `0x49a4` record (`message`, `phypara`, `acsm`) and reference sample stats |
| [10-persistence-and-selection.md](10-persistence-and-selection.md) | Persistence & Selection | `0x9800` SPI flash envelope (`DDRT`), CRC-32 coverage, and quickboot selection rules |
| [11-quickboot.md](11-quickboot.md) | Quickboot | Result-to-MR shadow mapping, DMEM/ACSM restoration, MemReset, and 10-step fast flow |
| [12-arc-execution-environment.md](12-arc-execution-environment.md) | ARC Execution Environment | ARCv2 EM core, `0x1fc` entry, `-mrf16` register file, `gp`/`.bss` split, AUX RTC timer |
| [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md) | Firmware Behavioral Requirements | Normative hardware, ISA, memory-width, timing, and toolchain rules (BR-01 to BR-13) |
| [14-validation-and-current-status.md](14-validation-and-current-status.md) | Validation & Measurements | Reference platform benchmarks, boot timings, validation matrix, and safe observation rules |
| [15-open-questions.md](15-open-questions.md) | Open Questions & Gaps | Unresolved hardware, protocol, calibration, and validation questions |
| [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md) | Flash, Build & Recovery | Unsigned vs. signed boot chains, ARC/SPL build flags, SPI NOR flashing, and FDL recovery |
| [appendix-message-block-fields.md](appendix-message-block-fields.md) | Appendix: Message Block Fields | Complete 293-field catalog (offsets `0x000`–`0x13f`, CSR indices, types, directions) |
| [appendix-phypara-order.md](appendix-phypara-order.md) | Appendix: `phypara` Order | Exact 69-segment, 5170-halfword serialization table |
| [appendix-streaming-catalog.md](appendix-streaming-catalog.md) | Appendix: Streaming Catalog | Catalog of 124 observed streaming message IDs, payload lengths, and trace examples |

---

## Notation and Evidence Conventions

- **Host word index (`i`)**: Used in DPHY/PMU windows. Host bus byte address is `DPHY_BASE + 4 * i`. Each 32-bit host word carries one 16-bit PHY CSR in its lower halfword.
- **Byte offset**: Used in DDRC and CFG/DPLL blocks (`DDRC_BASE + byte_offset`, `CFG_BASE + byte_offset`), as well as inside the 320-byte message block and persistent envelope.
- **ARC address**: 32-bit address in the PMU's own little-endian memory space (`0x0000xxxx` IMEM, `0x8000xxxx` DMEM, `0x90xxxxxx` PHY/mailbox peripherals). Never add an ARC address to `DPHY_BASE`.
- **Evidence labels**:
  - **Observed / Interface**: Established from hardware behavior, bus traces, device trees, or interface structures.
  - **Inference / Recommended**: Engineering conclusion or recommended policy for a new implementation, kept distinct from observed hardware facts.
  - **Unknown**: Not established by available evidence; must not be guessed or filled from proprietary sources.

---

## Intellectual Property and Clean-Room Boundaries

- This specification is an **independent description of hardware interfaces and observable behavior**. It contains no vendor source code, binary code, or proprietary text.
- Interface field names (such as message-block field identifiers) are interoperability identifiers used nominatively to describe the hardware/firmware data contract.
- Memory part numbers and product names belong to their respective owners and are used solely to identify target hardware configurations.
- The **ARCv2/EM** instruction set architecture is publicly documented; this specification describes only the subset and hardware constraints implemented by the K3 PMU core.
