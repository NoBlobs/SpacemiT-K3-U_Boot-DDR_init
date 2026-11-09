# Flash, build, and recovery

## Scope and safety

This chapter documents the build, flashing, and recovery procedures for testing first-stage bootloader (SPL/FSBL) and PMU firmware images on the SpacemiT K3 Pico-ITX platform.

Before modifying SPI NOR flash:

- Maintain a verified full SPI NOR backup (`bootinfo`, `fsbl`, `env`, and the persisted `DDRT` training envelope at `bootinfo+0x10000`).
- Record exact artifact SHA-256 hashes, compiler versions, build flags, and configuration parameters for every test run.
- Connect a 115200 8N1 serial console and ensure physical access to disconnect and reconnect board power.
- **Preserve the UEFI/EDK2 partition**: on K3 boards using a UEFI boot flow, the SPI partition named `uboot` stores the EDK2 firmware volume (`fit`), not a standalone U-Boot payload. Overwriting `uboot` with `u-boot.itb` breaks the UEFI boot chain.
- Ensure a working USB Firmware Download (**FDL** / fastboot) recovery path before erasing or replacing `fsbl`.

> **DANGER — Never access PHY/PMU MMIO from Linux (`/dev/mem`):**
> Reading or writing DPHY/PMU MMIO registers after the OS boots stalls the SoC interconnect and requires a physical power cycle. Perform all PHY/PMU inspection from the bootloader while the PMU is powered, or via JTAG.

---

## Bootloader chains: unsigned source build vs. signed prebuilt FSBL

Two distinct FSBL chains exist for the K3 and must not be confused:

### 1. Unsigned source-built SPL/FSBL chain

When built from open-source bootloader sources (`k3_defconfig`), the resulting `FSBL.bin` is **unsigned**. The K3 BootROM accepts this unsigned image format and executes it directly. This unsigned chain is the required vehicle for testing custom DDR initialization logic or independent PMU firmware images.

### 2. Signed prebuilt factory FSBL

Prebuilt factory `FSBL.bin` images carry a cryptographic signature verified by the BootROM. Modifying any byte inside a signed prebuilt `FSBL.bin` (such as replacing or padding an embedded PMU firmware payload, even if compressed and decompressed sizes match) invalidates the signature and causes the BootROM to reject the image during boot. Keep an unmodified copy of the known-good prebuilt `FSBL.bin` strictly as a recovery artifact.

---

## Building the unsigned SPL/FSBL chain

From the bootloader source tree root, build in-tree using a 64-bit RISC-V GNU toolchain:

```sh
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- k3_defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j"$(nproc)"
```

Key generated artifacts:

- `FSBL.bin` — first-stage bootloader containing SPL and embedded DDR/PMU initialization payloads;
- `bootinfo_spinor.bin` — SPI NOR boot header;
- `u-boot-env-default.bin` — default environment image;
- `u-boot.itb` — later-stage FIT image (do **not** flash over the EDK2 `uboot` partition on UEFI systems).

---

## Building independent ARC/PMU firmware safely

When compiling independent PMU firmware for the embedded ARCv2 EM core, enforce all constraints specified in [12-arc-execution-environment.md](12-arc-execution-environment.md) and [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md):

1. **Target and reduced register file**: Compile and link with `-mcpu=em -mrf16 -mlittle-endian`.
2. **Pass `-mrf16` at both compile and link stages**: When `-flto` is enabled, LTRANS regenerates machine code at link time and will emit illegal `r16`–`r25` instructions unless `-mrf16` is present on the linker command line.
3. **Startup and linker layout**:
   - Place the reset branch at `0x0` targeting effective startup at **`0x1fc`**.
   - Set initial stack top to `0x80010000`.
   - Keep `.bss` at **`0x80004000`** so startup zeroing never overwrites host-supplied `phypara` data at `0x80000400` (including ODT settings at `0x80000424`).
4. **Hardware arithmetic libcall overrides**: Compile the translation unit that overrides `__muldi3`, `__udivsi3`, and `__divsi3` (using hardware `mpymu`/`mpyu`/`divu` instructions) **without `-flto`** so LTO cannot discard the global replacements.
5. **Post-link verification**: Audit the linked ELF before packaging:
   ```sh
   arc-elf32-objdump -d firmware.elf | grep -E '\br(1[6-9]|2[0-5])\b'
   ```
   This check must return zero instruction matches. Also confirm the binary fits within the 64 KiB IMEM window (`<= 0x10000` bytes for full training, `<= 0x6800` bytes for quickboot decompression).

Example toolchain invocation pattern:

```sh
ARC_CC=arc-elf32-gcc

# Compile firmware sources with -mrf16 (and optional -flto)
"$ARC_CC" -std=gnu23 -mcpu=em -mrf16 -mlittle-endian -Os -flto \
  -ffreestanding -fno-builtin -ffunction-sections -fdata-sections \
  -c firmware_main.c -o firmware_main.o

# Compile global arithmetic libcall overrides WITHOUT -flto
"$ARC_CC" -std=gnu23 -mcpu=em -mrf16 -mlittle-endian -Os \
  -ffreestanding -fno-builtin \
  -c pmu_libcalls.c -o pmu_libcalls.o

# Link with -mrf16 explicitly passed to the linker driver
"$ARC_CC" -mcpu=em -mrf16 -flto -nostdlib -static \
  -Wl,--gc-sections -Wl,--no-warn-mismatch \
  -Wl,-T,pmu_firmware.ld -o firmware.elf startup.o firmware_main.o pmu_libcalls.o -lgcc
```

---

## SPI NOR flash layout and update procedure

On the reference board, the SPI NOR flash partitions used during early boot are:

| Partition | SPI Offset | Role |
|---|---:|---|
| `bootinfo` | `0x00000` | BootROM header; stores the `0x9800`-byte `DDRT` envelope at offset `+0x10000` |
| `fsbl` | `0x20000` | First-stage bootloader (`FSBL.bin`) |
| `env` | `0xa0000` | Bootloader environment |

Always verify partition names via `/proc/mtd` rather than assuming fixed `/dev/mtdX` indices across different kernel configurations.

### Flashing `FSBL.bin` from Linux

When updating `fsbl` from a running Linux system:

```sh
flash_erase /dev/mtdX 0 0
mtd_debug write /dev/mtdX 0 <fsbl-byte-length> FSBL.bin
```

- Flash `FSBL.bin` only to the `fsbl` partition.
- Do not touch `bootinfo` unless intentionally invalidating or restoring the `DDRT` training envelope at `bootinfo+0x10000`.
- Erasing the first 4 KiB sector at `bootinfo+0x10000` invalidates `magic` (`0x54524444`) and forces full training on the next boot.

---

## FDL staging and recovery procedures

### Non-destructive RAM staging via FDL (recommended for testing)

When the `fsbl` partition in SPI NOR is erased, the BootROM automatically enters USB Firmware Download (**FDL**) mode on power-up. This allows staging and executing a candidate `FSBL.bin` directly in RAM without wearing or risking SPI flash:

```sh
fastboot stage FSBL.bin
fastboot continue
```

*(Select the specific USB device path with `-s usb:...` if multiple fastboot devices are attached.)*

### Recovering from a PMU or bus hang

If a candidate firmware image hangs the PMU or stalls the DPHY/APB bus:

1. **Power-cycle the board**: After a PMU hang, the USB DFU/FDL gadget and internal PHY state remain wedged; toggling UART DTR/RTS or issuing a warm reset does not recover the core. Physically disconnect power, wait for capacitors to discharge, and reconnect power.
2. **Enter FDL**: With `fsbl` erased (or FDL straps active), allow the board to enumerate in fastboot/FDL mode.
3. **Stage a known-good image**:
   ```sh
   fastboot stage known_good_FSBL.bin
   fastboot continue
   ```
4. **Restore SPI flash**: Note that in the standard BootROM/U-Boot recovery environment, `fastboot flash fsbl` is a no-op for SPI NOR. To write SPI NOR permanently during recovery, either boot into Linux via a staged `FSBL.bin` and use `flash_erase`/`mtd_debug`, or stage U-Boot and use U-Boot `sf` / `mtd` commands against the staged download buffer.

---

## Post-test validation and rollback checklist

After every hardware test run:

1. Capture the complete 115200 8N1 serial log, recording the loader banner, training mode (full training vs. quickboot), reported execution time, and detected DRAM capacity/rate.
2. Verify that no mailbox timeout or PMU error (`0xff`) occurred.
3. Record whether the post-activation host memory smoke test was enabled and passed.
4. When testing is complete, restore the known-good `FSBL.bin` and valid `DDRT` envelope in `bootinfo+0x10000`, perform a cold boot, and confirm that quickboot completes in **~9 ms**.

## Related chapters

- [10-persistence-and-selection.md](10-persistence-and-selection.md)
- [12-arc-execution-environment.md](12-arc-execution-environment.md)
- [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
