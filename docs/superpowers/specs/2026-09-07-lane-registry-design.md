# pdraw lane registry — design

**Date:** 2026-09-07
**Status:** Approved, not yet implemented
**Scope:** Subsystem #1 of a four-part extensibility layer

## Context

pdraw is a 501-line single-file, stdlib-only, no-sudo Mac power monitor. Its live
view (`watch()`) plots four hardcoded lanes in a hardcoded 2x2 grid. The goal is
to make lanes definable without editing the render loop, as the foundation for
three later subsystems.

### Why a registry is justified

Enumerating the SMC key space on this machine (Apple Silicon, 2026-09-07):

| Measure | Count |
|---|---|
| Total SMC keys | 3,669 |
| Keys of type `flt ` | 2,242 |
| `P*` (power) keys | 81 |
| `P*` keys reading non-zero | 38 |
| **Surfaced by pdraw today** | **3** |

There is a real metric space here. A registry is not speculative generality.

### Why provenance is a first-class concern

Two of the live `P*` keys read `PZT0 = 343.000 W` and `PHPB = 200.000 W` — round
numbers exceeding what the 140 W charger can supply. These are almost certainly
static power *limits*, not live rails. Apple documents none of these keys; the
three pdraw uses were verified by energy balance (`pdraw:5-9`).

Adding a lane is mechanically trivial. Establishing what a key *means* is manual,
per-hardware, and easy to get wrong. **The design's job is not "make lanes
pluggable" — it is "make lanes pluggable without laundering unverified keys into
authoritative-looking charts."**

### nvitop relationship

Concepts are ported, not code. Verified 2026-09-07: nvitop v1.7.1 (released
2026-07-10), last commit 2026-07-27, 7,140 stars, actively maintained, not
archived. It is **dual-licensed**: `nvitop/api` and `select.py` are Apache-2.0,
but `cli.py`, `__main__.py` and the entire `tui/` package are **GPL-3.0** — which
is exactly the part whose panels and charts resemble pdraw. No nvitop code is
copied. pdraw has no LICENSE file today; one should be added before this layer
grows.

## Goals

1. Lanes are data, not code in the render loop.
2. Default behavior is unchanged for existing users.
3. Unverified metrics are always visibly unverified.
4. Stay single-file, stdlib-only, python3.9+, no install step.

## Non-goals (explicitly out of scope)

- The `-s` snapshot and `--json` / `--log` output are **untouched**. The registry
  drives the live plot only. (Decided deliberately; subsystem #2 may revisit.)
- No config file, no plugin modules, no entry points.
- No promotion of additional SMC keys to verified built-in status. That requires
  separate energy-balance verification work.
- Subsystems #2 (collector), #3 (interactivity), #4 (process attribution).

## Design

### 1. Data model

`LANES` (`pdraw:340`) becomes a table of descriptors:

```python
Lane = namedtuple("Lane", "name label color path key unit ref verified")

BUILTIN_LANES = (
    Lane("draw",    "draw",    "33", "sys_power_w",     None, "W", "rated", True),
    Lane("charger", "charger", "36", "adapter_power_w", None, "W", "rated", True),
    Lane("compute", "compute", "35", "compute_w",       None, "W", None,    True),
    Lane("battery", "battery", None, "battery_flow_w",  None, "W", 0.0,     True),
)
```

Fields: display identity (`label`, `color`), value source (**exactly one** of
`path` or `key`), render hints (`unit`; `ref` is the dashed reference line —
a literal, `None`, or the sentinel `"rated"` meaning PD contract watts), and
`verified`.

`battery` carries `color=None` because it keeps its existing signed red/green
treatment (`pdraw:384`).

### 2. Resolution — the provenance boundary

A lane sources its value **either** from a dotted path into the existing sample
dict **or** from a raw SMC key. That split *is* the provenance boundary:

- **Verified** lanes have a slot in the sample schema because someone proved what
  the key means.
- **Ad-hoc** lanes (`--lane PDBR:dc-batt:green`) have no schema slot. They read
  the key raw into `sample["raw"][KEY]` and carry `verified=False` into the panel
  title.

Provenance is therefore structural, not a flag someone must remember to set.

`lane_values(sample, lanes) -> {name: float}` returns the same shape as today, so
`watch()`'s history deques, resize re-flow, and per-panel scaling are untouched.

### 3. Acquisition

`read_sample(smc, extra_keys=())` gains one optional argument. With `extra_keys`
empty it produces a **byte-identical** dict to today's, so `--json`, `--log`,
`-s` and `--selftest` are provably unaffected. Extra keys land under a new
`"raw"` sub-dict — additive, never reshaping existing fields.

### 4. Layout

```python
def layout(n, cols, lines, panel_h=5):
    c_fit = max(1, cols // 33)                       # cols keeping charts >=24 cells
    c = min(max(1, ceil(sqrt(n))), c_fit)            # squarish, capped by width
    r = ceil(n / c)
    while r*(panel_h+2) + 1 > lines and c < c_fit:   # too tall -> widen first
        c += 1; r = ceil(n / c)
    while r*(panel_h+2) + 1 > lines and panel_h > 3: # still too tall -> shorten
        panel_h -= 1
    return c, r, max(24, min(80, (cols - 9*c)//c)), panel_h
```

Panel geometry: a panel occupies `width + 8` visible cells (`inner = width + 6`
plus two border chars) and `panel_h + 2` lines. A row of `c` panels joined by
single spaces spans `c*(width+8) + (c-1)` cells. Solving that for width yields
`(cols - 9*c + 1) // c`. The implementation deliberately uses the
one-cell-conservative `(cols - 9*c) // c` **because that is exactly today's
formula at c=2** — preserving default output byte-for-byte outranks reclaiming a
single column at odd terminal widths.

`c_fit = cols // 33` follows from the same geometry: a minimum-width panel spans
`24 + 8 = 32` cells, plus one joining space.

That is the generalization of today's `(cols - 2*9) // 2` (`pdraw:343`).
**Verified by sweeping cols 40..240 at n=4: zero mismatches against today's width
wherever 2 columns are chosen.** Today's 2x2 is literally the `n=4` evaluation of
the general rule, not a special case.

Scaling (120x40 terminal): `n=6 -> 3x2 @ w=31`, `n=8 -> 3x3`, `n=12 -> 3x4`. On a
short 120x20 terminal, `n=8` shrinks `panel_h` 5->4 rather than clipping.

`side_by_side()` (`pdraw:335`) is replaced by an N-wide grid join, and the
longhand 2x2 construction in `watch()` (`pdraw:383-397`, ~14 lines) is deleted.

#### Disclosed behavior change

Today's formula clamps width to 24 but then needs 65 columns to draw two panels,
so **below 65 columns the current frame silently wraps**:

```
cols=50: today w=24 frame=65 OVERFLOW  |  new 1x4 w=41 frame=49
cols=60: today w=24 frame=65 OVERFLOW  |  new 1x4 w=51 frame=59
cols=66: today w=24 frame=65 ok        |  new 2x2 w=24 frame=65
```

"Identical default" holds exactly for **cols >= 66**. Below that the new code
drops to one column, fixing an existing wrap bug. This is an intentional,
accepted change.

### 5. CLI surface

```
--lanes NAME[,NAME...]      select/reorder built-ins
                            (default: draw,charger,compute,battery)
--lane KEY[:label[:color]]  repeatable; ad-hoc raw SMC key, always unverified
--list-keys                 print live 'flt ' P* keys with current values, exit
```

`--lanes` and `--lane` are deliberately **not** merged. Built-in names and raw
SMC keys share a lexical space (`draw` and `PDBR` are both bare tokens), so one
flag would need disambiguation heuristics whose failure mode is silent: a typo'd
built-in name becomes an unverified raw key and pdraw plots whatever that fourcc
returns. Two flags make the verified/unverified choice explicit at the call site.

`--list-keys` exists because `--lane` is otherwise unusable without prior
knowledge that a key like `PDBR` exists among 3,669.

**Color format.** `BUILTIN_LANES` stores raw ANSI SGR codes (`"33"`). The
`--lane` argument accepts *either* a raw code or a name from a fixed map
(`yellow cyan magenta green red blue grey`), resolved to a code at parse time, so
the `Lane.color` field holds one type everywhere. An unknown color name errors
and lists the valid names.

Validation fails fast: an unknown built-in name errors and lists the valid set; a
key that does not exist or is not type `flt ` errors with the reason rather than
plotting zeros. Ad-hoc lanes without an explicit color draw from a cycling
palette that skips the four built-in hues, so a probe lane never impersonates a
known rail.

### 6. Testing

Follows the existing inline-assert `--selftest` pattern (`pdraw:422`). No pytest,
no new dependency.

1. `layout()` is pure — table-driven assertions over the cols 40..240 sweep,
   including the `n=4 @ 100x40 -> (2, 2, 41, 5)` anchor.
2. Lane resolution against a synthetic sample: dotted path, `raw` lookup, and a
   missing key producing a clean error.
3. **Golden-frame test** — render a full frame from a fixed synthetic sample with
   default lanes and compare against a stored expected string. This is the real
   guard: it fails loudly if the refactor perturbs default output by one cell.
4. An ad-hoc lane asserts `verified=False` reaches the panel title.

## Migration path

This is Approach A of three considered. Approach C (polymorphic `MetricSource`
classes) is the correct eventual shape once subsystem #4 introduces a genuinely
different source type (`powermetrics`), but costs ~120 lines of abstraction with
only one source type to justify it today.

The migration is mechanical: `Lane` gains a `source` field, and `path` / `key`
become two cases of it. Not a rewrite.

## Risks

| Risk | Mitigation |
|---|---|
| Refactor perturbs default output | Golden-frame test; width formula proven equal to today's at c=2 |
| Users plot limit keys as if rails | `verified=False` in panel title; separate flag from built-ins |
| Registry shape wrong for subsystem #4 | Documented mechanical migration to Approach C |
| Scope creep into snapshot/log | Explicit non-goal; `read_sample` default path byte-identical |

## Estimated size

~+80 lines added, ~14 deleted. Single file, stdlib only, no new dependency.
