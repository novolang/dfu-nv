# dfu-nv

**Status: NOT IMPLEMENTED — interface only.**

Firmware update over USB and over serial: the USB DFU 1.1 class state
machine with its requests and status codes, a signed image format with
a CRC-32 and a signature slot the caller fills, the object-based serial
bootloader protocol over frame-nv, and a host-side updater that drives
either transport.

Every `pub fn` body is a `todo()`. The signatures and the effect rows
are published so the design can be reviewed and effect-checked before
anyone writes a body against it; `novo pkg add dfu-nv` resolves,
downloads and builds, and the first call panics with
`not implemented: dfu-nv.<module>.<fn>`.

## Build, run and test

```bash
novo pkg build          # type-check and effect-check every module; a library, so no binary
novo test               # the API suites — RED until the bodies land
novo doc .              # the reference page, with every example block compiled
```

There is nothing to run: this is a library, and at 0.0.1 every body is
a `todo()`, so `novo test` is red on purpose and the suites are the
protocol written as assertions.

## The one example that will work

A bootloader deciding whether to erase anything:

```novo
use dfuimg

// The header at the front of the image, read from wherever it arrived
// — a control transfer, a UART, a radio.
let h = dfuimg.header_decode(staging, 0)

// The one question, asked BEFORE the erase.  Six conditions: the
// magic, the format version, the product, the board revision, the
// declared length against what arrived, and the CRC-32.  A bootloader
// that checks five of them is a bootloader that bricks a device on the
// sixth, and after the erase there is nothing left to run.
if dfuimg.can_commit(h, MY_PRODUCT, MY_BOARD, arrived_len, computed_crc)
    erase_and_write(h)
else
    // And the refusal names the numbers, because a failed update is
    // reported by a user who saw a progress bar stop.
    report(dfuimg.check_header(h, MY_PRODUCT, MY_BOARD, arrived_len, computed_crc))
```

## The layer, and why

`core`, with one `host` module named in the manifest.

The plan placed the row at `host` because the updater is the visible
half. The half that has to be **right** is the other one: a bootloader
is the last code on a device that cannot be updated remotely, so its
state machine and its image parser run with no allocator, no clock and
no second chance. `docs/publishing.md` § A package with a core and a
host half is the shape — the narrow layer declared and the wide module
named — and it is what makes the device claim a thing the audit
**builds**.

- **A device links** `dfustate`, `dfuimg` and one of `dfuusb` or
  `dfuserial`, plus `dfucrc` if it has no CRC peripheral. All are `[]`
  throughout, and `tests/embedded_probe.nv` builds the first two for
  `--target=nrf52-qemu`.
- **A laptop links** `dfuhost`, the one module with a row: `[io]`, and
  it covers exactly one thing — reading the firmware file. The
  transfers are the caller's transport, declared where that transport
  is.

## The load-bearing interface

**`DfuStep`, and specifically its `wait_ms` field.**

DFU 1.1 § 6.1.2: after every DNLOAD the host **must** issue GETSTATUS,
and the reply carries `bwPollTimeout` — how many milliseconds to wait
before asking again. The device is erasing or programming flash during
that window. There is no other mechanism: DFU has no flow control, no
window, and no acknowledgement of a write. The whole protocol's
back-pressure is that one number.

A design whose "write a block" call returns `Result<(), DfuError>` has
nowhere to put it, so every implementation that starts that way ends up
with a hidden sleep inside its transport — which makes the timing
untestable and quietly ignores the device's own number. Here the
machine **answers** the deadline and the caller waits. That is also
what keeps the module `core`: a package with no clock cannot sleep.

The symptom of getting it wrong is the worst kind: a device that
updates correctly on a slow machine and bricks on a fast one.

**The second load-bearing value is `dfuimg.can_commit`.** Everything
checkable is checked before anything is erased — the magic, the format,
the product, the board revision, the length, the CRC — because after
the erase there is nothing left to run and no way to ask for a better
image. Six conditions in one call, so a bootloader cannot check five.

## The reference implementations

[`dfu-util`](https://dfu-util.sourceforge.net/) for the host half and
the file suffix, [USB DFU 1.1](https://www.usb.org/document-library/device-class-specification-dfu-11)
for the class itself, and Nordic's serial DFU for the object protocol
in `dfuserial` — the transport the nRF52 boards in `orbit/bsp`
actually use. ST's DfuSe is named rather than implemented; `dfuse` says
why.

## What is here

| Module | Layer | What it is |
| --- | --- | --- |
| `dfustate` | `core` | The ten states, the eight requests, the sixteen status codes, `DfuStatus` and `DfuStep`. |
| `dfuimg` | `core` | The image header, the six pre-erase checks, the signed region, and a partial transfer's state. |
| `dfuusb` | `core` | DFU as a USB interface: the functional descriptor, the class requests, and the bridge to `dfustate`. |
| `dfuserial` | `core` | The object protocol: ten opcodes, the request and response values, and the transfer state machine with its resume point. |
| `dfuse` | `core` | ST's DfuSe extensions and the DFU file suffix, named. |
| `dfucrc` | `core` | The one module that reaches crc-nv. |
| `dfuerr` | `core` | One error type, with `is_pre_erase` and `is_retryable`. |
| `dfuhost` | `host` | `DfuTransport[e]`, `DfuPlan`, and reading the firmware file. `[io]`. |

## What `novo flash` does today, and where this sits beside it

`novo flash` ([docs/tooling.md § novo flash](../../docs/tooling.md#novo-flash))
builds a `.nv` for an nRF52 board and hands the ELF to **probe-rs**,
which drives the on-board debug probe over SWD and writes flash
directly. That is the **development** path: it needs a debugger
attached, it writes any address, and it does not care what is already
on the device.

This package is the **field** path: no debugger, a device running its
own bootloader, an image that is checked before anything is erased, and
a transport that is already there for another reason — the USB port a
product has anyway, or the UART a gateway already speaks over.

The two do not overlap and neither replaces the other. A board on a
desk takes `novo flash`; a board in a product takes this.

## Three things the design decided, and why

**The signature slot is a slot, not a signature.** This package does no
cryptography: `signature_offset` and `DFU_SIGNATURE_LEN` say where
sixty-four bytes live, and a caller fills them with **ed25519-nv** or
leaves them zero. A signature scheme is a decision a product makes once
and cannot change — key length, key storage, revocation, what happens
on a verification failure — and a format that chose one for its users
would be a format half of them could not use. What this package **does**
fix is the one thing that must not be a decision: the signed region is
the header-without-the-slot plus the payload, so two implementations
cannot disagree about what was signed.

**The header is written last.** It is 64 bytes — a whole flash write
page on nothing, deliberately — so on a device that stages images a
power cut during the payload leaves a slot with no valid header, rather
than a valid header over a partial image.

**The serial protocol is object-based and the USB one is not**, and the
difference is worth having both. DFU over USB sends numbered blocks and
hopes; the serial protocol creates an object of a declared size, writes
into it, asks for its CRC, and only then executes it — so a transfer
interrupted halfway leaves a device that still boots, and `select`
answers how much already arrived, which makes a resume possible. On a
slow UART that is the difference between an update that completes and
one that never does.

## Depending on usb-nv

`usb-nv` is a **path dependency** while the two are developed together.
`novo pkg publish` refuses one — `--dry-run` included — and is right
to: a tarball carrying `../usb-nv` sends every consumer to a directory
that is not on their machine. The published line is
`usb-nv = "^0.0.1"`, and usb-nv has to reach the registry first.

`dfustate` deliberately names **no** usb-nv type. The state machine is
transport-independent, the serial bootloader drives the same one, and
`dfuusb` is the only place the two packages meet.

## What is deliberately outside

- **Cryptography.** The slot, not the signature. ed25519-nv is the
  package named for it.
- **Compression and encryption.** `DFU_IMG_COMPRESSED` and
  `DFU_IMG_ENCRYPTED` exist so a bootloader can **refuse** an image it
  cannot expand, rather than writing it and producing a device that
  does not boot. flate-nv is the package that would expand one.
- **A flash driver.** Nothing here erases or programs. That belongs
  with the board, over `orbit/hal`.
- **A DfuSe implementation.** Named, with the vocabulary a reader meets
  in a capture and the arithmetic to recognise a DfuSe device. No board
  this project supports uses it.

## Missing rows this package found

- **ed25519-nv is on the registry and this package does not depend on
  it.** The signature slot is deliberately empty, but nothing on the
  grid names the *pairing* — a firmware-signing tool. It is an `app`,
  not a package, and it is the obvious next thing after this.
- **A delta format.** `DFU_IMG_DELTA` is a flag with no package behind
  it: a bootloader that could apply a binary patch would move a tenth
  of the bytes over a slow link, which for a battery-powered gateway is
  the difference between a feasible update and an infeasible one.
  `bsdiff`-class algorithms are the reference, and nothing on the grid
  names one.
- **A `wTransferSize` larger than 64 needs a staging buffer**, and
  heapless-nv is where a device's would come from — but this package
  deliberately does not depend on it, because the buffer is the
  caller's. The README of a real bootloader has to say how large.

## Licence

Apache-2.0.
