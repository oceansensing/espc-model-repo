# espc-model-repo

The US Navy ESPC-D-V02 half of the ocean map's data, published to
<https://oceansensing.org/espc-model-repo/>.

It exists because ESPC is most of what the map publishes, and GitHub Pages
caps a site at 1 GB. Two depth-integrated products were the change that
would not fit.

**Measured 2026-08-28**, from the 2026-08-27 17:21Z run's own byte log rather
than from a projection: **93 MB of grids on the `published` branch and
738.7 MB of CI-built tiles across ten sets** — two leads each for surface,
50 m and the three caps, 71–78 MB per set — so **832 MB, 81% of the cap**.
The earlier figures here (114 MB and 505 MB) predated the third cap and the
second lead. What headroom exists is one lead per depth, which would halve
the tier; that is the "two frames per window" decision, and reopening it is
the owner's call.

`CLAUDE.md` is the operator's half — how to read a run, and what has already
gone wrong. `PLAN.md` is the running record: what has been measured here, and
what is open. Which document gets what, across all four repositories of this
project, is the doctrine block at the top of `CLAUDE.md`.

## It holds no code

The **fetchers** live in [`oceansensing.github.io`](https://github.com/oceansensing/oceansensing.github.io)
under `scripts/`, and the **orchestrator** lives in
[`realtime-data-repo`](https://github.com/oceansensing/realtime-data-repo)
under `pipeline/`. Both are checked out at run time, and `PIPELINE_ROOT`
points the orchestrator at this workspace so it assembles and publishes *this*
repository's tree from *this* repository's `pipeline/products.toml`.

The code lives once; the schedule, the declaration and the storage live per
repository. Copying 988 lines of orchestrator into every new data repository
was the alternative, and three faults were found in that file in a single day
— each would have had to be found twice and fixed twice.

## Live since 2026-08-22 — and what changed on 2026-08-27

Everything the section that used to sit here called "not set up yet" is
set up: the crons run (`7,27,47` past each hour, plus `:12` on the 3-hour
anchor boundaries, offset from the sibling so two repositories never read
HYCOM in the same minute — it was hourly at :05 until 2026-08-27, when
GitHub was measured delivering that schedule 45 min to 4 h 19 min apart
and never at the nominal minute), `PIPELINES_SSH_KEY` is armed with its own
read-only deploy key,
and the `published` branch exists and force-updates each publish.

`pipeline/products.toml` declares **three currents products, one per
upstream read** — `currents-surface` (the z=0 level), `currents-50m`
(z=14), `currents-caps` (the one shared profile read the three
depth-averages are computed from). One fetch step serves all three; what
splits is FATE. The split exists because HYCOM corrupted the deep reads
while the surface stayed clean (2026-08-26: every level below ~20 m
served as a column-constant broadcast profile with HTTP 200 throughout),
and a single product would have held the good surface hostage.

Each product declares a **physics quality gate**
(`fetch-currents.py --quality --only=<domain>`): anisotropy of the field
plus correlation against a running daily mean (`*-vbar1d.json`, folded
only from accepted base fields, judged-then-folded, deduped by valid
hour). A failing domain HOLDS — previous publish served, reason in
`status/status.json` — while the others publish.

**And the tree says when it is serving something old** (2026-08-27).
`status.json` carries `ageHours`, the distance from the reader's clock to
the NEAREST published frame, and `stale` against each product's
`max_age_hours` — 2 h here, because the anchor publishes frames that
bracket the reader and the nearest is at most 1.5 h away by construction.
It was 9 h, which is six times the owner's bar and measured the wrong
quantity besides: `stale` compared when the pipeline last PUBLISHED, so a
run republishing a nine-hour-old field every hour reported fresh. Old with
a fate of `fresh` means upstream has nothing newer and is a note; old with
any other fate means the data was there and was not published, and the
cross-origin watchdog in the site repository opens an issue for it. The fetch itself
probe-exits when nothing is new and nothing is missing, so most runs cost
two metadata reads — which is what makes three attempts an hour polite to
an upstream that has been answering some requests with timeouts.

**The quality gate judges SPEED, not velocity** (2026-08-27, the owner's
call). The motion that dominates where the mean flow is weak is rotary —
near-inertial and semidiurnal, and at high latitude those periods converge
— so the vector sweeps a circle while |v| barely moves, and the old check
read the u component alone. Measured on live grids 3 h apart against the
column-broadcast defect rebuilt from the same data, the good cases rose
from 0.496/0.578/0.863 to 0.742/0.799/0.922 while the corrupt ones did not
move: separation 0.52 to 0.70, threshold unchanged at 0.35. The reference
is a running mean of SPEED folded beside the vector one, and a median
speed-ratio band backstops the scale error that correlation structurally
cannot see.

**Known open decision** (recorded in this repository's `PLAN.md`): a held
domain at an older hour of the same model run fails the site contract's
ESPC hour rule, which sets deploy=False and freezes the whole Pages
tree until the domain heals or the run changes. Whether a held product
should read as a note rather than a fault is the owner's call.

## The published contract

Every root this repository publishes must be declared here **and** be one the
site's `test-schema.mjs --roots` publishes, which is the one derived list of
published files. That contract is three-sided now — this repository,
`realtime-data-repo` and the site.

The two halves are split by what each side can answer alone. **Here**: a run
exits 2 on a root declared in `products.toml` that the contract does not
publish. It does not fail on the reverse, because a contract root this
repository has no product for belongs to another origin, whose declarations
this run cannot see — failing on it would stop this publish over a gap in
somebody else's. **On the site**: `check:docs` reads `MAP_ORIGINS` and every
origin's `products.toml` together, and fails when a root is declared by
nobody or by two. Both directions are covered, and the union half runs before
a push rather than after one.

`status/status.json` is the routing document consumers read: per product, the
`roots` this origin serves, its `source`, its `hour`, the `hours` it offers
and its `modelRun`. The map knows *origins*; origins know *products*.
