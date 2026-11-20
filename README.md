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

| Step | v1 | hesp2 |
|---|---|---|
| Language | `hesp_extract`, `hesp_recalib` are Python 2 (do not run on Python 3) | Python 3, vectorised |
| Frame types | `OBJ-TYPE` keyword | `OBJECT` name first (`OBJ-TYPE` was left at 4 for ThAr/flats/darks on this night), `OBJ-TYPE` as fallback |
| Bias | overscan (left strip, which has a ramp) + bias; **negative pixels set to 0** | right‑hand overscan per row + master bias, negatives kept, converted to electrons with a variance image |
| Traces | fixed 2017 polynomials, manual GUI shift | re‑measured on the night's flat for every order and both fibres (this night: −5.9 px, plus up to 1 px of change across the chip) |
| Scattered light | none | 2‑D model from the inter‑order gaps |
| Extraction | 5 px box sampled every 0.5 px → **flux ×2**, grid built with `linspace(0,4112,4112)` (≈1 px drift), point‑by‑point spline (hours per frame) | optimal extraction (Horne 1986) with the flat as spatial profile, outlier (cosmic‑ray) rejection, error spectra; < 1 s per frame |
| Flat field / blaze | **none** (quartz frames unused) | flat divided out; the quartz **lamp spectrum is separated from the blaze** first (see below) |
| Wavelengths | static 2017 table + global shift; `hesp_addwave` hard‑codes low resolution and writes 4 of 5 coefficients; reference files for `hesp_recalib` are missing from the repository | refitted every night: ThAr atlas cross‑correlation → iterative line identification against the KPNO ThAr line list → 2‑D Chebyshev in (x, m), saturated lines excluded |
| Other | — | barycentric correction (BERV, BJD), sky spectrum from the second fibre, per‑target combination with clipping, order merging, continuum normalisation, QA plots |

### Notes on specific choices
* **Wavelength solution.** For this night the v1 solution is off by −33 km/s on average and by up to ±40 km/s at the ends of the orders, so it cannot just be shifted. The pipeline first measures the offset in 8 chunks per order by cross‑correlating with the ThAr atlas, fits a smooth 2‑D correction, then identifies lines and iterates (each solution is the guess for the next) until the line list stops growing.
* **Lamp vs blaze.** The quartz ("QB") lamp spectrum has broad 30–50 % humps (e.g. near 4500, 5600, 7000 Å). Dividing a star by the raw flat imprints these, and they are as wide as the Balmer wings of A stars, which makes normalisation impossible. The flat is therefore modelled as `lamp(λ) × blaze(v)` with the order phase `v = m − K/λ` (K = m·λ at the blaze peak ≈ 344 750 Å), which is separable because neighbouring HESP orders overlap by about half. Only `flat / lamp(λ)` (blaze + pixel response) is divided out.
* **Cosmic rays.** L.A.Cosmic flags ~5 % of the pixels *on* the narrow fibre traces (only ~0.04 % off‑trace), so it is off by default; cosmic rays are rejected inside the optimal extraction and when the exposures of a target are combined.
* **Fibres.** In star‑sky mode the star is in fibre 2 (upper trace of each pair); the pipeline picks the brighter fibre automatically.

## Validation (night of 2026‑11‑13, low resolution)
* **ThAr fit:** 1055 lines (fibre 1) and 1001 lines (fibre 2) in 59 of 60 orders, rms 232 / 236 m/s (resolution element ≈ 10 km/s).
* **Internal consistency:** the same ThAr line seen in two neighbouring orders gets the same wavelength to 0.25 km/s rms, with no offset (v1‑based first pass: ~3 km/s).
* **Absolute zero point:** the [O I] 5577.338 Å night‑sky line in the sky fibre of every science frame is at +0.15 to +0.5 km/s, drifting by ~0.3 km/s over 5.5 h (`qa/zero_point_5577.png`, header keyword `DV5577`). The ThAr was only taken at the start of the night, so this is the check that it still holds later.
* **Order overlaps:** blaze‑corrected fluxes of neighbouring orders agree to ~1 %.
* **Background:** inter‑order residual after scattered‑light subtraction ≲ 0.1 % of the fibre peak.

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
Wavelengths are in air. Barycentric = observed × (1 + BERV/c).
`FLUX` in the per-order files is **not** sky‑subtracted; `SKY` (already scaled to the star fibre and
resampled onto its wavelengths) can be subtracted where night‑sky lines matter (e.g. Na D, [O I]).

## Known limitations
* **Continuum below ~3990 Å** in A‑type stars: overlapping Balmer wings leave no true continuum; the normalised spectrum is ~15–20 % low around Ca II H & K. Use the per‑order files and your own normalisation there.
* **Red (7200–9000 Å):** residual ±5 % undulations in the normalised spectrum; telluric lines (O₂, H₂O) are not removed.
* Orders 92–93 (λ < 3750 Å) have a noisy flat; a smooth blaze is used there.
* The two bluest orders have too few ThAr lines and rely on the 2‑D fit.

## Files
`hesp_night.py` (the script you run), `hesp2.py` (library), `v1/` (v1 traces and wavelength tables used as
guesses), `data/` (ThAr line lists and the KPNO ThAr atlas — Palmer & Engleman 1983, Norlén 1973 —
as distributed with PyReduce, Piskunov et al. 2021).

## Adding this to a fork of the HESP repository
Copy `hesp_night.py`, `hesp2.py`, `requirements.txt`, `v1/` and `data/` into the fork (for example
as a `hesp2/` folder next to `bin/`). `v1/` holds the trace and wavelength files from the original
`bin/` folder; the new code only reads them as starting guesses. Nothing needs to be installed or
put on the `PATH`, and there is no config file in the home directory any more.
