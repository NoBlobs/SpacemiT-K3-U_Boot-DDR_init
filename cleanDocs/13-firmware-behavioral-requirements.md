# Firmware behavioral requirements

## Purpose and scope

These normative requirements specify the processor, memory, peripheral, timing, and toolchain rules that any LPDDR5 PMU firmware implementation must satisfy to execute reliably on the SpacemiT K3 DDR PHY microcontroller. They apply to the ARC/PMU firmware image, not to host-side DDR controller programming.

In this document, **must** denotes a mandatory hardware or ABI requirement verified on the K3 platform. All addresses are in the PMU/ARC address space unless explicitly identified as a host CSR index.

## BR-01 — Target architecture, entry point, and reduced register file

- The target processor is a little-endian **ARCv2/EM** core.
- The image reset vector at offset `0x0` must branch to the effective startup entry at **`0x1fc`**. Hardware execution requires the `0x1fc` entry layout; linking a flat entry directly at `0x0` without the `0x1fc` startup sequence does not complete initialization.
- Startup must establish the stack top at **`sp = 0x80010000`**. For full-training firmware, the host-provided parameter table base is **`0x80000400`**.
- The K3 PMU core implements the 16-register reduced register file (`RF16`) and **does not implement general registers `r16`–`r25`**. Generated machine code must never read or write `r16`–`r25`.
- Both compilation and final linking must pass **`-mrf16`**. When Link-Time Optimization (`-flto`) is used, passing `-mrf16` only during compilation is insufficient because the linker (`LTRANS`) regenerates code using the full 32-register file unless `-mrf16` is also present on the link command line.
- If the compiler uses `gp` (`r26`) for its own small-data section, that pointer must be kept separate from the `0x80000400` parameter-table pointer so writes are never redirected into IMEM or invalid regions.

## BR-02 — Startup memory placement and `.bss` isolation

- Firmware startup must initialize only memory regions owned by the firmware's own static variables. It must never clear host-supplied input state.
- Firmware `.bss` **must not overlap `0x80000400`**. That region holds the `phypara` and electrical configuration written by the host SPL (including ODT parameters near `0x80000424`). Zeroing `0x80000400` during startup erases the input configuration and causes training to abort.
- Place firmware `.bss` and internal uninitialized globals at **`0x80004000`** (or higher unused DMEM below the stack and message scratch area).
- The compiled binary must fit both the 64 KiB PMU IMEM (`0x50000`) and the host loader's decompression buffer limit for its target boot mode (`0x10000` bytes for full training, `0x6800` bytes for quickboot).

## BR-03 — Peripheral/DMEM access widths and unaligned table strides

- Every DMEM (`0x8000xxxx`) and PMU/PHY peripheral CSR (`0x90xxxxxx`) access must use the exact hardware width required by the register:
  - Standard PHY CSRs, mailbox registers, and 16-bit DMEM fields must be accessed with **16-bit halfword instructions (`ldh`/`sth`, `volatile uint16_t`)**. Using 32-bit accesses (`ld`/`st`, `uint32_t`) on `0x8000xxxx` or `0x90xxxxxx` reads zero, clobbers adjacent 16-bit registers, or stalls the internal peripheral bus.
  - Single-byte message-block fields must be accessed with **8-bit byte instructions (`ldb`/`stb`, `volatile uint8_t`)** respecting little-endian byte order within the containing halfword.
- Packed tables whose element stride is **not a multiple of 4 bytes** (for example, 6-byte descriptor entries) must be accessed **byte-by-byte (`ldb`, `uint8_t *`)** and assembled explicitly in registers. A 32-bit dereference at an unaligned offset such as `base + 6` faults and hangs the ARC core.
- Pointer arithmetic and value extensions must preserve the exact byte stride and signed/unsigned semantics of each field.

## BR-04 — Register ABI and dispatch state preservation

- Standard function calls pass arguments in **`r0`–`r7`** and return results in **`r0`**, saving return addresses in **`blink` (`r31`)**.
- All helper routines (such as delay calculation and timer wait wrappers) must receive their full input parameters and forward computed values without truncation or zero-substitution.
- Where state machines or configuration tables use indexed inline branches (`bi`/`bih`), dispatch targets that share state with the caller must receive that context explicitly (for example via a context structure in C) rather than assuming isolated zeroed local state.

## BR-05 — Control flow and routine boundaries

- Every routine return—including delayed returns (`j_s.d [blink]`) and their delay-slot instruction—must cleanly terminate execution of that routine without falling through into adjacent routines.
- Conditional calibration and DQ-training branches must evaluate their exact hardware enable flags (`SequenceCtrl`, `CsPresent`, `X8Mode`, and stage status); unconditionally skipping a calibration branch leaves the PHY uncalibrated even if the firmware still emits a completion code.

## BR-06 — Delay slots and hardware loop counts

- Any assembly or low-level loop using ARC delayed branches (`dbnz.d`, `b.d`, `j_s.d`) must account for the instruction in the delay slot executing on the final branch taken.
- Specifically, a `dbnz.d` loop of `N` iterations with a register store in its delay slot (such as initialization loops writing `0x90020146` and `0x900201b2`) must execute the store **`N` times**, including on the final iteration—never `N - 1` times.
- Zero-overhead hardware loops (`lp_count`) and software polling loops must execute their full required iteration count across all active channels, ranks, and lanes.

## BR-07 — Scaled arithmetic and address calculation

- ARCv2 scaled arithmetic instructions (`add1`, `add2`, `add3` and `sub1`, `sub2`, `sub3`) multiply their shifted operand by **2, 4, and 8** respectively before addition or subtraction.
- Address stride calculations must preserve this scaling: for example, a base advance equivalent to `add3 r, r, 1024` represents a byte stride of **`1024 * 8 = 0x2000`**, not `0x400`.
- Lane and rank CSR addresses must be formed by combining the aligned family base constant with channel and rank selector bits using **bitwise OR (`|`)**, never addition (`+`).

## BR-08 — Valid PHY/PMU register targets

- All peripheral CSR writes must target valid, aligned addresses in the `0x90xxxxxx` space. In particular, PHY control block writes in the `0x900400xx` range must target the exact hardware offsets (such as `0x900400a6`, `0x900400e0`, and `0x900400e2`, rather than displaced offsets like `0x90040186`, `0x900401c0`, or `0x900401c2`).
- Local stack frame pointers (`fp`/`sp` in `0x8000xxxx`) must never be miscalculated or aliased into `0x9002xxxx`/`0x9003xxxx` peripheral addresses; writing to unmapped APB/CSR offsets stalls the PHY bus.

## BR-09 — Hardware RTC timer accesses and wrap handling

- The PMU hardware timer resides in ARC auxiliary registers **`0x103` (RTC control)**, **`0x104` (RTC count low)**, and **`0x105` (RTC count high)**.
- Firmware must access `0x103`–`0x105` exclusively with ARC **`lr` and `sr` instructions** (`__builtin_arc_lr` / `__builtin_arc_sr` or inline assembly). Treating these numbers as memory addresses causes timer polling loops to hang forever.
- To time a delay:
  1. Check AUX `0x103`; if the counter is not running, initialize and enable it (`sr 2, [0x103]` followed by `sr 1, [0x103]`).
  2. For scaled delays below 16 units, execute the short-delay register access (`0x90040048`) and return immediately.
  3. Otherwise, sample the 64-bit counter (`lr` from `0x104` and `0x105`), compute the 64-bit expiration target, and poll until the counter reaches the target.
  4. When polling across 32-bit boundary wraps, compare the newly sampled **`0x105` (count-high)** against the **previously sampled count-high** (never against count-low) to detect rollover accurately.

## BR-10 — Hardware multiply/divide instructions and `libgcc` overrides

- Standard `arc-elf32` `libgcc` implementations of 64-bit multiplication (`__muldi3` / `__umulsi3_highpart`) and 32-bit integer division (`__udivsi3` / `__divsi3` / `__udivmodsi4`) hang on the K3 PMU core.
- Firmware must use the hardware **`mpymu`** (unsigned 32×32 high-part multiply), **`mpyu`/`mpy`**, and **`divu`/`divs`** instructions:
  - Upper 32-bit products (`(uint64_t)a * b >> 32`) must emit `mpymu`.
  - Unsigned and signed 32-bit divisions must emit `divu` and `divs`.
- To prevent the compiler from emitting calls to broken `libgcc` helpers anywhere in the image, override **`__muldi3`, `__udivsi3`, and `__divsi3` globally** using `mpymu`/`mpyu`/`divu`.
- Compile the translation unit containing these global libcall overrides **without `-flto`** (while keeping `-mrf16`) so that LTO linking cannot discard or inline-bypass the overrides.
- Inspect the final linked ELF disassembly to verify that no `libgcc` `__umulsi3_highpart` or `__udivmodsi4` bodies remain and no `r16`–`r25` registers appear.

## BR-11 — Message scratch vs. persistent diagnostic regions

- The DMEM region **`0x8000e000`–`0x8000e0ff`** is the transient firmware-to-host message area. Because the host SPL reads and zeroes this area during execution, firmware must not store persistent diagnostic state there.
- If optional diagnostic progress markers are enabled during bring-up, place them in dedicated 32-bit slots at **`0x8000e100 + 4 * slot`** (for example, slot 120 at `0x8000e2e0`, readable from the host at DPHY index `0x5f170`).
- Any diagnostic instrumentation must be non-overlapping with `phypara` (`0x80000400`), `.bss` (`0x80004000`), the stack (`0x80010000` downward), and the result message block (`0x80000000`–`0x8000013f`), and must be compiled out of production builds so training timing and mailbox traffic are unaltered.

## BR-12 — Mailbox handshake and 16-bit protocol compliance

- All PMU accesses to the mailbox registers—status **`0x90180008`**, low data/command **`0x90180064`**, doorbell/ACK **`0x90180066`**, and high data **`0x90180068`**—must use **16-bit halfword operations (`ldh`/`sth`)**.
- Sending a major message requires:
  1. Polling `0x90180008` bit 0 until `0` (mailbox free);
  2. Writing the 16-bit code to `0x90180064`;
  3. Writing `1` to `0x90180066` (doorbell);
  4. Polling `0x90180008` bit 0 until `1` (host acknowledged), then clearing `0x90180066` to `0`.
- Sending a streaming message (`0x08`) requires first sending command `0x08` with its handshake, then sending the 32-bit header `(id << 16) | len` and each of the `len` 32-bit payload words as `(low_16 -> 0x90180064, high_16 -> 0x90180068)` with a full doorbell/ACK handshake per word.

## BR-13 — Verification invariants before hardware deployment

Before loading or flashing a newly built PMU firmware image, verify:

1. **Disassembly audit**: `arc-elf32-objdump -d` shows zero references to `r16`–`r25` and no unoverridden `libgcc` division/64-bit multiplication routines.
2. **Entry layout**: offset `0x0` branches to startup at `0x1fc`.
3. **Section placement**: `.bss` starts at or above `0x80004000` and does not touch `0x80000000`–`0x80003fff`.
4. **Image size**: binary size is within the 64 KiB IMEM limit (and `<= 0x6800` bytes if targeting quickboot).
5. **Runtime completeness**: full training executes all enabled `SequenceCtrl` stages, emits major `0x07`, leaves `CsTestFail == 0`, populates all 5170 `phypara` halfwords and 4096 ACSM halfwords, and passes post-activation memory verification.

## Related chapters

- [02-addressing-and-access.md](02-addressing-and-access.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [12-arc-execution-environment.md](12-arc-execution-environment.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
- [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md)
