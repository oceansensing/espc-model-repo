# espc-model-repo

The US Navy ESPC-D-V02 half of the ocean map's data, published to
<https://oceansensing.org/espc-model-repo/>.

It exists because ESPC is 91% of what the map publishes — 114 MB of committed
grids and 505 MB of CI-built tiles against 59 MB of everything else — and
GitHub Pages caps a site at 1 GB. Two depth-integrated products were the
change that would not fit.

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
set up: the crons run (hourly at :05, plus :32 on the 6-hour boundaries,
offset from the sibling so two repositories never read HYCOM in the same
minute), `PIPELINES_SSH_KEY` is armed with its own read-only deploy key,
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
`status/status.json` — while the others publish. The fetch itself
probe-exits when nothing is new and nothing is missing, so most hourly
runs cost two metadata reads.

**Known open decision** (recorded in the site's `PLAN.md`): a held
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
