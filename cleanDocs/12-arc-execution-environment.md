# ARC execution environment

## Scope and evidence

The LPDDR5 training and quickboot programs execute on the PHY's embedded **ARCv2 EM PMU**, not on the host RISC-V CPU. This document specifies the hardware execution environment, memory map, register file constraints, and ABI expectations required to build and run independent PMU firmware.

Facts confirmed by instruction-level behavior and hardware experiments are marked **observed**. Design guidance is marked **inferred**, and unresolved behavior is listed separately. No vendor implementation code is reproduced here.

## Observed: processor and image model

- The core is a 32-bit, little-endian **ARCv2 EM** processor supporting 16-bit and 32-bit instruction encodings, two zero-overhead loop registers (`lp_count` with start/end bounds), and decrement-and-branch (`dbnz`).
- It has no MMU or FPU in this environment; all addresses are physical in the PMU's local address space.
- The host loads the firmware image into PMU IMEM at PHY CSR index `0x50000`, transferred as 16-bit halfwords.
- Hardware probing demonstrated a complete **64 KiB IMEM** window: instructions at offsets up to `0xfff8` executed successfully. Note that in the quickboot loader path, the host decompression buffer is bounded to `0x6800` bytes (26,624 bytes), whereas the full-training loader path accommodates up to 64 KiB.
- The host releases the PMU using the `MICRO_RESET = 9, 1, 0` sequence (`0xd0099`) and communicates exclusively through shared DMEM and the mailbox registers. There is no register-based parameter passing between host and PMU at reset.
- The initial stack top is **`0x80010000`**, and the stack grows downward.

The host's PHY CSR indexing convention is distinct from ARC addresses: a host `phy_reg[index]` accesses a 16-bit CSR through a 32-bit bus word at `DPHY_BASE + 4 * index`. ARC DMEM addresses (`0x8000xxxx`) and peripheral addresses (`0x90xxxxxx`) are in the PMU's native byte-addressed space, not host CSR indices.

## Observed: effective entry point is `0x1fc`

The image begins with a reset branch at offset `0x0`, but the effective startup entry is **`0x1fc`**. The reset vector at `0x0` branches to `0x1fc`; normal execution must not be linked as a flat function body starting directly at `0x0` without preserving the `0x1fc` entry layout.

The training-firmware startup at `0x1fc` initializes core state, sets `sp = 0x80010000` and the parameter global pointer (`gp = 0x80000400`), and invokes the main training flow. Hardware injection experiments confirmed that intercepting execution at `0x1fc` (and optionally resuming at `0x200`) works deterministically, whereas redirecting the reset vector at `0x0` to an arbitrary offset did not complete normal initialization. An independent image should therefore keep a reset branch at `0x0` targeting startup code aligned at `0x1fc`.

## Observed: reduced register file (`-mrf16`)

This K3 PMU core implements the **16-entry reduced ARCv2 register file** (`RF16`) and **does not implement general-purpose registers `r16` through `r25`**. The architectural register set available to firmware is:

- `r0`–`r15` (general-purpose registers, with `r0`–`r7` used for argument passing and `r0` for return values);
- `gp` (`r26`, small-data / global pointer);
- `fp` (`r27`, frame pointer);
- `sp` (`r28`, stack pointer);
- `r30`; and
- `blink` (`r31`, branch link return address).

Executing any instruction that reads or writes `r16`–`r25` (for example a compiler-generated callee-save push such as `st.aw r25, [sp, -4]`) immediately faults or hangs the PMU core.

Consequently, **`-mrf16` is mandatory on every compiler invocation and on the final linker invocation**:

- When Link-Time Optimization (`-flto`) is enabled, the linker invokes `LTRANS` to regenerate machine code across translation units. Passing `-mrf16` only during compilation (`-c`) is insufficient: if `-mrf16` is omitted on the link command line, `LTRANS` reverts to the 32-register ABI and emits `r16`–`r25` instructions.
- Every linked ELF image must be verified with `arc-elf32-objdump -d` to confirm zero instructions reference `r16`–`r25`.

## Observed: indirect jump tables and state inheritance

The reference firmware uses indexed inline branch instructions (`bi` for word-scaled jumps and `bih` for halfword-scaled jumps) to dispatch state machines and configuration handlers. Four primary dispatch tables are observed:

| Dispatch form | Entries | Index step (bytes) | Behavioral role |
|---|---:|---:|---|
| Word-indexed (`bi`) | 48 | 4 | Training stage/sequence selector (indices `0x00`..`0x2f`) |
| Halfword-indexed (`bih`) | 9 | 2 | Operating mode selector |
| Halfword-indexed (`bih`) | 7 | 2 | Configuration variant selector |
| Halfword-indexed (`bih`) | 20 | 2 | Training state-machine step selector (indices `0x00`..`0x13`) |

Targets of these indexed branches are often **chained basic blocks, not isolated ABI-compliant functions**: they inherit live values in registers from the dispatcher. An independent C implementation should use explicit `switch` statements or pass an explicit context structure rather than assuming each dispatch target has independent function scope.

## Observed: `phypara` base (`0x80000400`) vs. firmware `.bss` (`0x80004000`)

Before releasing the PMU, the host loader populates the message block (`0x80000000`–`0x8000013f`) and pre-training PHY parameter structures starting at **`0x80000400`** (including ODT configuration near `0x80000424`). The reference training firmware sets `gp = 0x80000400` to address this host-provided parameter block.

This region is **live input state**, not zero-initialized scratch memory:

| Region / Purpose | ARC Address | Contract |
|---|---|---|
| Message block | `0x80000000`–`0x8000013f` | Host input parameters and PMU output fields |
| Host-provided `phypara` / parameter area | **`0x80000400`** | Live configuration written by SPL; **must not be zeroed** by startup |
| Mode-register shadow / configuration data | `0x800009a8` | Consumed during initialization and training |
| Independent firmware `.bss` / globals | **`0x80004000`** | Safe region for zero-initialized compiler `.bss` and static state |
| Firmware-to-host message scratch | `0x8000e000`–`0x8000e0ff` | Read and cleared by host SPL; not persistent across phases |
| Diagnostic progress marker slots | `0x8000e100 + 4 * v` | Persistent 32-bit diagnostic slots (e.g. slot 120 at `0x8000e2e0`, host CSR `0x5f170`) |
| Initial stack top | `0x80010000` | Grows downward |

If a C startup routine places `.bss` at `0x80000400` and zeros it on boot, it wipes the SPL-supplied `phypara` and ODT tables before training begins, causing immediate training failure. Furthermore, if the compiler uses the hardware `gp` register (`r26`) for its own small-data section (`.sdata`/`.sbss`), that compiler `gp` must not be confused with the `0x80000400` parameter base.

*(Note: the quickboot firmware image uses a different internal layout with `gp = 0x80005400`; memory-layout constants must match the specific image role.)*

## Observed: access widths and unaligned table strides

The PMU's DMEM (`0x8000xxxx`) and PHY/peripheral CSR space (`0x90xxxxxx`, including mailbox registers `0x90180008`, `0x90180064`, `0x90180066`, and `0x90180068`) are **16-bit interfaces** on the ARC side.

### ARC CSR address formation

When the PMU accesses a PHY CSR index from the ARC core, the 16-bit halfword address in the `0x90xxxxxx` space is formed by doubling the 16-bit CSR index (combined with any block/channel/rank selector bits via bitwise OR):

```text
ARC_CSR_address = 0x90000000 | (2 * (csr_index | block_or_rank_bits))
```

In addition, internal descriptor-transfer routines use `0x9001e000 + 2 * index` for write-side descriptor transfers, `0x90000000 + 2 * index` for read-side descriptor transfers, and `0x90082000 + 2 * index` for packed command halfwords.

### Observed PMU-side CSR and DMEM landmarks

| ARC Address / Range | Observed Role |
|---|---|
| `0x80000458`, `0x8000045e` | Pointers/counters for command serialization into `0x90082000` |
| `0x800009a0` | Initial configuration block copied to the stack at startup |
| `0x800009a8` | Mode-register shadow/configuration table (stride `0x36` per P-state/rank, `0x6c` per duplicated set) |
| `0x80000b6c` | Logical rank / P-state / channel state variables |
| `0x80000ba4` | Per-rank 8-value result staging buffer |
| `0x80000da0`, `0x80000db0` | Working command buffers prior to `0x90082000` serialization |
| `0x8000631c` | Sweep/training working buffer (`0x8480` bytes) |
| `0x90020020`/`24`/`32`/`38`/`d0`/`d4`/`d8`, `0x90020100` | Per-rank/lane trained delay and result registers |
| `0x90020146`, `0x90020158`, `0x9002016e`, `0x900201b2` | Per-lane training control and read/write eye CSRs |
| `0x9003e000`, `0x9003e034`, `0x9003e126` | Hardware training engine control (start/stop, mask, mode, reset) |
| `0x90040040`/`42`/`44`/`46`/`48`/`4e`/`62` | PHY engine control, P-state parameters (`+ 2 * pstate`), status, and short-delay read (`0x90040048`) |
| `0x900400a6`, `0x900400e0`, `0x900400e2` | PHY configuration registers (must not be displaced by `+0xe0`) |
| `0x900400e4`/`e6`/`f4`/`fc` | Active rank / P-state / channel selection |
| `0x90040120`, `0x90040146`, `0x90040154` | Initialization and control pulse registers |
| `0x90040238` | Terminal status pulse register (`1 -> 0` on normal completion `0x07`; `4 -> 0` on fatal error `0xff`) |
| `0x9005e012`, `0x9005e0a4`, `0x9005e140` | Training finalization, synchronization, and delayed pulse |
| `0x90060002`/`04`, `0x90061002`, `0x9007e002`/`04`/`34` | Per-lane CA/CS and training result/control registers |

### Access width rules

- Standard CSR and DMEM word accesses must use **16-bit halfword loads and stores (`ldh`/`sth` / `volatile uint16_t`)**. Accessing `0x8000xxxx` or `0x90xxxxxx` as 32-bit (`ld`/`st` / `uint32_t`) reads zero, corrupts adjacent halfwords, or stalls the internal peripheral bus.
- Individual 8-bit message-block fields are accessed via byte loads/stores (`ldb`/`stb` / `volatile uint8_t`) at their exact little-endian byte offset.
- Packed parameter tables whose element stride is **not a multiple of 4 bytes** (for example, 6-byte descriptor records) must be read **byte-by-byte (`ldb` / `uint8_t *`)** and assembled in registers. Performing a 32-bit load on the second element (`base + 6`) causes an unaligned memory access fault/hang on the PMU core.

## Observed: auxiliary registers (`RTC` timer and exceptions)

The PMU uses ARCv2 auxiliary registers accessed exclusively via **`lr` (load from auxiliary register) and `sr` (store to auxiliary register)** instructions—never via memory-mapped loads/stores:

| AUX Address | Register Role | Usage |
|---:|---|---|
| `0x103` | RTC control | `lr`/`sr`; enables/resets the free-running 64-bit hardware counter (writes `2`, then `1`) |
| `0x104` | RTC count low (32-bit) | `lr`; lower 32 bits of the free-running timer |
| `0x105` | RTC count high (32-bit) | `lr`; upper 32 bits of the free-running timer |
| `0x400` | `ERET` (Exception Return) | `lr`; faulting PC saved by the low-memory exception handler |
| `0x403` | `ECR` (Exception Cause) | `lr`; exception cause code saved by the exception handler |

### Delay implementation

Firmware delays check AUX `0x103` (starting the counter if stopped), sample the 64-bit counter across `0x104` (low) and `0x105` (high), and poll until the target delta has elapsed while properly handling 32-bit low-word wraparound into the high word. Very short scaled delays (below 16 units) take a fast path that touches `0x90040048` and returns immediately without entering the RTC polling loop. Modeling `0x103`–`0x105` as ordinary memory locations causes every delay loop to hang indefinitely.

### Low-memory exception dump

An exception handler at the bottom of the image captures post-mortem state into DMEM `0x80000000`–`0x8000001c` before halting:

| ARC Address | Stored Value |
|---|---|
| `0x80000000` | Sentinel `0x1234dead` |
| `0x80000004` | `ERET` (AUX `0x400`) |
| `0x80000008` | `ECR` (AUX `0x403`) |
| `0x8000000c` | `r0` |
| `0x80000010` | `r1` |
| `0x80000014` | `sp` |
| `0x80000018` | `fp` |
| `0x8000001c` | Sentinel `0xabcddead` |

## Observed: arithmetic instructions and `libgcc` constraints

Several software arithmetic helpers in standard full-register `arc-elf32` `libgcc` fail or hang on the K3 PMU core:

1. **64-bit multiplication (`__muldi3` / `__umulsi3_highpart`)**: Standard `libgcc` implementations using `rlc`/`adc` sequences hang the PMU core when computing 64-bit products or upper 32-bit products (`(uint64_t)a * b >> 32`). The hardware implements the unsigned 32×32 high-part multiply instruction **`mpymu`** (and `mpy`/`mpyu`). Firmware must use hardware `mpymu` (or provide a global `__muldi3` override built around `mpymu`/`mpyu`).
2. **32-bit integer division (`__udivsi3` / `__divsi3` / `__udivmodsi4`)**: Standard `libgcc` division hangs on the small-divisor fast path (when `divisor >= dividend / 2`, such as `5 / 4`). The hardware implements **`divu`** (and `divs`). Firmware must use hardware `divu`/`divs` instructions directly or override `__udivsi3` and `__divsi3` globally using hardware division.
3. **Global override preservation under LTO**: When compiling firmware with `-flto`, the translation unit providing global `__muldi3`, `__udivsi3`, and `__divsi3` replacements must be compiled **without `-flto`** (while still passing `-mrf16`) so that `LTRANS` does not discard the custom symbols and fall back to the broken `libgcc` archive implementations.

## Observed: delay-slot and scaled-addressing semantics

When writing assembly or auditing generated code for the ARCv2 EM core:

- **Branch/loop delay slots (`.d`)**: An instruction in the delay slot of `dbnz.d` or `j_s.d [blink]` executes unconditionally on the final branch/iteration as well. In a `dbnz.d` loop of `N` iterations with a store in the delay slot (for example to `0x90020146` or `0x900201b2`), the store executes **`N` times**, not `N - 1`.
- **Scaled arithmetic (`add1`/`add2`/`add3`, `sub1`/`sub2`/`sub3`)**: These instructions shift their second source operand by 1, 2, or 3 bits (multiplying by **2, 4, or 8**) before adding/subtracting. For example, `add3 r, r, 1024` advances a pointer by `1024 * 8 = 0x2000` bytes.

## Summary of host ↔ PMU runtime contract

### What the PMU expects from the host

1. PHY ownership transferred to PMU (`MICRO_CONT_MUX_SEL = 1` at `0xd0000`).
2. Complete firmware loaded into IMEM (`0x50000`) and any required initial DMEM loaded.
3. Validated 320-byte message block at `0x58000` (`0x80000000`) and `phypara` tables at `0x80000400`.
4. Unused DMEM cleared (`0x584d2`–`0x5ffff` in full training).
5. No concurrent host CSR accesses while the PMU owns the mux.
6. Continuous mailbox consumption and acknowledgement (`0xd0004`, `0xd0031`, `0xd0032`, `0xd0034`).

### What the host expects from the PMU

1. PHY hardware version/signature verification at startup, failing early on mismatch.
2. Progress notifications and optional streaming frames (`0x08`) over the 16-bit mailbox.
3. Trained outputs (CDDs, VREFs, margins, DFE/DCA, quickboot PLL words) written to the message block and PHY/ACSM memories before completion.
4. Terminal major message `0x07` on normal completion or `0x000000ff` on fatal error, followed by a clean halt.
