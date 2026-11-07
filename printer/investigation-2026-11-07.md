# Ender-3 V3 KE investigation, 2026-11-07

**Answer:** the printer is healthy, with clean logs and stable temperatures. Tuning will not stick until two things are dealt with:

- **Creality's software overrides pressure advance** at print start.
- **Creality's software rewrites the saved Z offset and mesh** before every print. Your Orca start G-code then probes its own 5x5 mesh on top.

The bed also tilts by about 1.3 mm, and the camera cannot judge fine print quality.

Evidence labels: **measured** was read directly. **Inferred** is my reading of that evidence. **From memory** was not checked against a source today.

Nothing was heated, moved, restarted or edited by this investigation. A print was running for most of it and finished normally.

## Access and services

**Moonraker is there, and remote control looks possible without a login. I did not test any control command.**

| Item | Value |
|---|---|
| Firmware | V1.1.0.17, `printer.cfg` header says v1.2.12 |
| Klipper | `09faed31-dirty` (Creality fork), state ready |
| MCU firmware | `38d96adc-dirty` (2023-10-16), host MCU `v0.11.0-372` |
| Kernel | Linux 4.4.94 mips, Buildroot 2020.02.1 |
| Disk | `/usr/data` 47% used, 2.9 GB free. `/rom` at 100% is normal |
| SSH | `dropbear_2019.78`: rejected ed25519, accepted an ECDSA key |

| Port | Service | Result |
|---|---|---|
| 7125 | Moonraker v0.9.3-123 | Up, 200 |
| 4408 | Fluidd (nginx) | Up, 200 |
| 4409 | Mainsail (nginx) | Up, 200 |
| 80 | Creality web UI | Up, 200 |
| 9999 | Creality web-server | Listening, `/` returns 404 |
| 8080 | mjpg_streamer | Up, snapshot works |

**To control prints remotely:**
- **Already in place:** Moonraker trusts `192.168.0.0/16`, so no API key should be needed from your LAN.
- **Endpoints:** `/server/files/upload`, `/printer/print/start` and `/printer/gcode/script`.
- **Orca:** currently uses `host_type: crealityprint`, not Moonraker.

**Also running:** OctoEverywhere, so the printer is reachable from outside your LAN too.

**Access risk:** the root password is still a published Creality default. Change it once tuning is done.

## Config problems, most important first

### 1. Pressure advance is set by Creality, not by you (PETG)
- **Measured:** three different values exist.
  - `printer.cfg`: 0.036 (factory)
  - `Overture PETG` preset: 0.05, with `enable_pressure_advance = 0`
  - Live during the print: **0.07**
- **Measured:** the print's G-code contains no `M900` or `SET_PRESSURE_ADVANCE`.
- **Measured:** PA changed to 0.07 at 15:44:16 UTC, as the print started.
- **Measured:** Creality's `print_para_config.json` lists PETG `optimal` as 0.07.
- **Measured:** the `master-server` binary contains `SET_PRESSURE_ADVANCE`.
- **Inferred:** `master-server` sets PA by material type at print start.
- **Inferred:** with PA enabled in Orca, its `M900` should run later and win. Your PLA Pro and Matte PLA presets have it enabled. I did not observe a PLA print.
- **Other table values:** PLA 0.044, ABS 0.05, TPU 0.09, Silk-PLA 0.054, other 0.044.

### 2. Z offset and mesh are rewritten before each print
- **Measured:** `printer.cfg` was saved at 09:40:42 and 09:44:16 (printer log time), and the print started at 09:44:20.
- **Measured:** 251 timestamped `printer-*.cfg` backups exist, mostly in pairs a few minutes apart.
- **Measured:** the last four saved Z offsets were 2.267, 2.275, 2.239 and 2.285, a 46 micron range.
- **Measured:** a stage ran first at bed 60°C, nozzle 180 then 200°C. This matches `[z_compensate]` (`bed_add_temp: 60`, `hot_rub_temp: 200`).
- **Measured:** the Orca start G-code then heats the bed to 85°C and runs `BED_MESH_CALIBRATE PROBE_COUNT=5,5`, before heating the nozzle to 260°C. The live mesh during the print was that 5x5.
- **Measured:** `printer.cfg` holds a 10x10 `default` mesh, but the backup `printer-20261107_094416.cfg` holds a 5-row one.
- **Inferred:** Z offset is measured well below printing temperature, so it can differ from the real first-layer height.
- **Inferred:** the 10x10 mesh is replaced by the 5x5 for the print, so the time spent probing it may be wasted.
- **Inferred:** Creality's save step sometimes writes the 5x5 back as `default`.
- **Unknown:** what triggers the pre-print stage. Suspects are Creality's print path or a leveling option.

**Tuning impact:** a manual Z offset may not last, because the next calibration can replace it. Any unsaved change could also be written by the next auto-save.

### 3. Bed tilt is large
- **Measured:** the live 5x5 mesh at 85°C spans **1.33 mm**, from -1.015 to +0.318.

```
Mesh height in mm (front row at bottom)
rear   -1.015  -0.812  -0.588  -0.330  -0.087
       -0.642  -0.475  -0.285  -0.105   0.083
       -0.275  -0.147  -0.010   0.130   0.220
        0.050   0.125   0.198   0.255   0.307
front   0.237   0.287   0.305   0.312   0.318
        left                             right
```

- **Measured:** the rear-left corner is lowest.
- **Measured:** the 10x10 spread was 0.607 mm in October 2025 and has been about 1.2 to 1.4 mm since November 2025.
- **Inferred:** the mesh hides this, but Z moves constantly in the first 10 mm (`fade_end: 10`). Worth correcting mechanically before first-layer tuning.

### 4. Y shaper may not support your 5000 mm/s² accel
- **Measured:** X is `2hump_ei` at 62.2 Hz. Y is `ei` at 35.2 Hz. Both were last saved in the backup dated 2025-12-12.
- **Measured:** presets use 5000 for inner walls, infill and travel.
- **From memory:** an `ei` shaper at 35 Hz suggests about 4000 mm/s² maximum. Confirm with the next shaper calibration output.

### 5. Hotend settings are still factory
- **Sensor:** `EPCOS 100K B57560G104F`, same as factory. **Measured** from config. I cannot confirm from here that this matches the ceramic hotend's thermistor.
- **max_temp:** 320, same as factory. Check it against your hotend's rating.
- **min_extrude_temp:** not set, so the default 170 applies. Fine.
- **PID:** 20.584 / 1.737 / 60.981, same as factory and unchanged in all 251 backups. It holds well anyway: 265.0 mean, 0.24 standard deviation, range 264.5 to 265.6 over the sample.
- **rotation_distance:** 7.53, factory and unchanged in all backups.
- **Verdict:** a PID retune is optional. Check rotation distance before flow tuning.

### 6. Macros that change what you think you set
- **Fan scaling (`M106`, measured):** your start G-code sets `fan0_min` to 30, so Orca 30% gives about 38% and Orca 50% gives about 56%.
- **After a restart:** `fan0_min` is 140 until a print's start G-code runs, so any non-zero `M106` gives at least 55%.
- **`FINISH_INIT` (runs on cancel):** resets PA to config, corner velocity to 5, accel to 5000 and accel-to-decel to 5000.

### 7. Smaller items
- **No idle cooldown:** `idle_timeout: 99999999`, so heaters left on stay on. This is a safety gap.
- **Stale camera entry:** `moonraker.conf` points at `192.168.1.218`. The printer is `.217`.
- **File newer than Klipper's loaded copy:** Klipper parsed Z offset 2.275 at its 03:48 start, and the file now says 2.285. The value actually in use can't be read from this, as Creality's code may apply offsets in memory.
- **Log clock:** printer log times are 6 hours behind UTC.

### Could be overwritten by updates
- **Inferred, not tested:** a firmware update or factory reset may restore `factory_printer.cfg`. That would drop the Helper Script include and your saved shaper values.
- **Macros:** `gcode_macro.cfg` and `sensorless.cfg` are Creality files dated 2024, and could also be replaced.
- **Safety net:** keep this backup.

## Accelerometer status

**No USB accelerometer is present. An SPI accelerometer is configured, but whether it's connected, and where it's mounted, is unverified.**

- **Measured:** `lsusb` shows only the hub and the C170 camera.
- **Measured:** no `/dev/serial/by-id`, `/dev/ttyUSB*` or `/dev/ttyACM*`.
- **Measured:** no extra `[mcu]` for a USB sensor. The only extra one is `[mcu rpi]`, the host.
- **Measured:** one `[adxl345]` on host SPI (`spidev2.0`, `axes_map: z,y,x`), and `accel_chip: adxl345` for both axes.
- **Measured:** `~/Downloads` on the Mac holds a Creality `G-Sensor` manual and bracket model.
- **Possibility:** your bed sensor may be that G-sensor on SPI. If so, the USB checks could never show it, so they don't mean it's unfitted.
- **Not established:** `/dev/spidev2.0` exists whether or not a chip is connected. `ACCELEROMETER_QUERY` would settle whether one responds, but not where it's mounted. I did not run it.
- **Note:** a USB sensor would need its own config section, plus a Klipper restart.

## Camera

**Good for a rough look only. It cannot judge first layers, PA patterns or flow surfaces.**

- **Hardware:** Logitech C170, on `/dev/video4`.
- **Streamer:** `mjpg_streamer` on port 8080, with no resolution argument.
- **Snapshot URLs:**
  - `http://192.168.1.217:8080/?action=snapshot` works
  - `http://192.168.1.217:4408/webcam/?action=snapshot` works
  - `http://192.168.1.217/webcam/?action=snapshot` returns 404
- **Snapshot resolution:** 640x480, measured.
- **Maximum listed:** 1024x768. **From memory:** the C170's native video is 640x480, so this is probably upscaled.
- **Scale:** about **0.6 mm/px** at the bed, plus or minus 0.1. Estimated from the bed edges and the 46 mm print in one bed position (Y about 110).

**What the picture shows:**
- **Bed:** spans about x=78 to x=465, so roughly 40% of the width is not bed.
- **Blocked view:** the toolhead covered most of the print in the snapshots, and the gantry cuts across the view.
- **Exposure:** the bright wall behind is blown out.

| Job | Verdict | Why |
|---|---|---|
| Failure watching | Marginal | Big failures are visible unless they are behind the head |
| First layer | No | A 0.45 mm line is under 1 pixel |
| Pressure advance patterns | No | About 6 times too coarse for 0.1 mm/px |
| Flow surfaces | No | Can't rank surfaces; only gross defects would show |

**What to change:**
1. **Now:** use phone photos for PA and flow. This costs nothing and has far more detail than the C170.
2. **Move it:** put it forward and to one side, angled in so the head and gantry don't block the bed.
3. **Light it:** turn it away from the bright wall and add a light over the bed.
4. **Replace it if you want 0.1 mm/px:**
   - 1080p framed tightly on a 150 mm patch gives about 0.08 mm/px.
   - Across the whole bed, 0.1 mm/px needs about 4K.
   - It needs autofocus or close focus.
   - Not tested: load on the printer's small CPU. A separate host may be safer.

## Log problems

**No thermal or MCU faults in the 6 klippy logs checked (29 October to today).**

| Check | Count |
|---|---|
| `Timer too close` | 0 |
| `Lost communication` | 0 |
| `ADC out of range` | 0 |
| `not heating at expected rate` | 0 |
| Shutdown transitions | 1 |

- **The shutdown:** a `FIRMWARE_RESTART` at 03:48:36 (log time) hit an internal error during restart, then started normally. The restart itself did not come from this investigation. Probably harmless, and worth watching if it recurs.
- **Startup warnings:** 13 `got {...}` lines at startup. Normal noise.
- **MCU link:** healthy during the print. `bytes_retransmit=73` of about 27 MB, `bytes_invalid=0`, `print_stall=0`.
- **Load:** sysload 1.78 on 2 cores while printing.
- **Wi-Fi:** `wl_run_escan` fills `dmesg` about every 30 seconds. Not linked to a print problem, though it could explain stream or network jitter.

## Conflicts with the Orca presets

| Setting | Printer | Preset | Effect |
|---|---|---|---|
| PETG pressure advance | 0.036 config, 0.07 live | 0.05, disabled | **Conflict:** preset value not used |
| PLA pressure advance | 0.036 config | 0.033 Pro, 0.014 Matte, enabled | Set in both, preset should win |
| Bed mesh | 10x10 saved by the pre-print stage | Start G-code probes 5x5 | **Conflict:** the print uses the 5x5, and the saved mesh may be overwritten |
| Accel | `max_accel` 8000 | 5000 | Printer allows it, Y shaper may not |
| Accel to decel | 2500 config | Start G-code sets 2500 | Consistent, but a cancel sets 5000 until the next start |
| Corner velocity | 5 config | Jerk 7, live 7 | Preset wins |
| Fan | Scaled by `M106` | 30 to 50% for PETG | Real speed is higher |
| Input shaping | Saved values | `input_shaping_emit = 0` | No conflict |
| Print area | 220 x 220 | 220 x 220 | No conflict |

**Limits lower than the presets ask for:** none found.

**Preset problems seen in the last print's G-code:**
- **Wrong process:** the PETG job used `print_settings_id = PLA_0.4_Standard_0.2`, though `PETG_0.4_Standard_0.2` exists. It got 65 mm/s outer walls, not 55.
- **Temperature out of range:** the PETG preset uses 265°C, while its own range tops out at 260°C. Check this on a temperature tower.

## Recommended next steps, in order

1. **Make Orca own pressure advance.** Enable PA in `Overture PETG`, then confirm the live value on the next print. No printer change needed.
2. **Pick the right process.** Use a PETG process for PETG jobs.
3. **Find the pre-print calibration trigger.** Try one print sent through Moonraker, then check whether a new pair of `printer-*.cfg` backups appears.
4. **Correct the bed tilt mechanically,** then re-mesh. Needs homing and heat, so your approval.
5. **Confirm which accelerometer you have** with `ACCELEROMETER_QUERY`, then re-run shaper calibration and compare suggested accel with 5000. This moves axes and saves config.
6. **Check rotation distance,** then optionally PID at 265°C. Needs heat and extrusion.
7. **Run a temperature tower,** then PA and flow, judged with phone photos.
8. **Improve the camera.**
9. **Housekeeping:** fix the stale camera IP, set a sensible idle timeout, change the root password. These edit the printer.

## Backup

- **Location:** `printer/config-backup/2026-11-07/`, on branch `printer-config-backup`.
- **Contents:** all 264 regular files, with checksums matching the printer.
- **Symlink:** `Helper-Script/improved-shapers/improved-shapers.cfg` is a symlink on the printer. The backup holds a real copy of the file, also with a matching checksum.
- **Credentials:** none found. The only match was an empty `moonraker_api_key`.
