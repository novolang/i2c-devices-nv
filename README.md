# i2c-devices-nv

I2C is a two-wire bus that connects a controller to small peripheral
parts over a shared clock and a shared data line, specified in NXP's
[UM10204](https://www.nxp.com/docs/en/user-guide/UM10204.pdf). Most of
the sensors a battery-powered measuring device carries are on it. This
package brings four of them to novo-lang: the Bosch BME280 environment
sensor, the Sensirion SHT3x humidity sensor, the ROHM BH1750FVI light
sensor and the Texas Instruments INA219 current monitor.

**Status: NOT IMPLEMENTED — interface only.**  Every function is
declared with its full signature, but every body is a `todo()` that
panics when called.  The package is published so its design can be
reviewed and depended on before it is implemented.  Version 0.1.0 will
be the first working release.

## What it is

A sensor on an I2C bus does not answer a temperature. It answers a
number in one of its registers, and turning that number into a
temperature takes a calculation the part's manufacturer publishes in its
datasheet. For the BME280 the calculation needs twenty-six bytes of
calibration data burned into the individual part at manufacture, and it
is thirty lines of integer arithmetic. For the SHT3x it is one
multiplication and one subtraction, after a checksum has been verified.
For the BH1750 it is a division by a factor the manufacturer gives as a
range rather than a value. This package is mostly those calculations,
one module per part, with each constant named and the section of the
datasheet it comes from written beside it.

The rest of it is the bus. A driver has to write a register, read some
bytes back and know what to do when nothing answers. Those operations
are declared here as a trait, `I2cDeviceBus`, and this package
implements it for nothing: a program supplies a bus of its own, over a
microcontroller's peripheral, over a Linux kernel device, or over a
table of bytes recorded from a real part. Every driver function is
written once against the trait and works over all three.

**Measurement is not instantaneous.** Each part here measures when it is
asked to, and each takes time: about nine milliseconds for the BME280 at
the settings Bosch recommends for a weather station, fifteen and a half
for the SHT3x at its most repeatable, a hundred and eighty for the
BH1750. Reading a part before it has finished does not produce an error
that a program can see. The BME280 answers its previous measurement,
which is a real reading from a real sensor and is simply old. The SHT3x
refuses the read at the bus level, which looks exactly like a
disconnected wire. The BH1750 answers a partial integration, which is a
light level that is too low in proportion to how early the read was.

So a measurement here is two calls. The first triggers it and answers
how long it will take. The caller waits, by whatever means it has. The
second takes how long the caller actually waited, and refuses to produce
a reading when that is not enough. Nothing in this package sleeps,
consults a clock or holds the bus while a part converts.

**Nothing here performs input or output.** The parts of the package that
compute — every register constant, every compensation formula, the
Sensirion checksum, the unit conversions — are ordinary functions over
integers. The parts that talk to a part are written over the bus trait,
so what they cost is whatever the program's own bus costs. A program
whose bus is a microcontroller peripheral pays for hardware access; a
test whose bus is a recorded table pays nothing.

## Install

```
novo pkg add i2c-devices-nv
```

## Example

```novo
use bme280
use i2cbus
use i2csample
use i2cstatus

// The program's own bus.  A real one drives a peripheral; this one
// answers bytes from a recording, which is what a test uses.
struct RecordedBus
    bytes: Int

impl I2cDeviceBus[] for RecordedBus
    fn i2c_write_reg(self, addr: Int, reg: Int, val: Int) -> Bool
        true
    fn i2c_read_reg(self, addr: Int, reg: Int, n: Int) -> I2cWord
        i2cbus.word(self.bytes, n)
    fn i2c_write_raw(self, addr: Int, bytes: Int, len: Int) -> Bool
        true
    fn i2c_read_raw(self, addr: Int, n: Int) -> I2cWord
        i2cbus.word(self.bytes, n)
    fn i2c_present(self, addr: Int) -> Bool
        true
    fn i2c_frequency(self) -> Int
        400000

fn main() [io]
    let bus = RecordedBus { bytes: 0x6684000082f80080 }
    let addr = bme280.BME280_ADDR_LOW

    // Three parts share this address, so ask which one answered.
    if bme280.identify(bus, addr) != bme280.BME280_CHIP_ID
        println("not a BME280")
        return

    // Read the part's own calibration data.  Done once and kept: it is
    // thirty-three bytes and it never changes.
    let cal = bme280.read_calibration(bus, addr)

    // Configure the part and take one measurement.  `start_measurement`
    // answers how long the measurement will take.
    let settings = bme280.weather_monitoring()
    let _ = bme280.configure(bus, addr, settings)
    let wait = bme280.start_measurement(bus, addr, settings)

    // Wait, by whatever means the program has, and say how long it
    // waited.  A read that waited too little is refused.
    let waited = wait.micros
    let raw = bme280.read_raw(bus, addr, wait, waited)

    // Turn the raw registers into numbers.  Temperature first: the
    // pressure and humidity calculations both need a value it produces.
    let reading = bme280.compensate(cal, raw)
    let t = bme280.temperature(reading)

    // The temperature is in thousandths of a degree Celsius.
    if t.valid
        println("${i2csample.whole(t.value)}.${i2csample.milli_fraction(t.value)} C")
    else
        println(i2cstatus.status_name(bme280.raw_status(raw)))
```

Build and test with `novo pkg build` and `novo test tests`. Today
`novo test` fails on purpose: every test reaches a `not implemented`
panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `i2cbus` | The bus, as a trait a program implements: write a register, read up to eight bytes, write and read without a register pointer, and ask whether anything is at an address. Also the arithmetic that takes bytes out of a transfer, in both byte orders, and the conversion between the two ways a datasheet prints an address. |
| `i2cstatus` | The nine codes a driver answers when a reading does not arrive, and which of them a retry can fix. |
| `i2cfault` | The same nine as a value that carries the address, the checksums or the raw word that explain it, for a program with a log. |
| `i2ctiming` | How long a part needs before its measurement can be read, and whether a caller has waited that long. |
| `i2csample` | One measured quantity in an integer unit, the seven quantities the package measures, the range each part is specified over, and the pieces of a printed number. |
| `bme280` | The Bosch BME280 and the BMP280 that shares its address: the register map, the three sets of recommended settings, the calibration data, and the compensation calculations for temperature, pressure and humidity. |
| `sht3x` | The Sensirion SHT3x-DIS: the commands, the CRC-8 every reply carries, the two conversions and the status register. |
| `bh1750` | The ROHM BH1750FVI: the opcodes, the measurement time register and the illuminance conversion. |
| `ina219` | The Texas Instruments INA219: the six registers, the configuration word, the calibration a caller computes from its own shunt resistor, and the four values one poll produces. |

## How to choose an entry point

**A program with a bus calls the part's own module.** `bme280`,
`sht3x`, `bh1750` and `ina219` each have a `start_measurement` and a
`read_*` pair that take a bus, and that is the ordinary way in.

**A program with bytes already in hand calls the calculations
directly.** Every conversion — `bme280.compensate`,
`sht3x.temperature_of`, `bh1750.milli_lux_of`, `ina219.bus_uv_of` —
takes numbers and returns numbers with no bus involved. That is the form
to use when replaying a recording, when the bytes arrived by radio from
somewhere else, or when writing a test.

**Firmware calls the calculations and supplies its own bus.** The
modules that compute build for a microcontroller with no heap allocator.
See "Running on a microcontroller".

**A program that writes a log calls `i2cfault`; a program in firmware
calls `i2cstatus`.** The two describe the same nine failures. The first
carries the numbers that explain each one and allocates to do it; the
second is an integer.

## The rules a user needs

1. **An address here is seven bits.** Datasheets disagree about which
   number they print: Bosch's tables give the BME280 as 0xec, which is
   0x76 with the read/write bit appended. `i2cbus.addr_from_write_byte`
   converts. A program that uses the printed number addresses nothing.
2. **Check which part answered before configuring it.** 0x76 and 0x77
   are used by the BME280, the BMP280 and the MS5611. Writing a BME280's
   configuration to a BMP280 succeeds, and the humidity register then
   reads a value that compensates to a number (BST-BME280-DS002 § 5.3).
3. **Wait the conversion time, and pass how long you waited.** Every
   `read_*` takes the conversion its `start_*` answered and the elapsed
   microseconds, and refuses when they do not agree. Round the wait up
   to the millisecond with `i2ctiming.conversion_millis_ceil`: rounding
   8.7 down to 8 is a read that is too early on every cycle.
4. **Compensate the BME280's temperature before its pressure or
   humidity.** The temperature calculation produces an intermediate the
   other two consume (BST-BME280-DS002 § 4.2.3). `bme280.compensate`
   does all three in the required order; the three separate functions
   must be called in that order by hand.
5. **The BME280's calibration data is stored least significant byte
   first and its measurements most significant byte first**
   (BST-BME280-DS002 § 4.2.2 against § 5.4.7). `i2cbus` has a function
   for each order. Using one order for both gives a temperature that
   moves in the right direction and is wrong by tens of degrees.
6. **Two of the BME280's humidity coefficients share a byte.** dig_H4
   and dig_H5 are twelve bits each, packed into the three bytes at 0xe4
   to 0xe6, with the low nibble of 0xe5 belonging to the first and the
   high nibble to the second (BST-BME280-DS002 § 4.2.2, table 16).
7. **Write the BME280's humidity oversampling register before its
   measurement control register.** Changes to `ctrl_hum` take effect
   only when `ctrl_meas` is next written (BST-BME280-DS002 § 5.4.3).
   `bme280.configure` does this; a program writing the registers itself
   must.
8. **Read the BME280's eight measurement registers in one transfer.**
   In continuous mode the part can finish a new measurement between two
   single-register reads, so three bytes come from one measurement and
   three from the next (BST-BME280-DS002 § 4).
9. **Check both of the SHT3x's checksums.** A measurement reply is six
   bytes: two of temperature, a checksum over those two, two of
   humidity, and a checksum over those two (SHT3x-DIS § 4.3). The check
   is per pair, not per reply.
10. **Break the SHT3x out of continuous measurement before using
    single-shot commands.** A part in that mode ignores every command
    except break and fetch, a soft reset included (SHT3x-DIS § 4.5).
11. **The BH1750's reading saturates without saying so.** At the
    default measurement time the ceiling is about 54 612 lux, which is
    below direct sunlight. `bh1750.reading_status` answers
    `I2C_STATUS_SATURATED` there; lowering the measurement time register
    raises the ceiling to about 121 000 lux.
12. **The BH1750's measurement time register scales the reading as well
    as the integration.** Changing it without dividing the result by the
    same ratio reports a room that suddenly got brighter. `milli_lux_of`
    takes the register value for that reason.
13. **The INA219's current and power registers read zero until its
    calibration register is written** (SBOS448 § 8.5.1). Zero amperes is
    indistinguishable from an idle load. `ina219.current` and
    `ina219.power` answer an invalid sample against a part that has not
    been calibrated, and the two voltage registers are readable either
    way.
14. **Read the INA219's power register last.** Reading it is what
    clears the conversion-ready flag, so a loop that polls that flag and
    reads only the current register never finishes (SBOS448 § 8.6.3).
15. **The INA219's overflow flag is the only warning of a bad
    calculation.** When the current or power exceeds the range the
    calibration set up, bit 0 of the bus voltage register is set and
    both registers hold meaningless values (SBOS448 § 8.6.3).

## Running on a microcontroller

The modules that compute build for a microcontroller with no heap
allocator: `i2cbus`'s byte and address arithmetic, `i2cstatus`,
`i2ctiming`, `i2csample`, and every conversion in `bme280`, `sht3x`,
`bh1750` and `ina219`. `tests/embedded_probe.nv` is a firmware program
that calls ninety-two of them and is compiled for a Cortex-M as part of
this package's checks, so the claim is built rather than asserted.

Two things are excluded and each is excluded for a reason.

`i2cfault` is not, and nothing that is builds against it. Its values
carry the address or the pair of checksums that explain a failure, and a
value of that shape is allocated on a heap. A driver in firmware answers
an `i2cstatus` code, which is an integer, and a program with a heap
turns a code into a fault with `i2cfault.fault_of`.

The functions that drive a bus are not excluded so much as unfinished
until a program supplies a bus. They are written over `I2cDeviceBus`,
and the implementation is the program's: on a microcontroller it drives
a peripheral, which is the one part of a measurement that touches
hardware.

## What is not included

- **A bus implementation.** This package drives parts; it does not
  drive controllers. A microcontroller's I2C peripheral belongs to the
  board support for that part, and a Linux `/dev/i2c-N` device needs an
  `ioctl` call, which novo-lang reaches only through a C library.
- **Clock stretching.** The SHT3x can hold the clock line low until its
  measurement is finished, so that a read issued immediately blocks
  until the answer is ready (SHT3x-DIS § 4.3). Fifteen milliseconds of a
  held clock line is fifteen milliseconds during which no other part on
  the bus can be reached, many microcontroller I2C peripherals abort a
  transfer stretched that long, and a device that sleeps through its
  measurement cannot sleep in the middle of a transfer.
- **The SHT3x's periodic and alert modes.** Periodic measurement, the
  accelerated response time mode and the alert limit registers are a
  second interface to the same part, and a program that uses them holds
  state across calls that the single-shot interface does not need.
- **The SHT4x.** A different part with a different command set and one
  address rather than two, despite the name.
- **The BME280's continuous mode standby times.** The package writes
  the standby field and does not offer the timings as a choice, because
  a part that measures on its own is a part a battery-powered device
  does not want.
- **Altitude from pressure.** Computing it needs the sea level pressure
  of the day, which no sensor knows. A calculation that assumes the
  standard 101325 Pa answers a number that moves thirty metres with the
  weather. Pressure corrected *to* sea level, which needs only a known
  altitude, is `bme280.sea_level_pressure`.
- **The INA219's 16-address strapping table.** The address is a
  property of the board, and a program that knows its board knows its
  address.
- **Other parts.** The MCP9808 temperature sensor with its alert
  window, the ADS1115 analogue-to-digital converter, the SSD1306 display
  controller, the MPU6050 and LSM6DS3 motion sensors, and the DS3231
  real-time clock are all common on this bus and none of them is here.
  Each is a module of the same shape and each is a separate piece of
  work.

## Related packages

- **crc-nv** publishes CRC-16/CCITT-FALSE, CRC-16/MODBUS, CRC-32 and
  CRC-32C. The SHT3x needs CRC-8 with polynomial 0x31 and an initial
  value of 0xff, which is not among them, and `crc-nv`'s value owns a
  256-entry table as a list, which is a heap allocation and a quarter of
  a kilobyte to check three bytes. `sht3x.crc8` computes the same number
  in twenty-four iterations with no table. An eight-bit width in crc-nv,
  with a form that builds no table, would let this package drop it.
- **bitfield-nv** describes a hardware register as an offset, a width
  and a signedness, and extracts and inserts fields by those
  descriptions. The register packing here is written by hand as shifts
  and masks, because a description is a value a caller holds and there
  are only a dozen packed registers in the package.
- **fixedpoint-nv** is the package to reach for when a calculation
  wants fractions on a part with no floating-point unit. Everything here
  is integer arithmetic in a fixed unit, which is the same idea with the
  scale chosen once per quantity rather than carried in a type.
- **modbus-nv** and **can-nv** are the other two buses. Modbus reaches
  sensors over a serial line or a network and CAN reaches them over a
  differential pair; both carry parts that this one cannot, and neither
  overlaps it.

## Tests

The suite is five files under `tests/`, written against the signatures
and red until the bodies land. What they assert is drawn from the
datasheets rather than from an implementation: the BME280's own worked
example of 519888 raw counts giving 25.08 °C (BST-BME280-DS002
§ 4.2.3), the SHT3x's published checksum of 0x92 over the bytes 0xbe and
0xef (SHT3x-DIS § 4.4), the measurement times each manufacturer states
as a maximum, and the register values each part powers up holding.

`tests/i2cbus_tests.nv` also implements `I2cDeviceBus` over a table of
bytes, which is both the shape a full test suite will take and the
demonstration that a program's bus can be one that touches nothing.

`tests/embedded_probe.nv` is the microcontroller program described
above.

## Implementation status

| Item | Implemented |
| --- | --- |
| `i2cbus` — the trait and the byte arithmetic | no |
| `i2cstatus` — the nine codes | no |
| `i2cfault` — the codes as values with their numbers | no |
| `i2ctiming` — conversion times and the wait check | no |
| `i2csample` — quantities, units and ranges | no |
| `bme280` — registers, calibration and compensation | no |
| `sht3x` — commands, checksum and conversions | no |
| `bh1750` — opcodes, measurement time and illuminance | no |
| `ina219` — registers, calibration and the four values | no |

## Licence

Apache-2.0.  See `LICENSE`.
