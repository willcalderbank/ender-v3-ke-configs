# OrcaSlicer Settings — Ender 3 V3 KE (Ceramic Hotend)

OrcaSlicer presets for an **Ender 3 V3 KE** with an **upgraded ceramic hotend**.

These are the best settings I’ve found for my setup. Results may vary depending on your printer’s calibration, filament batch, ambient conditions, and hardware.

## What’s Included

This repo contains OrcaSlicer user presets in the standard folder layout:

- `machine/` — machine / nozzle presets
- `filament/` — filament presets
- `process/` — print (process) presets

## Supported Filaments

- Overture PLA Pro
- Overture PLA Matte

## Included Print (Process) Presets

All presets below are intended for a **0.4 mm nozzle**:

- PLA 0.4 Fine 0.12
- PLA 0.4 Optimal 0.16
- PLA 0.4 Standard 0.2
- PLA 0.4 Draft 0.24

## How to Use

### Option A: Copy into your OrcaSlicer user presets folder

1. Close OrcaSlicer.
2. Copy the `filament/`, `machine/`, and `process/` folders into your OrcaSlicer user presets directory.
3. Re-open OrcaSlicer.

Notes:
- If you already have presets with the same names, back them up first.
- You may need to select the new machine / nozzle preset, then pick the filament and process presets for a given project.

### Option B: Import presets from within OrcaSlicer

If you prefer to import rather than copy folders, use OrcaSlicer’s preset import features and select the relevant `.json` files from `filament/`, `machine/`, and `process/`.

## Tuning Tips (Recommended)

Even with good presets, you’ll usually get best results by validating:

- First-layer Z-offset
- Bed leveling / mesh
- Temperature tower for each filament
- Flow and pressure advance (if applicable to your workflow)

## License

See `LICENSE`.
