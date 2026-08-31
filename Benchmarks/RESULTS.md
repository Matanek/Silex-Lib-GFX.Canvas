# Vector-font Canvas migration

The blocking comparison uses `Sources/TerminalScreenRendering.sx` from
Silex-Benchmarks commit `07610f0`, once against the legacy Canvas checkout
and once against the Part 06 candidate. Both executions use the same
Silex 0.43.0 compiler, `--release --nocache`, a 1024×624 logical frame,
density 2 text layers, one warm-up frame, and five independent process runs.

Environment: 2026-08-31, macOS Darwin 25.6.0, ARM64 (`T6030` kernel
platform). These measurements make no x64 performance claim.

| Engine | `full_10_keys_ms` samples | Mean | `ui_10_keys_ms` samples | Mean |
| --- | --- | ---: | --- | ---: |
| Legacy raster oracle | 194.846, 194.941, 181.523, 192.367, 190.074 | 190.750 | 47.400, 44.694, 47.160, 43.568, 45.491 | 45.663 |
| GFX.Font candidate | 195.505, 194.996, 193.679, 200.695, 192.628 | 195.501 | 43.430, 43.460, 46.429, 45.539, 42.176 | 44.207 |

The candidate changes the full-screen mean by +2.49% and the retained UI mean
by -3.19%. Both blocking metrics remain within the Part’s 5% gate. The
candidate shapes each fixed-grid line once, places Unicode clusters at exact
cell origins, and draws cached hinted glyph bitmaps through a bounded
256-slot direct cache. It does not retain one raster per evolving terminal
line.

The same run reported a natural monospace width of 960 logical units for 100
`x` glyphs versus 959 for the legacy oracle. Terminal layout uses its explicit
fixed advance, so this one-unit natural-metric difference does not accumulate
across cells. Visual and golden-mask acceptance remains a separate blocking
gate.
