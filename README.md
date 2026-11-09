<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# SpacemiT K3 — Free LPDDR5 Initialization and PMU Firmware

This project aims to produce a free (GPLv3, GNU C23) LPDDR5 training and
quickboot firmware for the embedded **ARCv2/EM** microcontroller (**PMU**) inside
the DDR PHY of the **SpacemiT K3** RISC-V SoC.

During early boot, the first-stage bootloader (U-Boot SPL) configures the two DDR
controllers (**DDRC0** at `0xcb000000` and **DDRC1** at `0xcc000000`) and their clocks,
loads a firmware image into the PHY PMU's instruction memory (**IMEM** at CSR index
`0x50000`), populates a 320-byte **message block** in data memory (**DMEM** at CSR index
`0x58000`), and releases the PMU (`MICRO_RESET = 9 -> 1 -> 0`). Up to now, the PMU
training and quickboot programs distributed for the K3 have been **proprietary binary
blobs**. This project replaces those blobs with free software built from a clean-room
hardware specification.

## Clean-room process

The work is split between two teams to ensure the resulting firmware is an
independent work rather than a derivative of the vendor's binary blobs:

1. **Dirty team (hardware & interface specification).** The analysis team examined
   the K3 hardware on the bench, bus transactions, boot measurements, and the public
   ARCv2/EM instruction set architecture, and consolidated every hardware and
   interoperability fact into [cleanDocs/](cleanDocs/README.md). It describes only
   addresses, register access widths, message layouts, mailbox protocols, ordering
   rules, processor constraints, and known gaps — never vendor code expression.
2. **Clean team (implementation).** Everything outside `cleanDocs/` in this
   repository is written exclusively from:
   - [cleanDocs/README.md](cleanDocs/README.md);
   - public JEDEC LPDDR5 specifications (JESD209-5) and DRAM datasheets;
   - public **ARCv2/EM** ISA and GNU toolchain documentation;
   - public U-Boot SPL sources and device trees for the SpacemiT K3;
   - controlled experiments on the reference board using **this** firmware.

The clean team never reads, runs, or disassembles the proprietary PMU firmware blobs.
New findings made while implementing and testing are recorded in `docs/`, never copied
back into `cleanDocs/`.

```
hardware & bus analysis ──> cleanDocs/ (read-only spec) ──> src/ (free PMU firmware)
                                                                │
                                 docs/ <── new bench findings ──┘
                                 tools/ <── build, pack & FDL test helpers
```

## Reference hardware

- **Board:** SpacemiT K3 Pico-ITX
- **SoC:** SpacemiT K3, 16 RISC-V cores, up to 2.2 GHz
- **DRAM:** 16 GiB LPDDR5 across two controller instances (Micron **MT62F2G32D4DS**,
  2 ranks, x16 organization, 6400 MT/s)
- **PMU core:** 32-bit little-endian **ARCv2 EM** microcontroller with the reduced
  16-register file (`RF16`: `r0`–`r15`, `gp`, `fp`, `sp`, `r30`, `blink`; registers
  `r16`–`r25` are absent in silicon)

> **Hardware safety rule:** Never read or write DPHY/PMU MMIO registers from Linux
> user space (e.g. via `/dev/mem`) after boot. Once the bootloader hands off to the OS,
> accessing the PHY/PMU window stalls the SoC interconnect and requires a physical
> power cycle. All PHY/PMU accesses must occur in the first-stage bootloader while the
> PMU is powered and clocked, or via JTAG. See
> [cleanDocs/14-validation-and-current-status.md](cleanDocs/14-validation-and-current-status.md).

## Repository layout

```
cleanDocs/               clean-room hardware and interface specification (read-only)
src/                     ARCv2/EM PMU firmware source code (C23 + ARC assembly)
tools/                   image packing, ELF auditing, and FDL staging scripts
docs/                    implementation notes, bench logs, and open questions
LICENSE                  GNU GPL version 3
```

## Key ARCv2/EM firmware build constraints

Any firmware compiled for the K3 PMU must obey the hardware constraints detailed in
[cleanDocs/12-arc-execution-environment.md](cleanDocs/12-arc-execution-environment.md) and
[cleanDocs/13-firmware-behavioral-requirements.md](cleanDocs/13-firmware-behavioral-requirements.md):

1. **Pass `-mrf16` at both compile and link stages:** The PMU core does not implement
   `r16`–`r25`. When using `-flto`, the linker regenerates code and will emit illegal
   registers unless `-mrf16` is passed to the link command as well. Always verify with:
   ```sh
   arc-elf32-objdump -d firmware.elf | grep -E '\br(1[6-9]|2[0-5])\b'
   ```
2. **Entry point and memory map:** Reset at `0x0` must branch to effective startup at
   `0x1fc`. Stack top is `0x80010000`. Do not place `.bss` at `0x80000400` (which holds
   live SPL-supplied `phypara` and ODT tables); place `.bss` at `0x80004000`.
3. **16-bit access widths:** Access DMEM (`0x8000xxxx`) and PMU/PHY CSRs (`0x90xxxxxx`,
   including mailbox `0x90180008`/`64`/`66`/`68`) as 16-bit halfwords (`ldh`/`sth`), and
   read packed tables whose stride is not a multiple of 4 (such as 6-byte descriptors)
   byte-by-byte (`ldb`).
4. **Hardware timer and arithmetic:** Access the 64-bit free-running RTC (`0x103`,
   `0x104`, `0x105`) with real `lr`/`sr` instructions, and override `__muldi3`,
   `__udivsi3`, and `__divsi3` globally (compiled without `-flto`) using hardware
   `mpymu` and `divu` instructions.

## License

Copyright (C) 2026 Thalles Campagnani

This project is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later
version. It is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See [LICENSE](LICENSE) for details.

Firmware source files intended for integration with U-Boot carry
`SPDX-License-Identifier: GPL-2.0-or-later` where applicable so they can be
distributed with U-Boot (GPL-2.0). Documentation and standalone tools are
licensed under **GPL-3.0-or-later**.

This project is not affiliated with or endorsed by SpacemiT, Synopsys, or
Micron; product names and part numbers belong to their respective owners.
Programming DRAM controllers and PHYs is hardware-specific — use at your own
risk.

