# Changelog

All notable changes to dfu-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `dfustate` — the ten states, the eight requests, the sixteen status
  codes, `DfuStatus` with its twenty-four-bit poll timeout, `DfuStep`,
  the state transition table as `request_allowed`, and the block
  arithmetic including the sixteen-bit block number's limit.
- `dfuimg` — the image header, the six checks `can_commit` makes before
  anything is erased, the signed region as a predicate, and
  `DfuReceive` for a transfer in progress.
- `dfuusb` — the class triple, the functional descriptor, the class
  requests as control transfers, the bridge to `dfustate`, and the two
  enumerations a DFU device has.
- `dfuserial` — the ten opcodes, `DfuSerialReq` and `DfuSerialRsp`,
  `DfuSerialXfer` with SELECT as its resume point, and the object-size
  and frame-size arithmetic.
- `dfuse` — DfuSe's four commands, its block-2 data numbering, the
  reach of plain DFU that decides whether it is needed, and the DFU
  file suffix.
- `dfucrc` — the one module that reaches crc-nv.
- `dfuerr` — one error type, with `is_pre_erase` and `is_retryable`.
- `dfuhost` — the `host` module: `DfuTransport[e]`, `DfuPlan` for
  either wire, and reading the firmware file, `[io]`.

### Known

- **The load-bearing interface is `DfuStep.wait_ms`.** DFU's whole
  back-pressure is `bwPollTimeout`, and a design that returns
  `Result<(), _>` from a block write has nowhere to put it — so every
  implementation that starts that way hides a sleep in its transport
  and ignores the device's own number. Here the machine answers the
  deadline and the caller waits, which also keeps the module `core`.
- **The layer is `core` with one `host` module**, not the plan's
  `host`: a bootloader is the half that has to be right.
- **`usb-nv` is a PATH dependency** while the two are developed
  together, and `novo pkg publish` refuses one. The published line is
  `usb-nv = "^0.0.1"`, and usb-nv has to reach the registry first.
  `dfustate` names no usb-nv type on purpose, so the state machine
  stays transport-independent and the serial bootloader drives the
  same one.
- **A `@value` struct cannot be a `Result` or optional payload**
  (E2015), so `status_decode`, `header_decode`, `req_decode` and
  `rsp_decode` all answer a zeroed value with a `check_*` beside them.
- **The signature slot is a slot.** No cryptography here; what is
  fixed is the signed region, so two implementations cannot disagree
  about what was signed. ed25519-nv is the package named for the
  signature itself.
- **The device claim covers `dfustate` and `dfuimg`** and
  `tests/embedded_probe.nv` builds them for a Cortex-M4. `dfucrc` is
  deliberately outside it, which is why every other module takes a CRC
  as an `Int` the caller supplies.
- **`novo flash` is the development path and this is the field one.**
  probe-rs over SWD with a debugger attached, against a device running
  its own bootloader over a transport it has anyway. The README says
  where each belongs.
- **Two missing rows**: a firmware-signing tool to pair with the
  signature slot, and a delta format for `DFU_IMG_DELTA`, which is a
  flag with no package behind it.
- Cryptography, compression, a flash driver and a DfuSe implementation
  are deliberately outside.
