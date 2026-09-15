# Changelog

All notable changes to i2c-devices-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-15

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `i2cbus` — `I2cDeviceBus[e]`, the trait a program implements over its
  own peripheral, and the arithmetic that takes bytes out of a transfer.
  A transfer answers up to EIGHT BYTES IN ONE INTEGER, because eight
  bytes is sixty-four bits and the longest burst any part here needs is
  the BME280's measurement block — so a whole burst is one integer in
  the caller's frame and nothing is allocated.  The word carries its own
  `ok` rather than a sentinel: an eight-byte read fills all sixty-four
  bits, and -1 is what a bus with no pull-up resistors reads back.  Both
  byte orders are separate functions, because the BME280 mixes them.
- `i2cstatus` — nine codes, `@tier(embedded)` throughout, and
  `is_transient` as the distinction a retry loop has to make: an address
  that never answered is a wiring fault and no retry fixes it, a part
  that is still converting answers the moment the caller waits.
- `i2cfault` — the same nine as an enum carrying the address, the two
  checksums or the raw word.  SPLIT FROM `i2cstatus` because a
  payload-carrying enum variant is a heap value that `@tier(embedded)`
  refuses: no part module imports this one, so the device probe's use
  graph never reaches it.
- `i2ctiming` — the package's load-bearing interface.  A measurement is
  TWO CALLS: the first answers how long it takes and the second takes
  how long the caller waited and refuses when it is short.  Nothing here
  sleeps or reads a clock.  The wait rounds UP to the millisecond,
  because truncating 8.7 to 8 is a read that is too early on every
  cycle — and an early read is a stale measurement on the BME280, a
  bus-level refusal on the SHT3x and a partial integration on the
  BH1750, none of which reports an error.
- `i2csample` — one measured quantity in an integer unit.  Integers and
  not `Float`, because a Cortex-M0+ has no floating-point unit and
  novo-lang's `Float` is double.  `in_range` is the check that matters:
  the failure this package is most likely to produce is a PLAUSIBLE
  number, and neither a byte-swapped coefficient nor an uncompensated
  reset value is caught by asking whether the transfer succeeded.
- `bme280` — the register map with its datasheet sections, three sets of
  recommended settings, the calibration in three values rather than one
  struct of eighteen fields, and § 4.2.3's compensation.  Temperature is
  compensated FIRST and answers the `t_fine` the other two consume, so a
  caller cannot forget an ordering the datasheet states in prose.
- `sht3x` — the commands, the CRC-8 the datasheet publishes a check
  value for, the two conversions over 2¹⁶-1 rather than 2¹⁶, and the
  status bit that says the part restarted.  Clock stretching is NOT
  used, and the README gives three reasons.
- `bh1750` — the opcodes, the measurement time register that scales the
  reading as well as the integration, and `is_saturated`, because a
  part at its ceiling answers a number and keeps answering it.
- `ina219` — the six registers, the calibration a caller computes from
  its own shunt, and `gain_fits`, which catches a calibration that is
  arithmetically fine and physically impossible.  The current and power
  registers read a clean zero until register 0x05 is written, which is
  the whole reason `calibrated` is a field on every reading.
- `tests/embedded_probe.nv` — ninety-two checks over the arithmetic,
  built for `--target=nrf52-qemu`.
- API tests in five files, red until the bodies land, with the
  datasheets' own published values as the oracle.

### Named as missing

**An eight-bit width in crc-nv.**  The SHT3x checks with CRC-8,
polynomial 0x31 and initial value 0xff.  crc-nv publishes four
algorithms and none is eight bits wide; its `Crc` value also owns a
256-entry table as a `[Int]`, which allocates.  `sht3x.crc8` is the
bitwise form, and the row crc-nv should open is that width plus a
table-free variant for a device.

**A bus implementation for either platform.**  A microcontroller's I2C
peripheral belongs to that board's support package; a Linux
`/dev/i2c-N` device needs `ioctl`, which reaches novo-lang only through
a C library.  So this package drives parts and implements
`I2cDeviceBus` nowhere, which is why it declares no effect of its own.

**The drivers the row assumed already existed.**  The plan's note for
this row reads "the sensor drivers sensorhub carries, as a package", and
`orbit/sensorhub` carries none: its one sensor is the nRF52's own on-die
temperature peripheral, read over memory-mapped registers, and its
README says in as many words that a BME280 over I2C would be a more
interesting number and a worse demo.  Nothing in the repository drives
an I2C part today.  The surface here is therefore ported from the
embedded driver crates directly, and what sensorhub would take from it
is its temperature and humidity against a two-second duty cycle, plus
the INA219 — whose reading is the current draw sensorhub's own README
says a program cannot report about itself.
