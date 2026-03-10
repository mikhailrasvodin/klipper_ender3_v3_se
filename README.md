### Installation instructions

1. Run KIAUH and uninstall klipper (ONLY klipper)
2. In KIAUH folder create file named klipper_repos.txt and add these 2 lines:

```text
https://github.com/mikhailrasvodin/klipper_ender3_v3_se
https://github.com/Klipper3d/klipper
```

3. Then in KIAUH settings change klipper repo to this repos.
4. Install klipper in KIAUH as usual
5. Once installed build firmware as usual from the klipper folder
6. Flash the firmware to printer
7. Create `prtouch.cfg` in your configuration directory. You can use the [template from this repository](prtouch.cfg) as a starting point.
8. In your `printer.cfg`, add the line: `[include prtouch.cfg]`
9. Restart the Klipper service.
10. `PRTOUCH_PROBE_ZOFFSET` should now be available. After running it, the Z-offset will be saved at the end of your `printer.cfg` (or in `SAVE_CONFIG` block).

When running `PRTOUCH_PROBE_ZOFFSET`, always be ready to press the Emergency Stop if anything goes wrong.

---

## PRTouch Configuration

### Configuration Parameters

A complete `prtouch.cfg` requires several sections to function correctly. Below is a full example including the required dependencies:

```ini
[filter]
# Digital filtering for strain gauge data

[dirzctl]
# Direct Z-axis control for probing
# step_base: 2

[hx711s]
# Strain gauge sensor configuration
sensor0_clk_pin: PA4
sensor0_sdo_pin: PC6
count: 1

[prtouch]
# Sensor configuration
sensor_x: 32.0               # X position of the strain gauge sensor
sensor_y: 30.0               # Y position of the strain gauge sensor
sensor_random_offset: 5      # Random offset applied during probing (default: 5mm)

# Measurement thresholds
min_hold: 3000               # Minimum sensor reading threshold (default: 3000)
max_hold: 50000              # Maximum sensor reading threshold (default: 50000)
base_count: 40               # Number of samples for baseline calibration (default: 40)
pi_count: 32                 # Number of samples per probe attempt (default: 32)

# Probing behavior
probe_speed: 2.0             # Z-axis probing speed in mm/s (default: 2.0)
g29_xy_speed: 150            # XY travel speed during bed mesh (default: 150 mm/s)
g29_rdy_speed: 2.5           # Z approach speed (default: 2.5 mm/s)
max_probe_retries: 3         # Maximum retry attempts for probe measurements (default: 3, range: 1-10)
max_accuracy_failures: 3     # Maximum failures for accuracy tests (default: 3, range: 1-10)

# Bed mesh validation
bed_max_err: 4               # Maximum bed error tolerance in mm (default: 4)
check_bed_mesh_max_err: 0.2  # Maximum error for mesh validation (default: 0.2mm)

# Temperature limits
s_hot_min_temp: 140          # Minimum hotend temp for nozzle cleaning (default: 140°C)
s_hot_max_temp: 200          # Maximum hotend temp for nozzle cleaning (default: 200°C)
s_bed_max_temp: 60           # Maximum bed temp for calibration (default: 60°C)

# Nozzle cleaning area
clr_noz_start_x: 15          # X start position of cleaning area
clr_noz_start_y: 20          # Y start position of cleaning area
clr_noz_len_x: 0             # X length of cleaning area
clr_noz_len_y: 15            # Y length of cleaning area (minimum: pa_clr_dis_mm + 5)
pa_clr_dis_mm: 5             # Cleaning distance in mm (default: 5)
pa_clr_down_mm: -0.1         # Z offset during cleaning (default: -0.1mm)
wipe_retract_distance: 0     # Filament retraction distance during wipe (default: 0mm)

# Associated probe
probe_name: bltouch          # Name of the associated probe (default: 'bltouch')

# Debugging
show_msg: False              # Show detailed debug messages (default: False)
```

### Available G-code Commands

#### PRTOUCH_PROBE_ZOFFSET
Calibrates the Z-offset for the nozzle at the PRTouch sensor position.

**Parameters:**
- `CLEAR_NOZZLE=<0|1>` - Perform nozzle cleaning before probing (default: 0)
- `APPLY_Z_ADJUST=<0|1>` - Automatically apply Z-adjust to gcode offset (default: 0)

**Usage:**
```gcode
G28                                    # Home all axes first
PRTOUCH_PROBE_ZOFFSET                  # Basic Z-offset calibration
PRTOUCH_PROBE_ZOFFSET CLEAR_NOZZLE=1   # With nozzle cleaning
PRTOUCH_PROBE_ZOFFSET APPLY_Z_ADJUST=1 # Auto-apply offset
SAVE_CONFIG                            # Save the offset
```

**What it does:**
1. Validates homing status
2. Optionally cleans the nozzle
3. Probes with the standard probe (e.g., BLTouch)
4. Probes with the PRTouch strain gauge sensor
5. Calculates the Z-offset difference
6. Optionally applies the adjustment
7. Saves the configuration

---

#### PRTOUCH_ACCURACY
Tests the repeatability and accuracy of the PRTouch sensor.

**Parameters:**
- `SAMPLES=<count>` - Number of probe samples to take (default: 10)
- `PROBE_SPEED=<speed>` - Probing speed in mm/s (default: from config)
- `MAX_FAILURES=<count>` - Maximum allowed failures (default: 3, range: 1-10)

**Usage:**
```gcode
G28                                   # Home all axes first
PRTOUCH_ACCURACY SAMPLES=20           # Test with 20 samples
PRTOUCH_ACCURACY SAMPLES=50 MAX_FAILURES=5  # Extended test with custom failure limit
```

**Output example:**
```
PRTOUCH_ACCURACY at X:28.50 Y:4.30 (samples=20 speed=2.0 max_failures=3)
probe #1/20 at (28.50, 4.30): 1.523
probe #2/20 at (28.50, 4.30): 1.521
...
probe #20/20 at (28.50, 4.30): 1.522
probe accuracy results: maximum 1.524000, minimum 1.521000, range 0.003000,
average 1.522500, median 1.522500, standard deviation 0.001118
total attempts: 20, successful: 20, failed: 0
```

**Interpretation:**
- **Range** < 0.01mm - Excellent repeatability
- **Standard deviation** < 0.005mm - High precision
- **Failed attempts** - Should be 0 or minimal

---

#### NOZZLE_CLEAR
Performs the nozzle cleaning routine on the bed surface.

**Parameters:**
- `HOT_MIN_TEMP=<temp>` - Minimum hotend temperature (default: from config)
- `HOT_MAX_TEMP=<temp>` - Maximum hotend temperature (default: from config)
- `BED_MAX_TEMP=<temp>` - Maximum bed temperature (default: from config)
- `MIN_HOLD=<value>` - Minimum sensor threshold (default: from config)
- `MAX_HOLD=<value>` - Maximum sensor threshold (default: from config)

**Usage:**
```gcode
NOZZLE_CLEAR                           # Standard cleaning
NOZZLE_CLEAR HOT_MIN_TEMP=150          # Custom temperature
```

**What it does:**
1. Heats the hotend to cleaning temperature
2. Probes the cleaning area corners
3. Performs the wipe motion
4. Optionally retracts filament
5. Cools down

---

### Troubleshooting

#### Error: "PRTouch: First probe failed"
- **Cause:** Sensor did not trigger on first measurement
- **Solutions:**
  - Check sensor wiring (HX711 connections)
  - Verify `min_hold` and `max_hold` thresholds
  - Ensure bed is clean and level
  - Check sensor mounting and strain gauge contact

#### Error: "PRTouch: Second probe failed"
- **Cause:** Inconsistent sensor readings
- **Solutions:**
  - Increase `max_probe_retries` in config
  - Check for mechanical issues (wobble, loose parts)
  - Verify stable temperature (thermal expansion)

#### Error: "PRTouch: Failed to converge after N retries"
- **Cause:** Measurements are not repeatable within tolerance
- **Solutions:**
  - Reduce `check_bed_mesh_max_err` if acceptable
  - Check bed surface (clean, flat, no debris)
  - Verify sensor calibration with `PRTOUCH_ACCURACY`
  - Check for mechanical issues

#### Error: "Too many probe attempts"
- **Cause:** Excessive failures during accuracy test
- **Solutions:**
  - Increase `MAX_FAILURES` parameter
  - Run diagnostic: `PRTOUCH_ACCURACY SAMPLES=5 MAX_FAILURES=10`
  - Check sensor health and wiring

#### Error: "Cannot run hx711s query: MCU is shutdown or timed out"
- **Cause:** MCU communication failure
- **Solutions:**
  - Restart Klipper service
  - Check MCU connection and firmware
  - Verify `[hx711s]` configuration
  - Check for MCU overload (reduce polling frequency)

---

### Tips for Best Results

1. **Initial Calibration:**
   ```gcode
   G28
   PRTOUCH_ACCURACY SAMPLES=50  # Verify sensor stability
   PRTOUCH_PROBE_ZOFFSET        # Calibrate offset
   SAVE_CONFIG
   ```

2. **Regular Maintenance:**
   - Run `PRTOUCH_ACCURACY SAMPLES=20` weekly to verify sensor health
   - Clean the sensor area and bed surface regularly
   - Re-calibrate after any mechanical changes

3. **Optimal Settings:**
   - `probe_speed: 2.0` - Slower is more accurate but takes longer
   - `max_probe_retries: 3` - Good balance between speed and reliability
   - `sensor_random_offset: 5` - Helps detect bed inconsistencies

4. **Debugging:**
   - Set `show_msg: True` temporarily for detailed diagnostic output
   - Check `klippy.log` for detailed error messages
   - Use `PRTOUCH_ACCURACY` to isolate sensor issues

---

### Advanced Configuration Example

```ini
[prtouch]
# High-precision configuration
sensor_x: 28.50
sensor_y: 4.3
sensor_random_offset: 3        # Reduced for consistency
probe_speed: 1.5               # Slower for better accuracy
max_probe_retries: 5           # More retries for difficult beds
check_bed_mesh_max_err: 0.1    # Tighter tolerance
show_msg: False                # Disable unless debugging

# Conservative thresholds
min_hold: 2500                 # Lower threshold for sensitive detection
max_hold: 60000                # Higher ceiling for safety

# Fast configuration (for testing)
# probe_speed: 3.0             # Faster probing
# max_probe_retries: 2         # Fewer retries
# check_bed_mesh_max_err: 0.3  # Looser tolerance
```

---

### Safety Notes

- Always monitor the first few probe attempts
- Keep `EMERGENCY STOP` ready during initial setup
- Never probe with a cold hotend if `CLEAR_NOZZLE=1` is used
- Verify `sensor_x` and `sensor_y` positions are correct before first use
- Test with `PRTOUCH_ACCURACY` before production use
