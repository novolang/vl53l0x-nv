# vl53l0x-nv

A driver for ST's VL53L0X, a time-of-flight ranging sensor with an I2C
interface, specified in the datasheet "VL53L0X: World smallest
Time-of-Flight ranging and gesture detection sensor", DocID029104 Rev 2,
April 2018 (ST's DS11555), and in the API user manual "VL53L0X API
specification", UM2039 (DocID029105 Rev 1, June 2016). ST publishes no
register map for the part; the initialisation sequence and the tuning
registers are those of ST's API as two open drivers repeat them (see
"Sources"). The driver implements the `Rangefinder` and `Identifiable`
traits of [devices-nv](https://novo-lang.org/packages/devices-nv) and
reads the distance in millimetres with ST's range status.

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this sensor, so the driver has run only against a double of the part, in
the host suite. The device program below builds for the nRF52840 DK and
the nRF54LM20 DK and has not been flashed.

## The device program

`examples/range_reader.nv` brings the sensor up, selects the default
profile and prints a distance about ten times a second over the board's
UART:

```
range 523 mm
out of range
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio, BoardTimer }
use dev_range.{ RangeReading, Rangefinder }
use dev_bus
use vl53l0x.{ Vl53l0x, Vl53Profile }

// Writes `n`, 0 or more, in decimal.  Formatting into a string would
// allocate, which @tier(embedded) refuses, so each digit goes out as a
// byte.  An Int has at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + (n / div) % 10)
            div = div / 10

// Writes the line for a reading.
@tier(embedded)
fn report(r: RangeReading) [hw]
    match r.status
        RangeValid =>
            hal.uart.write("range ")
            digits(r.mm)
            hal.uart.write(" mm\n")
        RangeWeak =>
            hal.uart.write("weak ")
            digits(r.mm)
            hal.uart.write(" mm\n")
        RangeOutOfRange => hal.uart.write("out of range\n")
        RangeNoTarget => hal.uart.write("no target\n")
        RangeFault => hal.uart.write("sensor fault\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    var sensor = Vl53l0x.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, BoardGpio { id: 0 },
                                dev_bus.wiring_i2c(0, 0x29))
    // The sensor needs 1.2 ms after its supply; the program waits 5.
    hal.timer.delay_ms(5)
    if not sensor.begin()
        hal.uart.write("no answer from the sensor at 0x29\n")
    let _ = sensor.set_profile(Vl53Profile.Vl53Default)
    while true
        if sensor.range_start() < 0
            hal.uart.write("no answer\n")
        else
            match sensor.range_read()
                Some(r) => report(r)
                None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The program is `examples/range_reader.nv`. A board package that depends on
this package builds it as its `main`. `tests/device_build.sh` does so from a
scratch package for the two boards below and for the emulated board, where
it runs the image and checks that the program reports that the sensor does
not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 19,920 B | 7,040 B | 2,644 B |
| `nrf54lm20-dk` | 21,512 B | 7,626 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables; the
program includes the 160-byte tuning table and the driver's code for the
whole initialisation sequence. The arena, 65,536 B on both boards, is the
runtime's heap and is reserved whether or not the program uses it; the
driver allocates nothing in it. The whole of the driver's state is a
`Vl53l0x` value of 112 bytes in the program's frame.

Neither Nordic board package names a driver for its two-wire interface
(`@hal(i2c)`), so `BoardI2c` is absent hardware on both: every transfer is
refused, the program prints `no answer from the sensor at 0x29`, and the
build removes the code that follows the first bus read, which makes the
rows above smaller than the image of a board with a bus.

The wiring: SDA and SCL to the board's first I2C bus, the breakout's
supply to 3.3 V, and its XSHUT and GPIO1 pins either open (the breakouts
pull XSHUT up) or on two free pins of the board, which the wiring record
names.

### The effects

Every function of the driver that reaches the bus, a pin or the timer
declares `[hw]`, and nothing else. The driver performs no input or output
of its own and allocates nothing of its own. The bus functions of devices-nv
build one short frame per transaction inside a `system` block and release it
when the call returns. The module checks at every tier, `embedded`
included. Bringing the part up makes 157 bus transactions, which take about
20 ms at 400 kHz besides the calibrations; a measurement spins the core, polling the
part, for the length of its timing budget.

## What the sensor is

The VL53L0X is a laser rangefinder in a 4.4 x 2.4 x 1.0 mm package. A
940 nm VCSEL sends pulses, an array of single-photon avalanche diodes
(SPADs) counts the light that returns, and the part times the return
against a reference path, so the distance does not depend on the target's
reflectance. It ranges to about 2 m. It has an I2C interface at up to
400 kbit/s on the address 0x29 (the datasheet writes 0x52), a shutdown pin
XSHUT, active low, and an interrupt pin GPIO1. It boots in 1.2 ms after
XSHUT rises or the supply comes up.

Three registers can be read to validate the bus (datasheet Table 4): 0xC0
reads 0xEE, 0xC1 reads 0xAA, 0xC2 reads 0x10. Multi-byte values are most
significant byte first at ascending addresses.

A measurement is a sequence of steps: dynamic SPAD selection (DSS), the
minimum signal rate check (MSRC), the target centre check (TCC), the
pre-range and the final range. The driver runs the steps the API runs by
default: DSS, pre-range and final range, with MSRC and TCC off. The
**timing budget** is the time allowed for the sequence, 33 ms by default
and at least 20 ms; the final range takes what the other steps leave, and
a budget N times longer cuts the standard deviation of the result by the
square root of N (UM2039, "Overall timing budget").

### The initialisation

`start` runs the steps of ST's `VL53L0X_DataInit`, `StaticInit` and
`PerformRefCalibration`, in this order:

| Step | What it does |
|---|---|
| model check | reads 0xC0 and wants 0xEE |
| data initialisation (`data_init`) | 2.8 V I/O if asked (bit 0 of 0x89), I2C standard mode (0x88), the stop variable read from 0x91 on page 1, the signal-rate and pre-range limit checks switched off (0x60), the final range's signal-rate limit at 0.25 MCPS, every sequence step enabled |
| reference SPADs (`spad_info`, `ref_spads_apply`) | reads the count and type of the reference SPADs from the part's NVM through pages 6 and 7, and enables that many from the first SPAD of that type (12 for the aperture type, 0 for the other) |
| tuning (`tuning_load`) | writes the 80 register pairs of ST's default tuning table, in order |
| interrupt (`gpio_config`) | GPIO1 for "new sample ready", active low, interrupt cleared |
| budget | reads the budget, switches MSRC and TCC off (sequence 0xE8), applies the budget again |
| reference calibration (`ref_calibrate`) | the VHV calibration, then the phase calibration, then the sequence back |

The tuning table is `TUNING`, 160 integers, the register and the value of
each write, as the API lists them. A write to 0xFF selects a page of the
register file.

### The range status

The part reports a status in bits 6:3 of the first result byte. The driver
maps it with ST's table (the API's `VL53L0X_get_pal_range_status`) onto
the `RangeStatus` of devices-nv:

| Device code | ST name | Status | Distance |
|---|---|---|---|
| 11 | range complete | `RangeValid` | the range |
| 4 | MSRC no target (signal fail) | `RangeWeak` | the range |
| 6, 9 | phase check, phase consistency (wrap-around or noise) | `RangeOutOfRange` | 0 |
| 8, 10 | target centre check, minimum clip | `RangeOutOfRange` | 0 |
| 1, 2, 3 | VCSEL continuity, VCSEL watchdog, no VHV value | `RangeFault` | 0 |
| 0, 5, 7, 12 to 15 | no update | `RangeNoTarget` | 0 |

A range of 8190 mm or more, which the part reports when it sees nothing,
is `RangeOutOfRange` whatever the code says.

### The profiles

`Vl53Profile` holds the four profiles of UM2039 Table 6.

| Profile | Signal-rate limit | Budget | VCSEL periods (pre, final) | Typical range |
|---|---|---|---|---|
| `Vl53Default` | 0.25 MCPS | 33 ms | 14, 10 | 1.2 m |
| `Vl53HighAccuracy` | 0.25 MCPS | 200 ms | 14, 10 | 1.2 m |
| `Vl53LongRange` | 0.1 MCPS | 33 ms | 18, 14 | 2 m, in the dark |
| `Vl53HighSpeed` | 0.25 MCPS | 20 ms | 14, 10 | 1.2 m |

The manual's sigma limits for these profiles (18, 60 and 32 mm) are not
applied, because the sigma check is not implemented. The budget a profile
sets is rounded to what the timeout registers hold, so the budget read
back is up to 1 % below the budget asked for.

## What the package contains

| Module | What is in it |
|---|---|
| `vl53l0x` | The type `Vl53l0x` that holds a board's bus, timer and pins, the trait `Vl53l0xPort`, the bring-up as named functions, the timing arithmetic of the API, the profiles, the measurements and the range status |

## How to choose an entry point

A program on a board uses `Vl53l0x`, built by `attach` with the board's
I2C bus, timer and GPIO and the wiring, as a board's device table builds
it. The wiring's `en` is the XSHUT pin and its `irq` is GPIO1. A program
keeps the sensor in a `var`: bringing it up replaces the state it holds,
the stop variable.

| Method | What it does |
|---|---|
| `attach(bus, timer, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()`, `begin_io(io_2v8)` | cycles XSHUT if the wiring has the pin, and runs `start`; `begin` sets 2.8 V I/O; whether the part came up |
| `set_profile(p)` | one of the four profiles |
| `timing_budget_us()`, `set_timing_budget_us(us)` | the budget; 20 ms to 1 s |
| `signal_rate_limit_mmcps()`, `set_signal_rate_limit_mmcps(v)` | the limit in thousandths of a MCPS, 0 to 511.99 MCPS |
| `vcsel_period(final_range)` | the pre-range or final-range period in PCLKs |
| `range_start()` | `Rangefinder`: starts one single-shot measurement and answers the budget in microseconds; 0 when continuous ranging runs; negative when the part did not take it |
| `range_read()` | `Rangefinder`: waits for the result, at most 1.1 s, and answers the distance with its status; `None` when the part did not answer |
| `range_max_mm()` | `Rangefinder`: 2000 for the long-range periods, 1200 otherwise |
| `measure_now()` | start, wait and read in one call |
| `continuous_begin(period_ms)`, `continuous_end()` | continuous ranging, back to back for a period of 0; stop |
| `data_ready()`, `gpio1_level()` | the result flag, and the level of GPIO1 |
| `move_to(addr)` | gives the part another I2C address, kept until the next power-up or XSHUT |
| `device_id()` | `Identifiable`: 0xEE, with the revision register as `rev` |
| `reg(r)` | one register |

A program that holds its own bus, or a test with a double of the part,
implements `Vl53l0xPort` over its bus and calls the module's functions,
which take the port: `vl53l0x.start(port, true)`,
`vl53l0x.measure(port, stop_var)`. The framing is `i2c_read_into` and
`i2c_write_from`, generic over `I2cBus`.

## The rules a user needs

1. The part boots in 1.2 ms. `begin` waits 2 ms after each edge of XSHUT
   when the wiring has the pin and does not wait after the supply; the
   program waits.
2. After XSHUT the part is back at 0x29. Several sensors on one bus start
   with all XSHUT pins low, then raise one, `begin` it, `move_to` another
   address, and go on to the next.
3. `begin` sets the I/O to 2.8 V, which the common breakouts expect;
   `begin_io(false)` leaves the part at 1.8 V I/O. A wrong level makes the
   part stop answering on the bus.
4. `range_start` does not wait. It answers the budget, `range_read` waits
   for the result, and `range_read_after(waited_us, measure_us)` of the
   trait refuses a read that comes early. In continuous ranging
   `range_start` answers 0 and starts nothing.
5. The flag that says a result waits is cleared before a single
   measurement starts and after a result is read, so a result that was
   never read is not taken for the next one.
6. Stopping continuous ranging zeroes the stop variable in the part, and
   the driver keeps its own copy to write back before the next
   measurement.
7. A change of a VCSEL period runs a phase calibration, which takes a few
   milliseconds; `set_profile` changes the periods only when the profile
   needs other ones.
8. A reading with `RangeWeak` has a distance and a warning: the return was
   below the limit the part checks. `RangeOutOfRange`, `RangeNoTarget` and
   `RangeFault` have no distance.
9. The 8190 mm value is the part's "nothing there", not a distance.
10. A cover glass or a window in front of the sensor needs a cross-talk
    and an offset calibration, which are not implemented.

## What is not included

NOT IMPLEMENTED:

- The offset calibration and the cross-talk calibration of the API
  (`VL53L0X_PerformOffsetCalibration`, `VL53L0X_PerformXTalkCalibration`)
  and the registers that hold their results. A part behind a cover glass
  reads with an offset until they exist.
- The reference SPAD calibration, `VL53L0X_PerformRefSpadManagement`, the
  search that picks the reference SPADs on site. The driver applies the
  count and type the factory stored in the part's NVM, which the open
  drivers do as well. The calibration is needed under a cover glass.
- The sigma-estimate check, the reference-clip check and the
  range-ignore-threshold check of the API's status. The status here is the
  part's own code mapped through ST's table; the limit checks the API adds
  on top (it enables the sigma check by default) are not made, so a noisy
  reading the API would mark "sigma fail" is `RangeValid` here when the
  part's own code says range complete.
- Interrupt thresholds (low, high, window) and the GPIO1 modes besides
  "new sample ready".
- Fractional ranging (`SYSTEM_RANGE_CONFIG`) and the linearity corrective
  gain.
- The part's signal-rate and ambient-rate result registers, which the
  result read fetches and does not report.
- The async form of `Rangefinder`: a handle implements no trait spelled
  with `async fn` (SPEC section 14.5).

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `Rangefinder` and `Identifiable` traits this driver implements, the
  wiring record and the register helpers.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `GpioOut` and `Timer` traits and the board handles.
- [hcsr04-nv](https://novo-lang.org/packages/hcsr04-nv) drives the HC-SR04
  ultrasonic module as the same `Rangefinder`.

## Tests

`novo test` runs the module over a double of the part: a paged register
file with the model ID and reference registers of the datasheet, the SPAD
handshake of pages 6 and 7, the interrupt flag that a ranging or a
calibration raises after some polls, and a moveable I2C address. The
suite checks the framing of the bus, every named step of the
initialisation against the register writes in order, the 80 pairs of the
tuning table, the timing arithmetic (macro periods, timeout encoding, the
budget with each step on and off, and a budget written and read back), the
signal-rate limit, every VCSEL period, the calibration and its timeouts,
every status code of the range, single and continuous ranging, the four
profiles, and the type over the host's board handles. The initialisation,
the SPAD map, the calibration, the profiles and the starts each run once
for every transaction of a good run with that transaction failing, and
must answer that they failed, so no bus error is read as data. The suite
covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

## Sources

The sources read for this driver, for their fixes and not ported, are the
datasheet and the user manual above, Pololu's `vl53l0x-arduino` (the
sequence and the tuning table, MIT licence), the `vl53l0x` Rust crate of
copterust, Zephyr's `drivers/sensor/st/vl53l0x` (which wraps ST's library
and reports the status the same way), and ST's
`VL53L0X_get_pal_range_status` in the copy that stm32duino/VL53L0X
carries, for the range status. The pitfalls taken from them are named in
`src/vl53l0x.nv` at the sites that avoid them:

- Pololu's `init` returns from the SPAD handshake's timeout with the page
  register at 0x07 and the pages powered; here every page entered is left
  on every path.
- Both drivers read a register without checking the transfer, and Pololu's
  timeout is off by default; here every transfer is checked and every wait
  ends.
- Both drivers answer the range and drop the status, so 8190 mm reads as a
  distance; here the status is decoded.
- A result that was not read keeps the interrupt flag; here the flag is
  cleared before a measurement starts.
- The stop variable at 0x91 is zeroed by stopping continuous ranging; here
  the driver keeps it.

## Licence

Apache-2.0.
