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
pdraw              # live 2x2 plot: draw / charger / compute / battery,
                   # each boxed with its own scale; Ctrl-C to stop
pdraw -w SECS      # watch for SECS then stop
pdraw --log FILE   # watch and append each sample as JSONL to FILE
pdraw -i 250       # sample interval in ms (default 500)
pdraw -s           # one-shot snapshot
pdraw --json       # one sample as JSON
pdraw --selftest   # built-in checks
pdraw top          # run mactop, if installed
```

Snapshot (`pdraw -s`):

```
  ●  Net drain                             -8.6 W

     battery  68 %
     draw    136 W   ███████████████████████████▎
     charger 132 W   ██████████████████████████▎
     rated   140 W   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

The header is the whole story: `Net drain` with a negative number means the
battery is losing charge while plugged in — an undersized charger. The ledger
shows why: the charger delivers less than the system draws, and both sit under
the charger's rated ceiling. `battery` / `draw` / `charger` bars share one scale
(the charger's rated max, shown as the dim ceiling). Piped or non-interactive,
`-s` collapses to one line:

```
net drain 8.6 W · 68% · draw 136 W vs charger 132 W (rated 140 W)
```

## Requirements

- Apple Silicon Mac, python3 (stdlib only). Nothing to install, no sudo.
- `pdraw top` runs [mactop](https://github.com/metaspartan/mactop) if you have it
  (`brew install mactop`); everything else is self-contained.

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
