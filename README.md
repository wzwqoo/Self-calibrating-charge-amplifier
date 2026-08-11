# Self-calibrating charge amplifier — Tiny Tapeout sky130, 2×2 analog

Hard-macro analog submission. The GDS in `gds/` **is** the design; `src/project.v`
is a stub that exists only so the TT harness has a module to instantiate.

| | |
|---|---|
| top module | `tt_um_wzw_qamp` |
| tiles | `2x2` — 4 tiles, 341.32 × 225.76 µm |
| analog pins | **2** — `ua[0]` VOUT, `ua[1]` VCM |
| digital pins | 13 in (`ui[7:0]`, `uio[4:0]`), 0 out |
| supply | 1.8 V `VDPWR` only, `uses_vapwr: false` |

The layout is generated, not drawn by hand: every cell and the top come from
Python generators that emit magic TCL, so the tile is reproducible from source.

## What is in here

```
info.yaml                        project metadata — tiles, analog_pins, pinout
src/project.v                    Verilog stub (no logic; power pins + tie-offs)
gds/tt_um_wzw_qamp.gds           the design — copied from tt_v3/tt_um_wzw_qamp_sealed.gds
lef/tt_um_wzw_qamp.lef           SIZE 334.880 BY 225.760, 53 pins
docs/info.md                     datasheet
docs/tt_um_wzw_qamp.png          rendered layout
tt_analog_2x2.def                official, from tt-support-tools
scripts/tt_precheck_local.py     offline subset of TT precheck
```

The GDS is the **sealed** artifact, i.e. the one that went through the urpm
post-process (7 widths fixed, 14 spacings bridged) and passed final checks —
not the raw `tt_um_wzw_qamp.gds` and not the intermediate `_w.gds`.

## Sign-off status of the GDS

| deck | result |
|---|---|
| magic DRC | 0 |
| netgen LVS, top | match uniquely — 57 nets vs 57, 5 devices vs 5 |
| netgen LVS, per cell | match uniquely, all six |
| KLayout FEOL / BEOL / offgrid | 0 / 0 / 0 |
| zero-area, pin-label overlap | 0 / 0 |
| met5 (forbidden) | absent |
| bbox | **334.88 × 225.76 µm — matches the official DEF exactly** |

Against the official DEF, `scripts/tt_precheck_local.py` reports
**23 passed, 0 failed** — bbox == DIEAREA, unique top cell, prBoundary, no met5,
clean cell names, `ua[0..1]` connected while `ua[2..7]` are free, and the LEF
power pins carrying `USE POWER` / `USE GROUND`.

A ring-test audit of all 51 template pins against the official DEF is
**51 / 51**: `ua[0]`=152.26 and `ua[1]`=132.94 match to the nanometre, all 13
used digital inputs land, `uio_in[5:7]`/`clk`/`ena`/`rst_n` are free, and every
output is tied to VGND. Note that `def read` paints TT's own pin rects as met4,
so a plain intersection test reports all 51 connected and is worthless; the ring
test is what distinguishes your metal from the template's.

Needs `gdstk` and `pyyaml`:

```bash
python3 scripts/tt_precheck_local.py
```

## Fixed on 2026-08-11

The tile was 6.44 µm too wide: the old derived DEF halved the 4×2 width
(682.64/2 = 341.32) but the real series is `161.00 + n×173.88`, so 2×2 = 334.88.
No macro was ever over the line — only the frame, the bus tail and the VGND
stripe. Three constants in `tt_v3/scripts/gen_tt_v3_top.py` fixed it
(`TILE_W`, `STRIPE["VGND"]`, `TIE_X1`), which also tied the floating
`uo_out[0]`.

## Remaining

1. **No SPICE verification has been run on the v3 cells.** No testbench in the
   source repo drives `qdac.spice` or `slim.spice`; no corner run, no Monte
   Carlo. The `qdac` DNL figure quoted in the datasheet is analytical from drawn
   cap values, not simulated, and does not include cap mismatch. `ota_slim`,
   `bias_slim` and `integ_slim` have never been simulated in their v3 form —
   `bias_slim`'s start-up across ss is the specific untested risk.

This does not block the harness, but it is what decides whether the die works.
