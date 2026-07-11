# pdraw

Mac power draw on Apple Silicon: what the charger is set to deliver, how the
battery is flowing, and what the system is pulling. Snapshot or live plot.

No single tool showed supply and demand together.
[mactop](https://github.com/metaspartan/mactop) plots demand only;
[macpow](https://github.com/k06a/macpow) measures both but has no combined chart
and no PD offer menu; `ioreg` has the PD contract but doesn't render it.

Reads the SMC directly over IOKit, no sudo and nothing to install. Battery and
adapter detail come from `ioreg`.

## Usage

```
pdraw              # live 2x2 plot: supply / demand / compute / battery,
                   # each boxed with its own scale; Ctrl-C to stop
pdraw -w SECS      # watch for SECS then stop
pdraw --log FILE   # watch and append each sample as JSONL to FILE
pdraw -i 250       # sample interval in ms (default 500)
pdraw -s           # one-shot snapshot
pdraw --json       # one sample as JSON
pdraw --selftest   # built-in checks
pdraw top          # run macpow's TUI, if installed
```

Snapshot:

```
⚡ SUPPLY   contract 140W · 28V × 5A EPR · delivering 131.6W · pd charger
           offer menu: 5V/3A · 9V/3A · 15V/3A · 20V/5A
🔋 BATTERY  68% · draining 8.6W (-0.72A @ 12.03V) · 30°C · health 98% · 44 cycles
🖥  DEMAND   system pull 136.3W · compute rail 92.2W
⚖  HEADROOM delivered − pull = -4.7W (contract margin +3.7W) → battery assisting
```

Watch the headroom line. When it's negative the battery is draining while macOS
says it's charging, which is what an undersized charger does.

## Requirements

- Apple Silicon Mac, python3 (stdlib only). Nothing to install, no sudo.
- `pdraw top` runs macpow if you have it; everything else is self-contained.

## Install

```
ln -s "$PWD/pdraw" ~/.local/bin/pdraw
```

## Data sources

| Data | Source | Keys |
|------|--------|------|
| System pull, adapter delivered W, compute-rail W | AppleSMC over IOKit (no sudo) | `PSTR`, `PDTR`, `PHPS` |
| Battery %, flow, temp, health, cycles | `ioreg -a -r -n AppleSmartBattery` | `Amperage`, `Voltage` |
| PD contract and offer menu | `ioreg` `AdapterDetails` / `UsbHvcMenu` | `Watts`, `UsbHvcMenu` |

Power keys verified 2026-07-06 by energy balance (PDTR delivered + battery ≈ PSTR
system) and under CPU load, on macOS Tahoe / M5 Max. Earlier versions used
`macpow --json`, dropped because its SoC watts read 0.0 on this chip
([issue #12](https://github.com/k06a/macpow/issues/12)) and the snapshot hung
waiting for a nonzero sample.

Last verified: 2026-07-06.
