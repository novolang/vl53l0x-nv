# Changelog

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
  for the nRF52840-DK it is 19,920 B of flash and for the nRF54LM20-DK
  21,512 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  sensor. Neither Nordic board has a driver for its two-wire interface
  (TWIM) at this release, so the device program builds for them and finds
  an absent bus.
- ST publishes no register map. The sequences are those of ST's API as
  Pololu's `vl53l0x-arduino` and the `vl53l0x` Rust crate repeat them; the
  range status is ST's `VL53L0X_get_pal_range_status` less its three limit
  checks.
- Not implemented: the offset and cross-talk calibrations, the reference
  SPAD calibration (the factory's SPAD values are applied), the sigma,
  reference-clip and range-ignore-threshold checks, the interrupt
  thresholds, fractional ranging and the linearity corrective gain.
- `devices-nv` is a path dependency (`../devices-nv`, 0.3.3), which has the
  `Rangefinder` trait and is not on the registry yet. `embedded-hal-nv` is
  the registry's 0.3.2, because a path copy of it conflicts with the
  registry requirement of devices-nv.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module
  also checks at `rt` and `system`.
