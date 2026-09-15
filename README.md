# dfu-nv

**Device Firmware Upgrade (DFU)** is the USB class by which a device is sent
new firmware over the same port it uses for everything else. It is specified
in [USB DFU 1.1](https://www.usb.org/document-library/device-class-specification-dfu-11).
This package brings that class to novo-lang, together with an image format
whose header is checked before anything is erased, a second update protocol
that runs over a serial line, and a host-side plan that drives either. It is
built on [usb-nv](https://novo-lang.org/packages/usb-nv),
[crc-nv](https://novo-lang.org/packages/crc-nv) and
[frame-nv](https://novo-lang.org/packages/frame-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What DFU is

A device that supports DFU exposes a USB interface of class 0xFE, subclass 1.
In its ordinary running mode it answers one request, DETACH, which asks it to
restart into its bootloader. In bootloader mode it answers all seven.

The transfer is a sequence of numbered blocks. The host sends a DNLOAD
request carrying one block, then sends GETSTATUS and reads the reply. The
reply carries a status code, the state the device is now in, and
**`bwPollTimeout`**, which is how many milliseconds to wait before asking
again. The device is erasing or programming flash during that window.

That one number is the whole of DFU's back-pressure. The class has no flow
control, no window and no acknowledgement of a write. A host that ignores
`bwPollTimeout` and sends the next block writes into a device that is still
busy with the last one.

A **state** is where the device is in the transfer. DFU 1.1 section 6.1.2
defines eleven of them, from idle in the application through the download and
manifestation phases to the error state. A **status code** says why the
device is where it is; there are sixteen. The state and the status are
different things, and both come back in the same six-byte GETSTATUS reply.

**Manifestation** is the phase after the last block, in which the device
finishes writing and makes the new firmware the one it will run. A device
that is *manifestation tolerant* stays on the bus through it. One that is not
detaches, and the host has to wait for it to come back.

The serial protocol in this package is a different shape. It is
**object-based**: the host creates an object of a declared size, writes into
it, asks the device for the object's CRC, and only then executes it. A
transfer interrupted halfway therefore leaves a device that still boots, and
a SELECT request answers how much of the object already arrived, so the host
can resume rather than start again.

The **image** this package defines is a 64-byte header followed by the
payload. The header says which product and board revision the image is for,
what version it is, how long the payload is and what its CRC-32 is, where it
loads and where execution starts.

| Quantity | Value |
| --- | --- |
| DFU interface class, subclass | 0xFE, 1 |
| Interface protocol: running, bootloader | 1, 2 |
| States, class requests, status codes | 11, 7, 16 |
| Bytes in a GETSTATUS reply | 6 |
| Bytes in the DFU functional descriptor | 9 |
| Block number field | 16 bits |
| Largest image at a 64-byte block size | 4 MiB |
| Bytes in the image header | 64 |
| Bytes in the signature slot | 64 |
| Opcodes in the serial protocol | 10 |

## Install

```
novo pkg add dfu-nv
```

## Example

```novo
use dfuimg
use dfuerr

// What this device is: its product identifier and its board revision.
const MY_PRODUCT = 0x1209
const MY_BOARD = 1

fn main() [io]
    // The sixty-four byte header that arrived in front of the payload,
    // read out of wherever it landed: a control transfer, a UART, a radio.
    let staging: [u8] = [0x00 as u8, 0x00 as u8, 0x00 as u8, 0x00 as u8]
    let h = dfuimg.header_decode(staging, 0)

    // The one question, asked before anything is erased. Six conditions in
    // one call: the magic, the format version, the product, the board
    // revision, the declared length against what arrived, and the CRC-32.
    if dfuimg.can_commit(h, MY_PRODUCT, MY_BOARD, 4096, 0xCBF43926)
        println("erase and write")
    else
        // The refusal names the numbers, because a failed update reaches
        // you as a report from somebody who saw a progress bar stop.
        match dfuimg.check_header(h, MY_PRODUCT, MY_BOARD, 4096, 0xCBF43926)
            None    => println("refused, with no reason given")
            Some(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: dfu-nv.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `dfustate` | The eleven states, the seven requests, the sixteen status codes, the GETSTATUS reply as a value, the transition table, and the block arithmetic. |
| `dfuimg` | The image header, the six checks made before anything is erased, which bytes the signature covers, and the state of a transfer in progress. |
| `dfuusb` | DFU as a USB interface: the functional descriptor, the class requests as control transfers, and the bridge to `dfustate`. |
| `dfuserial` | The object protocol: ten opcodes, the request and reply values, and a transfer state machine whose resume point is a SELECT. |
| `dfuse` | ST's DfuSe extensions and the DFU file suffix, named rather than implemented. |
| `dfucrc` | The CRC-32 the image header carries, one-shot and streaming. The one module that reaches crc-nv. |
| `dfuerr` | One error type, with the two questions a caller asks of it. |
| `dfuhost` | The host side: the transport trait, a transfer plan for either wire, and reading the firmware file. |

## How to choose an entry point

**A bootloader on a device links `dfustate` and `dfuimg`, plus one of
`dfuusb` or `dfuserial`.** Add `dfucrc` when the part has no CRC peripheral.
None of those modules declares an effect.

**A host tool links `dfuhost`.** `dfuhost.plan_usb` and `plan_serial` build a
plan from an image and the device's transfer size. The plan answers which
chunk goes next, how long the transfer should take, and how long to wait.
Sending the bytes is the caller's transport, declared where that transport
is.

**Use the serial protocol over a slow or unreliable link.** It can resume.
DFU over USB numbers its blocks and has no way to ask how much arrived.

**`dfustate` names no USB type.** The same state machine drives both wires,
and `dfuusb` is the only module where the two packages meet.

## The rules a user needs

1. **After every DNLOAD the host must issue GETSTATUS, and must wait the
   number of milliseconds the reply asks for.** DFU 1.1 section 6.1.2. That
   number is the protocol's only back-pressure. A host that ignores it
   updates correctly on a slow machine and fails on a fast one, which is the
   worst kind of bug to be sent.
2. **The state machine answers the deadline; it does not sleep.**
   `dfustate.step` returns a `DfuStep` whose `wait_ms` is what to wait, and
   the caller waits. That is what lets the module run on a device with no
   clock, and it is what makes the timing testable.
3. **Check everything before erasing anything.** `dfuimg.can_commit` makes
   six checks in one call: the magic number, the format version, the product
   identifier, the board revision, the declared length against what arrived,
   and the CRC-32. After the erase there is nothing left to run and no way to
   ask for a better image. `dfuimg.check_header` is the same six with the
   reason named.
4. **Write the header last.** It is 64 bytes, which is a whole flash write
   page on nothing. On a device that stages an image, a power cut during the
   payload then leaves a slot with no valid header rather than a valid header
   over a partial image.
5. **The signature slot is a slot, and this package fills nothing.** It is
   64 bytes, the length of an Ed25519 signature, and a caller writes one with
   [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) or leaves it zero.
   What is fixed is the signed region: the header without the slot, plus the
   payload. `dfuimg.is_signed_region` and `signed_len` answer it, so two
   implementations cannot disagree about what was signed.
6. **A decoded value comes back zeroed on bad input, with a check beside
   it.** A `@value` struct may not be an optional or a `Result` payload
   (E2015), so `dfustate.status_decode`, `dfuimg.header_decode`,
   `dfuserial.req_decode` and `rsp_decode` each have a `check_*` function
   that says whether the bytes were well formed.
7. **The block number is sixteen bits, so a plain DFU transfer at a 64-byte
   block size carries at most 4 MiB.** `dfustate.max_image_len` answers the
   limit for a given block size. A device with more flash than that needs an
   address pointer, which is what DfuSe adds.
8. **A device that is not manifestation tolerant leaves the bus after the
   last block.** `dfustate.is_manifest_tolerant` reads the bit out of the
   functional descriptor, and a host that does not check it waits for a reply
   that will never come.
9. **The compressed, encrypted and delta flags exist so a bootloader can
   refuse.** Nothing here expands or decrypts an image. A bootloader that
   wrote a compressed image as if it were plain would produce a device that
   does not boot.
10. **Take the CRC as a number the caller supplies.** Every module except
    `dfucrc` does, which is what keeps them independent of crc-nv and lets a
    part with a CRC peripheral use it.
11. **The serial protocol's SELECT is the resume point.** It answers the
    object's size, how much has arrived and the CRC of what arrived. A host
    that starts every retry from zero on a slow link may never finish.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Seven of the eight modules
declare no effects and are inside it. A bootloader is the last code on a
device that cannot be replaced remotely, so its state machine and its image
parser run with no allocator, no clock and no second chance.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It covers `dfustate` and `dfuimg`.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

`dfuhost` is outside the claim. It reads a file, and one host-only function
anywhere in a compilation unit is an undefined symbol at link time on a
device, whether or not the firmware calls it. `dfucrc` is outside the probe
as well, which is why every other module takes a CRC as a number.

## What is not included

- **Cryptography.** The slot, not the signature. See rule 5.
- **Compression, encryption and delta patching.** The flags exist so an image
  can be refused. See rule 9.
- **A flash driver.** Nothing here erases or programs. That belongs with the
  board.
- **A DfuSe implementation.** `dfuse` carries the vocabulary a reader meets
  in a capture, the arithmetic to recognise a DfuSe device, and the DFU file
  suffix. No board this project supports uses it.
- **A staging buffer.** A transfer size above 64 bytes needs somewhere to
  hold a block, and that buffer is the caller's.
  [heapless-nv](https://novo-lang.org/packages/heapless-nv) is where a
  device's would come from.
- **The transport.** `dfuhost.DfuTransport[e]` is the trait a caller
  implements over its own USB or serial library, and it carries whatever
  effect that library costs.

## Related packages

- [usb-nv](https://novo-lang.org/packages/usb-nv) is the USB device side.
  `dfuusb` is written against its control-transfer and descriptor types, so
  that the two packages cannot disagree about a descriptor byte.
- [crc-nv](https://novo-lang.org/packages/crc-nv) is the CRC-32 the image
  header carries, on the IEEE 802.3 polynomial that every other firmware tool
  uses.
- [frame-nv](https://novo-lang.org/packages/frame-nv) frames the serial
  protocol's requests and replies.
- [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) is the signature
  scheme the slot is sized for.
- [flate-nv](https://novo-lang.org/packages/flate-nv) is what would expand a
  compressed image, on the host that builds it.
- `novo flash` in the toolchain is the other way of getting code onto a
  board. It drives an attached debug probe over SWD, writes any address, and
  does not care what is already there. That is the development path, for a
  board on a desk. This package is the field path: no debugger, a device
  running its own bootloader, and a transport the product has anyway.

## Tests

```bash
novo test                             # 94 tests
novo test tests/dfustate_tests.nv     # 20: the states, the transitions, the poll timeout
novo test tests/dfuimg_tests.nv       # 16: the header and the six pre-erase checks
novo test tests/dfuwire_tests.nv      # 33: the USB requests and the serial opcodes
novo test tests/dfuhost_tests.nv      # 18: the plan and the chunk arithmetic
novo test tests/dfuerr_tests.nv       #  7: which refusals are before the erase, and which retry
```

The references are USB DFU 1.1 for the class,
[dfu-util](https://dfu-util.sourceforge.net/) for the host behaviour and the
file suffix, and Nordic's serial DFU for the object protocol, which is the
transport the nRF52 boards this project supports actually use.

No test opens a device or reads a clock. Every reply is a value the test
writes out, every deadline is an argument, and a whole transfer is a list of
steps.

The tests compile today and fail at run, each on the
`not implemented: dfu-nv.<module>.<fn>` panic that is its body. That is the
expected state of an interface release. They turn green one at a time as
bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `DFU_*` and `DFUSE_*` constant | yes (they are constants) |
| `dfustate.DfuStatus`, `.DfuStep`, `dfuimg.DfuHeader`, `.DfuReceive`, `dfuusb.DfuFunctional`, `dfuserial.DfuSerialReq`, `.DfuSerialRsp`, `.DfuSerialXfer`, `dfuerr.DfuError` | declared |
| `dfuhost.DfuTransport[e]`, `.DfuPlan` | declared |
| `dfustate`: the status value, `step`, the transition table and the block arithmetic | no |
| `dfuimg`: the header, the six checks, the signed region and the receive state | no |
| `dfuusb`: the functional descriptor, the class requests and the bridge | no |
| `dfuserial`: the requests, the replies and the transfer machine | no |
| `dfuse`: the four commands, the block numbering and the file suffix | no |
| `dfucrc.crc32`, `.crc32_start`, `.crc32_step`, `.crc32_update`, `.crc32_finish` | no |
| `dfuerr.describe`, `.is_pre_erase`, `.is_retryable` | no |
| `dfuhost`: the plans, the chunk arithmetic and the file reading | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
