# i2c-devices-nv

I2C is a two-wire bus that connects a controller to small peripheral
parts over a shared clock and a shared data line, specified in NXP's
[UM10204](https://www.nxp.com/docs/en/user-guide/UM10204.pdf). Most of
the sensors a battery-powered measuring device carries are on it. This
package brings four of them to novo-lang: the Bosch BME280 environment
sensor, the Sensirion SHT3x humidity sensor, the ROHM BH1750FVI light
sensor and the Texas Instruments INA219 current monitor. It also
provides `I2cDeviceBus`, the bus the drivers are written over, and
implements it for the I2C bus of a board through
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv).

## What it is

A sensor on an I2C bus does not answer a temperature. It answers a
number in one of its registers, and turning that number into a
temperature takes a calculation the part's manufacturer publishes in its
datasheet. For the BME280 the calculation needs thirty-three bytes of
calibration data burned into the individual part at manufacture, and it
is thirty lines of integer arithmetic. For the SHT3x it is one
multiplication and one subtraction, after a checksum has been verified.
For the BH1750 it is a division by a factor the manufacturer gives as a
range rather than a value. This package is mostly those calculations,
one module per part, with each constant named and the section of the
datasheet it comes from written beside it.

The rest of it is the bus. A driver has to write a register, read some
bytes back and know what to do when nothing answers. Those operations
are declared here as a trait, `I2cDeviceBus`. A transfer answers up to
eight bytes in one integer, so a whole burst is a value in the caller's
stack frame and nothing is allocated. `i2cboard` implements the trait for
a board's I2C bus. A program can implement it for anything else: a Linux
kernel device, or a table of bytes recorded from a real part. Every
driver function is written once against the trait and works over all of
them.

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

**What a device program is charged.** The calculations perform no input
or output and declare no effect: every register constant, every
compensation formula, the Sensirion checksum and the unit conversions
are ordinary functions over integers. The functions that talk to a part
cost what the bus costs. Over `i2cboard` that is `[hw]`, the effect of
reaching the board's I2C hardware, and over a recorded table it is
nothing.

## Install

```
novo pkg add i2c-devices-nv
```

## Example

A device program: open the board's bus, check which part answered, read
its calibration, take one measurement and print the temperature. The
program waits with the board's own timer, because the package does not
wait for it.

```novo
use std.str
use bme280
use i2cboard
use i2csample
use i2ctiming

@tier(rt)
fn main() [hw]
    bsp.board.init()

    // The board's first I2C bus, at 400 kHz.
    let bus = i2cboard.open(0, 400000)
    let addr = bme280.BME280_ADDR_LOW

    // Three parts share this address, so ask which one answered.
    if bme280.identify(bus, addr) != bme280.BME280_CHIP_ID
        hal.uart.write("no BME280\n")
        return

    // The part's own calibration data: read once and kept, because it
    // is thirty-three bytes and never changes.
    let cal = bme280.read_calibration(bus, addr)

    // Configure the part and take one measurement.  `start_measurement`
    // answers how long the measurement will take.
    let settings = bme280.weather_monitoring()
    let wait = bme280.start_measurement(bus, addr, settings)

    // Wait, rounding up to the millisecond, and say how long that was.
    // A read that waited too little is refused.
    let millis = i2ctiming.conversion_millis_ceil(wait)
    hal.timer.delay_ms(millis)
    let raw = bme280.read_raw(bus, addr, wait, millis * 1000)

    // Turn the raw registers into numbers.  Temperature first: the
    // pressure and humidity calculations both need a value it produces.
    let t = bme280.temperature(bme280.compensate(cal, raw))

    // The temperature is in thousandths of a degree Celsius.
    if t.valid
        hal.uart.write(str.from_int(t.value))
        hal.uart.write(" ")
        hal.uart.write(i2csample.unit_name(t.quantity))
        hal.uart.write("\n")
    else
        hal.uart.write("no reading\n")
```

The calculations also work on bytes already in hand, with no bus. This
program compensates the datasheet's own example: a raw temperature of
519888 against the example coefficients, which Bosch gives as 25.08 °C.

```novo
use bme280
use i2cbus

fn main() [io]
    // dig_T1 27504, dig_T2 26435 and dig_T3 -1000, as the six bytes the
    // part stores them in: least significant byte first.
    let cal = bme280.temp_cal_from(i2cbus.word(0x706b436718fc, 6))
    let reading = bme280.compensate_temperature(cal, 519888)

    // Thousandths of a degree Celsius.
    println("${reading.milli_c}")   // 25080
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `i2cboard` | The bus of a board: `I2cDeviceBus` implemented for embedded-hal-nv's `BoardI2c`, and the two conversions between a list of bytes and a transfer's word. |
| `i2cbus` | The bus, as a trait a program implements: write a register, read up to eight bytes, write and read without a register pointer, and ask whether anything is at an address. Also the arithmetic that takes bytes out of a transfer, in both byte orders, and the conversion between the two ways a datasheet prints an address. |
| `i2cstatus` | The nine codes a driver answers when a reading does not arrive, and which of them a retry can fix. |
| `i2cfault` | The same nine as a value that carries the address, the checksums or the raw word that explain it, for a program with a log. |
| `i2ctiming` | How long a part needs before its measurement can be read, and whether a caller has waited that long. |
| `i2csample` | One measured quantity in an integer unit, the seven quantities the package measures, the range each part is specified over, and the pieces of a printed number. |
| `bme280` | The Bosch BME280 and the BMP280 that shares its address: the register map, the three sets of recommended settings, the calibration data, and the compensation calculations for temperature, pressure and humidity. |
| `sht3x` | The Sensirion SHT3x-DIS: the commands, the CRC-8 every reply carries, the two conversions, the dew point and the status register. |
| `bh1750` | The ROHM BH1750FVI: the opcodes, the measurement time register and the illuminance conversion. |
| `ina219` | The Texas Instruments INA219: the six registers, the configuration word, the calibration a caller computes from its own shunt resistor, and the four values one poll produces. |

| Quantity | Unit | Part |
| --- | --- | --- |
| Temperature | thousandths of a degree Celsius | BME280, SHT3x |
| Relative humidity | thousandths of a percent | BME280, SHT3x |
| Pressure | pascals | BME280 |
| Illuminance | thousandths of a lux | BH1750 |
| Voltage | microvolts | INA219 |
| Current | microamperes | INA219 |
| Power | microwatts | INA219 |

## How to choose an entry point

**A program with a bus calls the part's own module.** `bme280`,
`sht3x`, `bh1750` and `ina219` each have a `start_measurement` and a
`read_*` pair that take a bus, and that is the ordinary way in. On a
board, `i2cboard.open` makes the bus.

**A program with bytes already in hand calls the calculations
directly.** Every conversion — `bme280.compensate`,
`sht3x.temperature_of`, `bh1750.milli_lux_of`, `ina219.bus_uv_of` —
takes numbers and returns numbers with no bus involved. That is the form
to use when replaying a recording, when the bytes arrived by radio from
somewhere else, or when writing a test.

**A program on a bus of its own implements `I2cDeviceBus`.** One method
is a register write, two are the two kinds of read, one is a write with
no register pointer, one is a presence probe and one reports the bus
frequency. A table of recorded bytes is an implementation that performs
nothing, and every driver function over it declares no effect.

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
   `bme280.identify` and `bme280.read_calibration` read the identification
   byte first.
3. **Wait the conversion time, and pass how long you waited.** Every
   `read_*` takes the conversion its `start_*` answered and the elapsed
   microseconds, and refuses when they do not agree. Round the wait up
   to the millisecond with `i2ctiming.conversion_millis_ceil`: rounding
   8.7 down to 8 is a read that is too early on every cycle.
4. **A trigger that was not acknowledged is a conversion that never
   started.** The `start_*` function then answers
   `i2ctiming.not_started`, and the `read_*` that follows refuses
   however long the caller waits. Without that, the BME280 would answer
   the measurement it made before.
5. **Compensate the BME280's temperature before its pressure or
   humidity.** The temperature calculation produces an intermediate the
   other two consume (BST-BME280-DS002 § 4.2.3). `bme280.compensate`
   does all three in the required order; the three separate functions
   must be called in that order by hand.
6. **The BME280's calibration data is stored least significant byte
   first and its measurements most significant byte first**
   (BST-BME280-DS002 § 4.2.2 against § 5.4.7). `i2cbus` has a function
   for each order. Using one order for both gives a temperature that
   moves in the right direction and is wrong by tens of degrees.
7. **Two of the BME280's humidity coefficients share a byte.** dig_H4
   and dig_H5 are twelve bits each, packed into the three bytes at 0xe4
   to 0xe6, with the low nibble of 0xe5 belonging to the first and the
   high nibble to the second (BST-BME280-DS002 § 4.2.2, table 16).
8. **Write the BME280's humidity oversampling register before its
   measurement control register.** Changes to `ctrl_hum` take effect
   only when `ctrl_meas` is next written (BST-BME280-DS002 § 5.4.3).
   `bme280.configure` does this; a program writing the registers itself
   must.
9. **Read the BME280's eight measurement registers in one transfer.**
   In continuous mode the part can finish a new measurement between two
   single-register reads, so three bytes come from one measurement and
   three from the next (BST-BME280-DS002 § 4).
10. **The BME280's reset value is also a count it can measure.** The
    pressure and temperature registers read 0x80000, and the humidity
    register 0x8000, until the first conversion and when a channel is
    skipped (BST-BME280-DS002 § 5.4). `raw_status` answers
    `I2C_STATUS_RESET_VALUE` for a block that holds one. With the
    datasheet's example coefficients a raw temperature of 0x80000 is
    26.46 °C and a raw pressure of 0x80000 is 81874 Pa, so a part that
    answers can produce it. A program that has configured every channel
    and is refused for this reason takes another measurement.
11. **Check both of the SHT3x's checksums.** A measurement reply is six
    bytes: two of temperature, a checksum over those two, two of
    humidity, and a checksum over those two (SHT3x-DIS § 4.3). The check
    is per pair, not per reply. The status register's reply carries one
    too, and `sht3x.read_status` answers -1 when it disagrees.
12. **Break the SHT3x out of continuous measurement before using
    single-shot commands.** A part in that mode ignores every command
    except break and fetch, a soft reset included (SHT3x-DIS § 4.5).
    `sht3x.reset` sends the break first.
13. **The BH1750's reading saturates without saying so.** At the
    default measurement time the ceiling is about 54 612 lux, which is
    below direct sunlight. `bh1750.reading_status` answers
    `I2C_STATUS_SATURATED` there; lowering the measurement time register
    raises the ceiling to about 121 000 lux.
14. **The BH1750's measurement time register scales the reading as well
    as the integration.** Changing it without dividing the result by the
    same ratio reports a room that suddenly got brighter. `milli_lux_of`
    takes the register value for that reason, and `start_measurement`
    writes the register every time so that a measurement does not depend
    on what ran before it.
15. **The INA219's current and power registers read zero until its
    calibration register is written** (SBOS448 § 8.5.1). Zero amperes is
    indistinguishable from an idle load. `ina219.current` and
    `ina219.power` answer an invalid sample against a part that has not
    been calibrated, and the two voltage registers are readable either
    way.
16. **Read the INA219's power register last.** Reading it is what
    clears the conversion-ready flag, so a loop that polls that flag and
    reads only the current register never finishes (SBOS448 § 8.6.3).
    `ina219.read_reading` reads it last.
17. **The INA219's overflow flag is the only warning of a bad
    calculation.** When the current or power exceeds the range the
    calibration set up, bit 0 of the bus voltage register is set and
    both registers hold meaningless values (SBOS448 § 8.6.3).

## Running on a microcontroller

These modules are written for the embedded tier, which allocates
nothing and has no heap: `i2cbus`'s byte and address arithmetic,
`i2cstatus`, `i2ctiming`, `i2csample`, and every conversion in `bme280`,
`sht3x`, `bh1750` and `ina219`. The functions that drive a bus compile
for firmware as well, over whatever bus the firmware supplies.
`tests/embedded_probe.nv` is a firmware program that calls a hundred and
four of the conversions. It is compiled for a Cortex-M with
`novo build --target=nrf52-qemu`, and run in QEMU it prints `PASS` when
every one of them gives on the device the number it gives on the host.

Two modules are not embedded-tier code, and the registry measures the
tiers of each module from its sources rather than taking them from this
page.

`i2cboard` is `@tier(rt)`. embedded-hal-nv moves a transfer as a list of
bytes, and a list is allocated on the heap the runtime reserves. Each
transaction through it is `[hw]`.

`i2cfault` is `@tier(app)`. Its values carry the address or the pair of
checksums that explain a failure, and a value of that shape is allocated
on a heap. A driver in firmware answers an `i2cstatus` code, which is an
integer, and a program with a heap turns a code into a fault with
`i2cfault.fault_of`.

## What is not included

- **A bus for Linux.** A `/dev/i2c-N` device needs an `ioctl` call,
  which novo-lang reaches only through a C library. A program that has
  one implements `I2cDeviceBus` over it.
- **The traits of devices-nv.** [devices-nv](https://novo-lang.org/packages/devices-nv)
  defines the traits a program reads a thermometer, a hygrometer, a
  barometer or a power manager through. The drivers here are called
  directly and do not implement them.
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

- **embedded-hal-nv** defines `BoardI2c`, the handle that names one of a
  board's I2C buses, and the sync and async traits that move a list of
  bytes over it. `i2cboard` is the adapter between that interface and the
  one these drivers are written over.
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

The suites under `tests/` take their expected values from the
datasheets and from the manufacturers' reference routines rather than
from this implementation: the BME280's own worked example of 519888 raw
counts giving 25.08 °C and 415148 raw counts giving 100653 Pa
(BST-BME280-DS002 § 4.2.3), the SHT3x's published checksum of 0x92 over
the bytes 0xbe and 0xef (SHT3x-DIS § 4.4), the measurement times each
manufacturer states as a maximum, and the register values each part
powers up holding.

`tests/i2cbus_tests.nv` implements `I2cDeviceBus` over a table of bytes
that performs nothing, so every driver function runs there with no
effect declared.

`tests/i2c_edge_tests.nv` implements it over a table of registers that
keeps a log of every transaction in module-level state, and so declares
`[mutate]`. A test can then check the order of the writes a driver
makes, and make the transaction with a given number fail to check what
the driver does at each step.

`tests/i2c_board_tests.nv` runs `i2cboard` on the host, where the I2C bus
has no device and every transaction is refused.

`tests/embedded_probe.nv` is the microcontroller program described
above.

## Licence

Apache-2.0.  See `LICENSE`.
