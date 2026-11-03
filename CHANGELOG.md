# Changelog

All notable changes to i2c-devices-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-31

Every body under `src/` is written. The suites pass: 131 tests over seven
files, 36 documentation examples, 100% of the 596 instrumented lines
under `src/`, and the probe `tests/embedded_probe.nv`, which runs under
QEMU for a Cortex-M and prints `PASS` with all 104 of its checks. The
manifest moves to `stability = "experimental"`, `version = "0.1.0"`,
`layer = "host"` and `novo = ">= 0.22.0"`.

### Changed

- **The layer is `host`, and `embedded-hal-nv ^0.2.1` is a dependency.**
  `i2cboard` implements `I2cDeviceBus[hw]` for the board's I2C bus, and
  that bus is embedded-hal-nv's. A `core` package depends only on `core`
  packages, so the layer follows the module. Every other module is
  unchanged in effect: the calculations declare none, and the functions
  over a bus declare the bus's.
- **`i2cfault` is `@tier(app)` throughout.** A build for a device checks
  every module under `src/`, and a function with no tier is checked there
  at the embedded one, which refuses a variant that carries a payload.
- **The test files lose their release stamp**, and six fixtures that
  contradicted the suite's own words are repaired. Each is listed under
  "Test fixtures repaired" below.

### Added

- **`i2cboard`** — `BoardBus`, `open`, `word_of` and `bytes_of`. The bus of
  a board as `I2cDeviceBus[hw]`: a register read is one `write_read`
  transaction, a presence probe is a zero-length write, and the
  frequency is the one `open` initialised the bus with, because
  `BoardI2c` can set a frequency and cannot report it. `word_of` and
  `bytes_of` are the conversions between embedded-hal-nv's list of bytes
  and a transfer's word. `i2cboard` is `@tier(rt)`, because a list is
  allocated on the runtime's heap.
- **`i2ctiming.not_started` and `i2ctiming.is_started`.** A trigger that is
  not acknowledged answers a conversion of -1 microseconds. `is_elapsed`
  is false for it however long the caller waits, `remaining_micros` is 0,
  and `longest` of it and a real conversion is the real one. Without it
  the BME280's read after a failed trigger would return the measurement
  made before, which is the failure the two-call design exists to
  prevent. The four `start_*` functions answer it.
- **`bme280.press_cal_with_p9`.** The interface takes the pressure
  coefficients in two eight-byte words, which hold dig_P1 to dig_P8, and
  dig_P9 comes in a third read. `press_cal_from` answers zero for it and
  `read_calibration` calls this to add it. A pressure compensated with
  dig_P9 zero is wrong by a few hundred pascals.
- **`tests/i2c_edge_tests.nv`** — 78 tests. A simulated bus that logs every
  transaction and fails the one a test names checks the order of each
  driver's writes and what it does at each step; the arithmetic is
  checked at every boundary against values computed apart from this
  package (the datasheets' published examples and a transcription of
  Bosch's reference routine, which agree on 25.08 °C and 100653 Pa).
- **`tests/i2c_board_tests.nv`** — the two conversions, and a bus on which
  nothing answers, which is the host's.

### Decisions the interface left open

- **`i2cbus.word` clears the bits above its count and holds the count to
  0 through 8.** `read_reg_checked`, `read_byte` and `update_reg` take a
  transfer that did not complete as -1 or false, and `update_reg` writes
  one byte, `clear` first.
- **`bme280.read_calibration` reads six transactions**, not five: the
  identification byte first, then 0x88, 0x8e, 0x96, 0x9e (four bytes, the
  last of them dig_H1) and 0xe1. It stops at the first transfer that
  fails. A BMP280 takes five, and has no humidity coefficients.
- **`bme280.raw_status` names `I2C_STATUS_RESET_VALUE` when any of the
  three fields holds its reset value.** A block from a part with a channel
  skipped on purpose therefore always names it.
- **`ina219.reading_status` checks overflow, then calibration, then
  readiness.** A missing calibration is named before a conversion that
  is still running, because waiting repairs the second and never the
  first.
- **`ina219.calibration` rounds the current LSB up to a multiple of 50 nA**
  so that the power LSB is a whole number of microwatts, raises it until
  the divisor fits the sixteen-bit register, and clears bit 0 of the
  register value. 0.1 ohm and 3.2 A give 97700 nA, 1954 uW and a
  register value of 4192.
- **`bh1750.start_measurement` powers the part on and writes the
  measurement time register before every one-shot opcode.** Four
  one-byte writes, so a measurement does not depend on what ran before.
- **`sht3x.read_status` answers -1 when the checksum disagrees**, as it
  does for a transfer that failed. `sht3x.dew_point` is the Magnus
  formula in 20-bit fixed point with a logarithm built from integers; it
  differs from the formula in floating point by under a thousandth of a
  degree, and answers 0 outside -45 °C to 60 °C or for no humidity.
- **`bme280.sea_level_pressure` is the barometric formula in integers**,
  within 5 Pa of the floating-point value from -1000 m to 11000 m, and
  holds an altitude outside that range to its nearest end.
- **A value an unknown code, mode or repeatability names** answers the
  default of its family: `sht3x.measure_command` and `conversion_time`
  the high repeatability, `bh1750.oneshot_command` the high resolution,
  `i2cfault.fault_of` no fault.
- **The compensation refuses what would not fit in 64 bits.**
  `bme280.compensate_pressure` and `compensate_humidity` check each
  product against the range it enters and answer the reading with no
  pressure or no humidity, where the datasheet's routine wraps in
  32 bits or 64 and an overflow here is a panic. A part within its
  specification never reaches a check. A calibration image with extreme
  coefficients did panic in the pressure routine before the checks were
  written.

### Test fixtures repaired

Each is a fixture that contradicted the suite's own words. The repair
changes the fixture and leaves the expected value of every assertion as it
was, except the probe's count of its own checks.

- **`tests/bme280_tests.nv`, `example_temp_cal`.** Named the datasheet's
  example coefficients and built eight zero bytes, which cannot compensate
  to the 25.08 °C the case asserts. It is now dig_T1 27504, dig_T2 26435
  and dig_T3 -1000 as the six bytes `70 6b 43 67 18 fc`.
- **`tests/bme280_tests.nv`, the eight-byte block.** `6684000082f80080`
  puts the temperature's three bytes at positions 4 to 6, and the part's
  block has them at 3 to 5. The word that gives the asserted press and temp
  is `66840082f8000080`.
- **`tests/i2cbus_tests.nv`, the tape.** `I2cTestTape` answered its
  `canned` integer whole for a read of any size, so a one-byte read of
  0xd0 answered 0x85 and `chip_is`, `read_byte` and `bme280.identify`
  could not give the 0x60 the cases assert. A read of `n` bytes now
  answers the first `n` of them, as a recorded stream would.
- **`tests/i2cbus_tests.nv`, the status read.** The tape's three bytes
  `60 85 00` have the checksum 0x5b, not 0x00, and a status word that is
  not checked is the failure the SHT3x's header describes. The case reads
  from a second tape, `0x0010` with its checksum 0xc2.
- **`tests/light_and_power_tests.nv`, the bus voltage.** The register word
  0x0678 is 828 mV at 4 mV a count over bits 15 to 3, not the 3.308 V the
  case and the module's example assert. The word that is 3.308 V is
  0x19d8. The doc example in `ina219` uses it too.
- **`tests/embedded_probe.nv`.** The same bus voltage word, the same
  temperature coefficients (`0x7011` is dig_T1 alone, with dig_T2 and
  dig_T3 zero, which compensates to 0), and the count of checks: the
  comment and the comparison said 92 and the program makes 104.

### Known

- **The BME280's reset value is also a count it can measure.** With the
  datasheet's example coefficients a raw temperature of 0x80000 is
  26.46 °C and a raw pressure of 0x80000 is 81874 Pa. At oversampling ×1
  with the filter off the low four bits of the 20-bit value are zero, so a
  part sitting near that temperature produces the count on many readings,
  and `compensate_temperature` and `raw_status` refuse it. A program that
  has configured every channel takes another measurement. A fix wants the
  configuration beside the raw block, and `read_raw` answers a block.
- **An INA219 reading that failed has no code of its own.** The reading
  struct has no field for a transfer that did not complete, so a failed
  read answers `ready` false, and `reading_status` names it not ready or
  uncalibrated. `i2ctiming.ready_status` and `i2cbus.read_reg_checked`
  tell the cases apart.
- **A zero-length write as a presence probe is the board's to support.**
  `i2cboard`'s `i2c_present` writes an empty list. A hook that refuses
  it answers false for every address. It was built and run under QEMU,
  which has no I2C part, and not on silicon.
- **`i2cboard.open` initialises the bus and cannot report a failure**:
  embedded-hal-nv's `init` answers nothing.
- **The `char-literal` finding in `tests/embedded_probe.nv`** is five
  wire numbers that the linter reads as characters. They were there in
  0.0.1.

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
