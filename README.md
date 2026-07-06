# pdraw

One command for the whole Mac power story on Apple Silicon: **PD contract** (what the
charger promised), **battery flow** (who's covering the deficit), **SoC demand**
(what the silicon is burning) — as a one-shot snapshot or a live multi-series plot.

Built because no single tool showed supply and demand together:
[mactop](https://github.com/metaspartan/mactop) plots demand only;
[macpow](https://github.com/k06a/macpow) measures both but has no combined
realtime chart and no PD offer menu; `ioreg` has the PD data but no rendering.

## Usage

```
pdraw              # snapshot (~2s)
pdraw -w [SECS]    # live 2x2 panel grid (mactop-style): supply / demand / gpu / battery,
                   # each boxed with its own y-scale + color, PD contract as dashed
                   # reference line; Ctrl-C to stop
pdraw -i 250 -w    # faster sampling
pdraw top          # exec the macpow TUI
pdraw --json       # one merged sample as JSON (for scripts / logging)
pdraw --selftest   # built-in checks
```

Snapshot:

```
⚡ SUPPLY   contract 140W · 28V × 5A EPR · delivering 131.6W
           offer menu: 5V/3A · 9V/3A · 15V/3A · 20V/5A
🔋 BATTERY  68% · draining 8.6W (-0.72A @ 12.03V) · 30°C · health 98% · 44 cycles
🖥  DEMAND   system 136.3W · SoC 92.2W → gpu 56.4 + cpu 3.6 + dram 6.2 + fabric 25.8
           GPU 99% @ 1721 MHz · CPU 71°C · GPU 86°C · fans 5349/5782 rpm
⚖  HEADROOM delivered − demand = -4.7W (contract margin +3.7W) → battery assisting
```

The headroom line is the point: negative means the battery is silently draining
while "charging" — the net-drain regime an undersized charger puts you in.

## Requirements

- Apple Silicon Mac, python3 (stdlib only)
- [macpow](https://github.com/k06a/macpow) as the data engine:
  `brew tap k06a/tap && brew install macpow`

## Install

```
ln -s "$PWD/pdraw" ~/.local/bin/pdraw
```

## Data sources

| Data | Source |
|------|--------|
| SoC power, adapter delivered W, battery telemetry, temps, fans | `macpow --json` (IOReport/SMC, no sudo) |
| PD contract + full offer menu (`UsbHvcMenu`) | `ioreg -arn AppleSmartBattery` |

Last verified: 2026-07-06 (macpow v0.1.19, macOS Tahoe, M5 Max).
