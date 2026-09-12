# @vcjdeboer/uc2-esp v2026.09.12.4 — Swamp Club Extension

Drive a [UC2-ESP](https://github.com/youseetoo/uc2-esp32) device from swamp over
USB serial, and record every exchange as versioned data. UC2-ESP is an
open-source ESP32 firmware for microscopy control (stages, lasers, LEDs,
cameras); this extension gives it a provenance layer — every command and its
result becomes a timestamped, queryable swamp record.

One model type, `uc2-device`, speaks the UC2-ESP task protocol: each request is
`{ "task": "/endpoint", "qid": <n>, ... }`, and the board answers with an
immediate ACK echoing the qid, then optional async events, then a final DONE
carrying the result — all correlated by the request id.

The serial transport and detached holder under `_lib/` are device-agnostic and
shared in shape with [`@vcjdeboer/esp32-serial`](https://github.com/vcjdeboer/esp32-serial);
this extension only gives the lines UC2 meaning.

## Installation

```
swamp extension pull @vcjdeboer/uc2-esp
```

The device must already run UC2-ESP firmware. Deno is provided by swamp.

## Usage

Create one model instance per device. Leave `device` unset to auto-detect a
single attached board, or pass the port (`/dev/cu.usbmodem*` on macOS,
`/dev/ttyACM*` on Linux):

```
swamp model create @vcjdeboer/uc2-device scope --global-arg holder=true
```

Run a command endpoint (`act`) and read a state endpoint (`get`); each records
the correlated exchange:

```
# move two steppers; the ACK, async events and final DONE are all captured
swamp model method run scope act --input task=/motor_act \
  --input 'args={"motor":{"steppers":[{"stepperid":1,"position":10000,"speed":5000}]}}'
# read current motor state
swamp model method run scope get --input task=/motor_get
```

Read back any result as versioned data:

```
swamp data get scope act-latest --json    # acked, events, result, outcome
swamp data get scope state-latest --json  # the state the board returned
```

For faster back-to-back calls, keep the port open in a detached worker with
`swamp model method run scope hold`, and `... release` when done. With
`holder: true`, methods use it automatically.

## Models

| Model type | For | Methods |
| --- | --- | --- |
| `@vcjdeboer/uc2-device` | A device running UC2-ESP firmware | `detect`, `act`, `get`, `listen`, `hold`, `release` |

`act` runs a `/…_act` endpoint and records the ACK, the async events, and the
final DONE. `get` reads a `/…_get` endpoint and records the returned state.
`listen` captures unsolicited events the device pushes on its own (an encoder
turning, a heartbeat) — the async leg of the protocol:

```
swamp model method run scope listen --input timeoutMs=3000
swamp data get scope events-latest --json    # the captured event lines
```

Every record carries an explicit `outcome` (ok, error, or timeout), so a run
that fails partway is always distinguishable in the data. Reference records in
workflows as `data.latest("scope", "act-latest")`.

## Operational limits

- **Firmware required.** The device must run UC2-ESP; this extension does not
  flash it. Use the firmware and flashing tools from the UC2-ESP project.
- **USB-CDC serial only.** A board on a CP2102/CH340 bridge enumerates under a
  different name (`usbserial-*`, `wchusbserial-*`) that auto-detect does not yet
  match — pass `device` explicitly.
- **Timing is millisecond-class**, as the UC2-ESP paper notes: not for
  sub-microsecond triggering. Long actions rely on the final DONE, not on a
  fixed delay; a missing DONE within `timeoutMs` is recorded as `outcome:
  timeout`, then thrown.
- **Verified against a fake board** that emulates the UC2 ACK/event/DONE dance
  (see `extensions/files/fake_board.py` and the tests), not yet against physical
  UC2-ESP hardware.

## Citation

This extension implements the protocol described in, and interoperates with the
firmware from:

> Diederich, B., Fuchs, I., Wang, H., Bierhoff, H., Kuttke, C., & Heintzmann, R.
> (2026). UC2-ESP: A general-purpose framework for open-source microscopy
> control. *Journal of Microscopy*, 303, 249–262.
> https://doi.org/10.1111/jmi.70147

Firmware: https://github.com/youseetoo/uc2-esp32
