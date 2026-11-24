# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Vl53l0xPort` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Vl53l0xPort` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Vl53l0xPort` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Vl53l0xPort for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `rd`, `rd16`, `wr`, `wr16`, `upd`, `irq_level_of`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Vl53l0xPort` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The register writes of one byte and of 16 bits, `wr` and `wr16`, are `write_reg` and `write_reg16_be`; the 32-bit `wr32` stays in the module.
- `i2c_setup` stays, with the shutdown pin it sets; it makes the shutdown pin an output before the interrupt pin an input, the order the shared `wiring_i2c_setup` leaves it in, and calls it for the bus.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 35,888 bytes before and 35,440 after on the range reader (nrf52-qemu: 15,944 and 15,464).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed.  `examples/range_reader.nv` adds the `RangeBlinded` and `RangeSaturated` arms that devices-nv 0.4.0 gives `RangeStatus`.

The device program builds for the boards its README names, flash nrf52840-dk 35,912 B before and 35,532 B after; nrf54lm20-dk 37,392 B before and 37,012 B after; nrf52-qemu 15,968 B before and 15,620 B after.

## 0.1.0

The first release: a driver for ST's VL53L0X time-of-flight ranging sensor
on an I2C bus.

- `Vl53l0x`, the type a board's device table builds from the board's I2C
  bus, timer and GPIO handles and the wiring, implementing devices-nv's
  `Rangefinder` and `Identifiable`. `Vl53l0xPort` is the trait a test or a
  program implements to run the module's functions over its own bus.
- The initialisation as named steps in ST's order: the model ID check, the
  data initialisation, the reference SPAD count and type from the part's
  NVM applied, the 80-pair tuning table, the interrupt configuration, the
  timing budget re-applied with MSRC and TCC off, and the VHV and phase
  reference calibrations.
- The timing budget, the final-range signal-rate limit and the VCSEL pulse
  periods, with the API's arithmetic for macro periods and timeouts.
- Single-shot and continuous ranging, back to back or timed, and the stop
  variable kept across the stop of continuous ranging.
- The range in millimetres with a status: ST's device status codes mapped
  onto `RangeStatus`, and 8190 mm and above read as out of range.
- The default, high-accuracy, long-range and high-speed profiles of
  UM2039 Table 6, as `Vl53Profile`.
- A cycle of the XSHUT pin, a change of the I2C address, GPIO1 and the data
  ready flag.
- The device program `examples/range_reader.nv`, a distance reader. Built
  for the nRF52840-DK it is 39,688 B of flash and for the nRF54LM20-DK
  41,488 B; before the Nordic buses landed, with the reading path left out,
  it was 19,920 B and 21,512 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  sensor. The nRF52840 DK drives I2C with TWIM0 and the nRF54LM20 DK with
  TWIM22, so the device program's images carry the reading path; a reply from
  the sensor over either bus is NOT VERIFIED.
- ST publishes no register map. The sequences are those of ST's API as
  Pololu's `vl53l0x-arduino` and the `vl53l0x` Rust crate repeat them; the
  range status is ST's `VL53L0X_get_pal_range_status` less its three limit
  checks.
- Not implemented: the offset and cross-talk calibrations, the reference
  SPAD calibration (the factory's SPAD values are applied), the sigma,
  reference-clip and range-ignore-threshold checks, the interrupt
  thresholds, fractional ranging and the linearity corrective gain.
- `devices-nv` is the registry range `^0.3.3`, the first release with the
  `Rangefinder` trait. `embedded-hal-nv` is the registry range `^0.3.2`.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module
  also checks at `rt` and `system`.
