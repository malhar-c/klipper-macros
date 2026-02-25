## This file details the changes I did for original see @README OG.md

### Change 1: Added `variable_start_park_at_extruder_heating` Control Variable

**Problem:** During the PRINT_START sequence, after bed heating completes, the toolhead would park at the configured park position (Y=230.0) while waiting for the extruder to reach printing temperature. This caused the purge line to be printed at the front of the bed (Y=0), but by the time the print actually started, the bed would move back to Y=0, causing previously oozed filament to contaminate the print surface.

**Solution:** Added a new configuration variable `variable_start_park_at_extruder_heating` to control whether the toolhead parks during the extruder heating phase.

**Files Modified:**

1. **[globals.cfg](globals.cfg#L70)** - Added new global variable with default `True`:
   ```
   # Park toolhead during extruder heating phase (False to keep at current position).
   variable_start_park_at_extruder_heating: True
   ```

2. **[start_end.cfg](start_end.cfg#L232)** - Updated `_print_start_phase_extruder` macro to conditionally park:
   ```gcode
   {% if km.start_park_at_extruder_heating %}
     _KM_PARK_IF_NEEDED HEATER={printer.toolhead.extruder} RANGE=ABOVE
   {% endif %}
   ```

3. **[macros_config.cfg](../macros_config.cfg#L47)** - Override in user config set to `False`:
   ```
   variable_start_park_at_extruder_heating: False
   ```

**Behavior After Change:**
- Y axis stays at position 0 during extruder heating instead of moving to park position (Y=230)
- Purge line prints consistently at Y=0 where nozzle already is
- No bed movement between purge and print start, eliminating oozed filament contamination
- Users can re-enable parking by setting `variable_start_park_at_extruder_heating: True` if desired

### Change 2: Configured `_CLIENT_VARIABLE` for Mainsail PAUSE Macro

**Problem:** When pausing a print, the toolhead would move to the max X position (right side, X=235), where the Klicky probe is magnetically docked. When the X axis reaches max X, it automatically picks up the probe magnetically. Since the probe sits physically below the nozzle, resuming the print with the probe attached would cause the probe to crash directly into the model on the bed, ruining the print and the probe.

**Root Cause:** The Mainsail macro (`mainsail.cfg`) and klipper-macros (`pause_resume_cancel.cfg`) were conflicting. Both define a PAUSE macro, and the Mainsail version was being executed first, using its default park position `(max.x - 5.0)`. This caused the toolhead to travel to max X during pause, which magnetically picked up the probe. The probe would then crash into the print when the print resumed, since the X axis cannot move to max X positions during an active print while the probe is attached.

**How Klicky Probe Works:**
- The probe is magnetically docked at max X position
- When X axis reaches max X, the probe is magnetically picked up and attached to the nozzle assembly
- Macro magic handles undocking the probe after bed meshing by moving away from max X
- The X axis cannot travel to max X during printing (to avoid picking up the probe)

**Solution:** Added `_CLIENT_VARIABLE` macro configuration to `printer.cfg` with custom park coordinates set to X=0 (left side), far away from the probe dock. This prevents the toolhead from traveling to max X during pause, so the probe is never picked up, avoiding the crash during resume.

**Files Modified:**

1. **[printer.cfg](../printer.cfg#L28-L51)** - Added `_CLIENT_VARIABLE` macro:
   ```
   [gcode_macro _CLIENT_VARIABLE]
   variable_use_custom_pos   : True  ; enable custom park coordinates
   variable_custom_park_x    : 0.0   ; park at left side (X=0), away from probe dock at max X
   variable_custom_park_y    : -5.0  ; park at back (Y=-5)
   variable_custom_park_dz   : 2.0   ; lift nozzle 2mm when parking
   variable_speed_move       : 100.0 ; movement speed in mm/s
   # ... other variables with standard defaults
   ```

**Behavior After Change:**
- Toolhead parks at X=0 (left side) when pause is triggered, away from probe dock
- Probe is never magnetically picked up during pause
- Resume operation can proceed safely without probe crashing into the model
- Print is preserved from destruction by errant probe contact
- Can be easily customized by modifying the `_CLIENT_VARIABLE` section in printer.cfg

### Change 3: Fixed Bed Y-Axis Movement During Print Start Heating Phase

**Problem:** During the `_print_start_phase_preheat` phase, after the bed began heating, the toolhead would park using the macro defaults which included moving the bed (Y-axis) forward (to Y=230). This caused the nozzle to be positioned over the bed surface where any oozed filament left in the nozzle from the previous print would drip and contaminate the print surface before the new print even started.

**Solution:** Explicitly set Y=0 coordinate in the PARK command during bed heating to move the bed to the back position (away from the nozzle). This keeps the nozzle clear of the bed surface during heating, preventing oozed filament from contaminating the bed.

**Files Modified:**

1. **[start_end.cfg](start_end.cfg#L114)** - Updated `_print_start_phase_preheat` macro PARK command:
   ```gcode
   # Skip this if the bed was already at target when START_PRINT was called.
   {% if not bed_at_target %}
     PARK P=2 Y=0 Z=10
   ```
   Changed from: `PARK P=2 Z=10` (which would use default Y parking position, moving bed forward)
   Changed to: `PARK P=2 Y=0 Z=10` (moves bed to back position, Y=0)

**Behavior After Change:**
- During bed heating phase, Y=0 moves the bed to the back (away from nozzle)
- Nozzle remains clear of the bed surface while bed stabilizes at target temperature
- Any residual oozed filament drips clear of the bed instead of contaminating it
- Print surface remains clean and ready for the actual print to begin
- Combined with Change 1, ensures no filament contamination during PRINT_START sequence

### Change 4: Configured `variable_start_extruder_probing_temp` for Controlled Nozzle Heat During Mesh Probing

**Problem:** During the bed mesh calibration phase in `_print_start_phase_probing`, the nozzle was heating to the maximum printing temperature and holding it throughout the entire mesh mapping process. For large prints with large bed meshes, the probing can take several minutes. During this time, the nozzle at full printing temperature would continuously ooze melted filament onto the bed surface, contaminating the bed and potentially creating print quality issues or adhesion problems.

**Root Cause:** The klipper-macros system has two modes for extruder heating during probing:
1. **Fixed probing temperature mode** (when `variable_start_extruder_probing_temp > 0`) - Sets nozzle to a specific temperature and waits for it to stabilize, then completes probing before heating to full printing temperature.
2. **Preheat scale mode** (when `variable_start_extruder_probing_temp = 0`, the default) - Heats the nozzle to a fraction of the target temperature (controlled by `variable_start_extruder_preheat_scale`). However, without the fixed temperature override, the system would heat the nozzle too aggressively during probing.

**Solution:** Added `variable_start_extruder_probing_temp` configuration variable set to 175°C, which is significantly lower than typical maximum printing temperatures (PLA: 200-210°C, PETG: 230-240°C, ABS: 240-250°C). This allows the nozzle to reach a controlled, stable temperature suitable for probing operations while preventing continuous oozing during extended mesh calibration.

**Files Modified:**

1. **[macros_config.cfg](../macros_config.cfg#L52-L57)** - Added new user configuration variable:
   ```
   # Set the nozzle temperature during bed mesh probing (prevents oozing).
   # This should be lower than your max printing temperature to avoid filament oozing
   # during the mesh calibration process (especially for large prints with large meshes).
   # Example: if you print at 240°C, set this to 200-210°C
   variable_start_extruder_probing_temp: 175
   ```

**How It Works (from klipper-macros source code):**

In `_print_start_phase_probing` macro:
- If `variable_start_extruder_probing_temp > 0`, the nozzle heats to and stabilizes at that specific temperature (175°C in this case)
- Mesh probing occurs while nozzle is at this controlled temperature
- After probing completes, nozzle immediately heats to the full target printing temperature (M104 S{EXTRUDER})
- No oozing occurs during the typically several-minute-long mesh calibration process

**Customization Guide:**
Adjust `variable_start_extruder_probing_temp` based on your typical printing temperature:
- **PLA prints (200-210°C)**: Set to 170-180°C
- **PETG prints (230-240°C)**: Set to 200-210°C  
- **ABS prints (240-250°C)**: Set to 210-220°C

**Behavior After Change:**
- Nozzle heats to 200°C during bed mesh probing phase and stabilizes
- Extended mesh calibration can proceed without nozzle oozing filament onto the bed
- After probing completes, nozzle heats to full target temperature for printing
- Bed surface remains clean and free of oozed filament residue
- Especially beneficial for large prints where mesh probing can take 5-10+ minutes
- Print adhesion and first-layer quality remain consistent without filament contamination

### Change 5: Integrated `STATUS_*` LED macros into PRINT_START and FILAMENT CHANGE phases

**What I changed:**
- Added direct calls to the `STATUS_*` macros inside key PRINT_START phases so the LED effects run automatically during start-up (these use the `argb_lights.cfg` status macros which trigger `SET_LED_EFFECT ... REPLACE=1`).

**Files modified:**
1. `start_end.cfg` 
    - inserted calls to the `STATUS` macros in these locations:
    - After homing: `STATUS_HOMING` (runs immediately when homing starts)
    - When bed heating begins: `STATUS_HEATING`
    - Before extruder probing and during extruder heat: `STATUS_HEATING`
    - When mesh/leveling starts: `STATUS_MESHING`
    - After extruder preheat phase: `STATUS_HEATING` in `_print_start_phase_extruder`
2. `filament.cfg` 
    - added `STATUS_FILAMENT_LOAD` and `STATUS_FILAMENT_UNLOAD` to the start of the LOAD_FILAMENT and UNLOAD_FILAMENT macros respectively.
    - added `STATUS_READY` and `STATUS_ERROR` to the end of the LOAD_FILAMENT and UNLOAD_FILAMENT macros respectively.

**Why:**
- Ensures the `led_effect` animations (configured in `argb_lights.cfg`) are started at the appropriate PRINT_START phases without relying on slicer start-gcode.
- Uses `REPLACE=1` in the status macros to cleanly switch effects.

**Behavior After Change:**
- LED effects now automatically run during PRINT_START phases
- No need to add `SET_LED_EFFECT` commands in slicer start-gcode
- LED animations are synchronized with printer activities (homing, heating, meshing)
- Clean transitions between different LED effects during the start-up sequence

```

