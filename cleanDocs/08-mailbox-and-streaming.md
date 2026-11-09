# PMU mailbox and streaming protocol

## Scope

The PMU mailbox is the control and debug channel used while the PHY PMU executes initialization/training. The host must service it continuously: the PMU waits for acknowledgements and can stall if the host stops consuming messages.

This document separates the **observed reference behavior** from **recommended behavior for a new implementation**, especially for fatal error `0xff` and return-value propagation.

## Register views

### Host PHY CSR indices

Host byte address is `DPHY_BASE + 4 * index`.

| Index | Name | Role |
|---:|---|---|
| `0xd0000` | `MICRO_CONT_MUX_SEL` | 1=PMU owns PHY; 0=host/APB access. |
| `0xd0004` | `UCT_SHADOW` | Bit 0 mailbox synchronization/status. |
| `0xd0031` | `DCT_WRITE_PROT` | Host acknowledgement sequencing. |
| `0xd0032` | `UCT_WRITE_ONLY_SHADOW` | Major code, streaming low halfword, or streaming count. |
| `0xd0034` | `UCT_DATA_WRITE_ONLY_SHADOW` | Streaming high halfword. |
| `0xd0099` | `MICRO_RESET` | Start with 9,1,0; stop/stall with 1. |
| `0x200c9` | PLL status resource | Read by the reference after major `0xff`; it is diagnostic data, not an error code. |

### PMU ARC addresses

| Address | Role |
|---:|---|
| `0x90180008` | Mailbox status, bit 0 |
| `0x90180064` | Major/command or low 16 bits of a data word |
| `0x90180066` | Doorbell/acknowledgement control |
| `0x90180068` | High 16 bits of a data word |

The host shadow registers and PMU registers are two views of the same producer/consumer mechanism, not independent mailboxes.

## Single-item handshake

Before consuming an item, the host waits for `UCT_SHADOW.bit0 == 0`.

For a normal item, acknowledgement is exactly:

1. Write 0 to `DCT_WRITE_PROT` (`0xd0031`).
2. Wait until `UCT_SHADOW.bit0 == 1`.
3. Write 1 to `DCT_WRITE_PROT`.
4. Before reading the next item, wait until `UCT_SHADOW.bit0 == 0`.

On the PMU side, the complementary behavior is to wait for status 0, publish low/high halves, signal the doorbell, and wait for the host acknowledgement. The reference polls both sides without timeouts.

Acknowledge each major item, the streaming introducer, and **every streamed 32-bit word** unless the terminal `0xff` exception below applies.

## Major messages

After the ready wait, the host reads `0xd0032` and classifies it as follows:

| Value tested | Meaning | Reference action |
|---|---|---|
| Low byte `0x07` | Normal completion | ACK and return success (0). |
| Low byte `0x08` | Streaming introducer | ACK, then read count and streamed words. |
| Full value `0x000000ff` | Fatal/error notification | Special terminal path **without ACK**; see below. |
| Any other low byte | Major stage/progress item | Report optionally, ACK, continue. |

The error test is the full value `0xff`, not merely any item whose low byte is `0xff`.

A full-training capture on two controllers produced these 22 major bytes, including the final completion byte of the first controller:

```text
0d 00 01 02 fd 06 01 02 fd 04 03 09 07
0d 00 01 02 fd 06 01 02 fd
```

The second controller's capture ended in the truncation shown; it does not establish that completion is optional. Exact meanings of stage bytes and `0xfd` are not confirmed. A consumer should handle unknown major codes as progress items, not as fatal errors solely because they are unknown.

The PMU normal-completion path emits `0x07` and then remains stopped. Its fatal path emits `0xff` and remains in a terminal break/loop. Completion and error also cause different PMU-side PHY control pulses (normal 1→0, error 4→0 at the observed control resource), reinforcing that `0xff` is terminal rather than an ordinary item requiring the host to continue.

## Streaming transport

### Wire sequence

After receiving and acknowledging major/command `0x08`:

1. Wait for ready (status bit 0 = 0).
2. Read `j = UCT_WRITE_ONLY_SHADOW` (`0xd0032`). This is the last word index, so exactly **`j + 1` 32-bit words** follow.
3. For each word, in order:
   - read low 16 bits from `0xd0032`;
   - read high 16 bits from `0xd0034`;
   - reconstruct `word = (high << 16) | low`;
   - ACK that word;
   - wait for status bit 0 to return to 0 before the next word.
4. Resume major-message processing using the next `0xd0032` value.

The `0x08` item is not itself the 32-bit header and its acknowledgement does not replace any data-word ACK.

### Message framing

The first streamed word is a header:

```text
header = (id << 16) | len
payload words = exactly len
total words after 0x08 = len + 1 = j + 1
```

The following `len` words are payload. IDs are 16 bits in this framing; observed IDs were `0x0000`–`0x02ff`, and observed payload lengths were 0–31 words. Those observed ranges are capture facts, not a new implementation's license to omit bounds checks.

Example with two payload words:

```text
major 0x08, ACK
word 0: 0x00420002    -> id=0x0042, len=2
word 1: payload[0]
word 2: payload[1]
next major item
```

### Reference capture behavior and limits

The unmodified SPL consumes framing but does not decode payloads: its translation routine is empty, normal debug logging is disabled, and its optional capture array holds only 50 words. The instrumented capture printed raw words and produced approximately 1.4 MiB of UART output, greatly slowing boot.

One two-controller full-training capture contained **9,153 streaming messages and 124 distinct IDs**. Most payload schemas are unknown. Confirmed framing must not be confused with inferred opcode meanings.

### Filtering

`HdtCtrl` is a threshold: a streamed message is emitted when its level is **less than** `HdtCtrl`. Increasing the threshold therefore suppresses lower-priority detail. Named levels are:

| Level | Class |
|---:|---|
| `0x05` | Detailed debug such as eye delays |
| `0x0a` | Coarse/rank debug |
| `0xc8` | Stage completion |
| `0xc9` | Assertions |
| `0xff` | Firmware completion only |

`MsgMisc[4]` suppresses streaming, including streamed assertions, regardless of `HdtCtrl`. Major completion/error traffic is separate. The reference board's persisted `HdtCtrl=0x04` explains the detailed capture; it is not a required production value.

## Fatal `0xff`: observed behavior

The reference SPL handles full-word `0xff` specially:

1. It does **not** call the normal ACK routine for `0xff`.
2. It writes `MICRO_RESET=1`.
3. It performs a short busy wait.
4. It selects host/APB access with `MICRO_CONT_MUX_SEL=0`.
5. It reads `0x200c9` as `plllockstatus` and logs it when logging is enabled.
6. It returns that raw PLL-status value from the mailbox routine.

This creates inconsistent propagation:

- The normal LPDDR5 caller checks whether the mailbox return value **equals `0xff`**. Since the routine returns PLL status, an actual PMU `0xff` can escape detection when PLL status is not `0xff`.
- The quickboot caller ignores the mailbox return value entirely.
- Normal completion returns 0 and is unambiguous.

Therefore, the reference's `0xff` handling is **not a dependable error-detection contract**. PLL lock status is diagnostic metadata and must not be substituted for a terminal status.

The lack of ACK is deliberate observed behavior for this terminal path, not an omission to “fix” by blindly applying the normal ACK sequence to an already terminated PMU.

## Recommended new implementation

The following policy is recommended and intentionally differs from the reference's defective propagation:

1. Represent receive outcomes explicitly, for example `PROGRESS`, `COMPLETE`, `PMU_ERROR`, `TIMEOUT`, and `PROTOCOL_ERROR`; do not encode them in raw PLL status.
2. On full-word `0xff`, latch `PMU_ERROR`, preserve the last major/streaming diagnostics and PLL status, perform bounded PMU-stop/host-mux cleanup, and propagate failure to every caller, including quickboot.
3. Do not issue the normal ACK for `0xff` when implementing the observed terminal-peer protocol. Do not wait for another item afterward.
4. ACK all normal completion, major, streaming-introducer, header, and payload items exactly once and in order.
5. Add independent bounded waits to every ready/acknowledgement poll. On timeout, stop the PMU if possible, restore host ownership, keep DDRC ports disabled, and return a distinct error.
6. Validate `len` against the consumer's buffer/stream-processing limit before reading payload. Even when payload is discarded, continue to read and ACK exactly `len` words to preserve transport synchronization; otherwise abort with an explicit protocol error and cleanup.
7. Treat unknown major IDs as progress and unknown streaming IDs as opaque framed payloads. Reserve fatal handling for `0xff`, explicit local validation failures, and timeouts.
8. Require both normal completion and valid training outputs before result publication or controller activation.

## Receiver state summary

A compatible normal-path receiver can be understood as four states:

```text
WAIT_MAJOR
  -> ACK major, WAIT_MAJOR
  -> 0x08: ACK, READ_COUNT
  -> 0x07: ACK, COMPLETE
  -> 0xff: no ACK, PMU_ERROR

READ_COUNT
  -> expect count+1 words, READ_WORD

READ_WORD
  -> read low/high, ACK, wait ready
  -> remaining words: READ_WORD
  -> frame complete: WAIT_MAJOR
```

Every wait in this state model needs a timeout in a new implementation. The terminal states must carry status independently of diagnostic register values.
