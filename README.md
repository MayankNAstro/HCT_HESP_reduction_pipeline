# HCT HESP reduction_pipeline

hesp2 — updated reduction pipeline for HESP (HCT, Hanle)  An update of the HESP pipeline by Arun Surya ([arunsurya77/hesp_pipeline](https://github.com/arunsurya77/hesp_pipeline), v1.0.0, GPL‑3.0).

The v1 order traces and wavelength tables (`v1/`) are kept, but only as **starting guesses**:
everything is re‑measured on each night's own bias, flat and ThAr frames.

## Running a night

```
python hesp_night.py                       # asks for the folder of the night
python hesp_night.py  ~/data/2026-11-13_MN # shows the frames, asks a few questions, runs everything
python hesp_night.py  ~/data/2026-11-13_MN -y    # no questions, all defaults
```

What happens:
1. Every frame in the folder is listed with the type the pipeline gives it (bias, flat/QB, ThAr,
   science, dark). Frames whose `OBJ-TYPE` keyword disagrees with their name are marked
   (on 2026-11-13 the ThAr and QB frames were all left at `OBJ-TYPE = 4`).
2. Missing or thin calibrations are reported (no ThAr or flat for a mode means that mode is skipped).
3. You are asked: which resolution mode (only if both Low and High are present), which frames to
   leave out (names or just numbers, e.g. `12, 13`), the output folder
   (default `<parent>/reduced/<night>`), and whether to reuse existing calibrations.
4. The full reduction runs, then a summary per target is printed (frames, exposure, S/N, BERV,
   [O I] 5577 zero-point check), with a warning if the wavelength scale looks off.

Options: `-o OUT` output folder, `-x 12,13` frames to leave out, `-y` accept all defaults,
`--redo-calib`, `--no-plots`, `--cosmics` (see below).
Requirements: `numpy scipy astropy matplotlib` (`astroscrappy` only for `--cosmics`).
A night takes ~7 min the first time (most of it the wavelength solution), then ~1 min.

## What changed compared with v1, and why




## Output
```
calib/    master_bias.fits, master_flat.fits (rebuilt automatically if missing), calibration.fits (traces, WAVE, BLAZE,
          FLAT_EXTRACTED, THAR, ORDERS), thar_lines_used.csv
frames/   <frame>_spec.fits  — per science frame, extensions (orders x 4096 pixels):
            WAVE (air, observed frame), FLUX, ERR (electrons), BLAZE, SKY, SKY_ERR
            header: BERV, BJD_TDB, MJD-MID, STARFIB, SNRMAX, DV5577
          frames_summary.csv
targets/  <object>_orders.fits — exposures combined, per order: WAVE, WAVE_BARY, FLUX, ERR, BLAZE, SKY
          <object>_1d.fits      — merged, barycentric, 1.5 km/s log-lambda grid:
                                  WAVE, FLUX, ERR, CONT, NORM, NORM_ERR
          <object>_1d_norm.txt  — same, normalised, as text
qa/       traces, wavelength residuals, zero point vs time, a spectrum overview per target
```
Wavelengths are in air. 
## Files
`hesp_night.py` (the script you run), `hesp2.py` (library), `v1/` (v1 traces and wavelength tables used as
guesses), `data/` (ThAr line lists and the KPNO ThAr atlas — Palmer & Engleman 1983, Norlén 1973 —
as distributed with PyReduce, Piskunov et al. 2021).

## Adding this to a fork of the HESP repository
Copy `hesp_night.py`, `hesp2.py`, `requirements.txt`, `v1/` and `data/` into the fork (for example
as a `hesp2/` folder next to `bin/`). `v1/` holds the trace and wavelength files from the original
`bin/` folder; the new code only reads them as starting guesses. Nothing needs to be installed or
put on the `PATH`, and there is no config file in the home directory any more.
