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
pdraw              # live grouped plot, 2 wide: energy (draw / charger /
                   # compute / battery), gpu (util / memory), system
                   # (cpu / ram). Groups are split by a blank line, the
                   # same divider the snapshot uses; Ctrl-C to stop
pdraw -w SECS      # watch for SECS then stop
pdraw --log FILE   # watch and append each sample as JSONL to FILE
pdraw -i 250       # sample interval in ms (default 500)
pdraw -s           # one-shot snapshot
pdraw --json       # one sample as JSON
pdraw --selftest   # built-in checks
pdraw procs        # per-process CPU% / MEM% / GPU% / GPUMEM (no sudo)
                   #   sort: c cpu · g gpu · m mem · v gpumem · p pid · n name
pdraw procs --energy  # ...plus Apple's Energy Impact score (uses sudo)
pdraw top          # run mactop, if installed
```

Snapshot (`pdraw -s`):

```
  ●  Net drain                             -8.6 W

     battery  68 %
     draw    136 W   ███████████████████████████▎
     charger 132 W   ██████████████████████████▎
     rated   140 W   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░

     gpu      39 %   ██████████▉
     gpu mem  31 GB  ██████████████████▎
     cpu      18 %   █████
     ram      43 GB  █████████████████████████▎
```

The header is the whole story: `Net drain` with a negative number means the
battery is losing charge while plugged in — an undersized charger. The ledger
shows why: the charger delivers less than the system draws, and both sit under
the charger's rated ceiling. `battery` / `draw` / `charger` bars share one scale
(the charger's rated max, shown as the dim ceiling).

Below the blank line sits a second block on **different** denominators — `gpu`
and `cpu` against 100%, `gpu mem` and `ram` against installed memory. The gap is
deliberate: those bars are not comparable with the watt bars above them. Bounded
metrics are never auto-scaled, so a GPU steady at 40% reads as 40% rather than
filling its panel.

**`ram` excludes GPU memory, so `ram` + `gpu mem` is what the machine is using.**
On unified memory the GPU's allocation is charged as wired pages, and on this
hardware that was 30.1 of 43.0 GB — 70% of the raw "used" figure. Reporting the
raw number made `ram` a second view of `gpu mem` (they correlated at +1.00 over
a 10-minute soak) and implied the machine was nearly out of memory for
applications when apps held 12.9 GB and the GPU would release the rest.

Neither figure is attributable per-process: the GPU's share is kernel-side and
owned by no pid, which is why the sum of `pdraw procs` memory does not approach
either number.

The gpu and system blocks are omitted entirely when their data is unavailable,
leaving the original output untouched. Piped or non-interactive, `-s` collapses
to one line:

```
net drain 8.6 W · 68% · draw 136 W vs charger 132 W (rated 140 W)
```

## `pdraw procs` — who is using the machine

Bare `pdraw` never uses sudo. `pdraw procs` does, and it is the only path that
does: per-process energy comes from `powermetrics`, which requires root. It
spawns one long-lived `powermetrics` and reads its plist stream, rather than
re-running it each tick.

```
  ●  GPU 32% · gpu mem 1.9 GB · 20 W draw

         PID  PROCESS                     CPU ms/s   ENERGY     MEM
     ──────────────────────────────────────────────────────────────
       54555  gh                             227.8    660.3       —
       33214  stable                         114.7    326.5    162M
       67203  Moonlight                      282.9     94.9     73M
```

**Ranked by energy, not CPU.** Low CPU is the *absence* of a signal: it cannot
tell an idle daemon from a job that has handed its work to the GPU. Energy rises
for either. In the sample above `Moonlight` has 2.5x the CPU of `stable` but a
third of the energy — sorting by CPU would put the wrong process on top.

**Per-process GPU comes from `AGXDeviceUserClient` nodes in the IORegistry**,
which carry the owning pid and accumulated Metal GPU time in nanoseconds. No
sudo. It is *not* in the `IOAccelerator` subtree and *not* in `powermetrics`
(whose `--show-process-gpu` column reads `0.00` for every process on Apple
Silicon, `WindowServer` included, even at 36% GPU) — so it is easy to look in
the obvious places and wrongly conclude it does not exist.

`GPU%` is time-on-GPU as a share of wall clock, the same convention as `CPU%`.
Processes that hold GPU clients but are not ours to read — `WindowServer` above
all — are still listed, with `—` for CPU and memory.

### `GPUMEM%` — what it is, and what it is not

`GPUMEM%` sums a process's VM regions tagged `VM_MEMORY_IOACCELERATOR` (100) or
`VM_MEMORY_IOSURFACE` (88), via `proc_pidinfo(PROC_PIDREGIONINFO)`. No sudo.
It is shown as a percentage of installed memory — the same denominator as
`MEM%`, because unified memory means GPU allocations *are* system pages.

**It is user-mapped GPU memory only, and does not sum to the header's `gpu mem`.**
Measured across every process holding GPU clients it came to 0.33 GB while
`ioreg` reported 30.09 GB in use — 91x apart. Both figures are correct: most GPU
memory is kernel-side driver allocation, owned by no process. So `GPUMEM` tells
you which process is holding Metal buffers, not its share of the global figure.

It costs ~15 ms per process (one syscall per VM region, no bulk API, thousands
of regions for a busy process), so it is measured only for rows on screen and
cached for 5 s — GPU allocations move slowly. Amortised that is ~6 ms a tick.

The global `gpu mem` figure in the header is verified to track real GPU work: it
moved 1.8 -> 31.2 GB as utilization went 40% -> 99% while system RAM stayed flat.

`MEM` shows `—` for processes you do not own: `proc_pidinfo` reads all 200/200 of
your own processes but none belonging to other users, and a `0` there would be a
lie rather than a gap.

## Requirements

- Apple Silicon Mac, python3 (stdlib only). Nothing to install, no sudo.
- GPU figures come from `ioreg IOAccelerator`; CPU and RAM from mach
  `host_statistics` over ctypes — no subprocess, no per-process attribution
  (macOS does not expose per-process GPU on Apple Silicon).
- The battery gauge only refreshes every ~35 s, so pdraw re-reads it every 2 s
  and holds the value between reads. Log lines carry `battery_age_s`, the age of
  that reading. Note `Amperage` is a time-average over the gauge's window while
  the watt rails are instantaneous, so the two will not balance under varying
  load — that is the hardware, not a bug.
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
