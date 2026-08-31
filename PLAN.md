# espc-model-repo — running record

What this repository has done, measured, and what is open. Started
2026-08-28, when the four repositories were given matching document sets;
**records before that date live in `oceansensing.github.io/PLAN.md`**, which
carried this repository's history while it had no record of its own — the
split's design, the three-product fault-domain argument, the quality gate and
the currency work are all there and are not copied here.

## Where it stands

Live at <https://oceansensing.org/espc-model-repo/> since 2026-08-22, holding
no code: the fetchers come from `oceansensing.github.io`, the orchestrator
from `realtime-data-repo`, both checked out at run time. Three products, one
per upstream read — `currents-surface`, `currents-50m`, `currents-caps` — so a
clean surface publishes while a faulty depth holds. Crons at `7,27,47` past
the hour plus `:12` on the 3-hour anchor boundaries, offset from the sibling
so two repositories never read HYCOM in the same minute.

## 2026-08-31: the split is executed, and this repository did not move

`espc-model-fields-repo` published the ESPC scalars for the first time at
06:26Z — `sst-navy`, `sss-navy`, `sic-navy`, `sit-navy`, `ssh-navy`, all
fresh, four tile tiers, ≈169.5 MB, **16.6%** of its 1 GB cap. This repository
is untouched: it held the currents before and holds them now.

**What it owed was two stale claims about products it never had.**
`pipeline/products.toml`'s header said the Navy temperature and salinity
*stay in `realtime-data-repo`* — true from 2026-08-22, false from today — and
`index.html`, the page this origin actually serves at its root, said this
repository publishes "its sea surface temperature and salinity". **That one
had been false since 2026-08-22**, nine days, on a public page, and was found
by reading the file rather than by any gate. Both corrected.

**The three-way hour agreement.** Ten ESPC roots now sit in two repositories —
five here, five there — and the map holds them to one anchor across origins.
It works because the anchor is a pure function of time: the origins agree
*without coordinating*. Observed at 06:49Z, the two credit lines carried
different runs (this repository on 2026-08-30 12Z, the scalars on 2026-08-29
12Z) at the same valid hour, which is the documented note rather than a
fault — and it predates the move, since `realtime-data-repo` was publishing
the same 08-29 run before it.

**What the migration found that touches this repository not at all, but will
touch the next one:** a step's scope does not follow its products'. A fetch
script's families can split across repositories while its invocation does
not, and the write fence is what notices, at the cost of a failed run. This
repository owns every family `fetch-currents.py` writes, so its step needs no
`--only` — and that is exactly why the trap is easy to miss here.

## Storage, measured 2026-08-28

The figure that decides whether the Navy products can move here, taken from
the 2026-08-27 17:21Z run's own byte log rather than projected:

| | |
| --- | --- |
| tile tier, whole | **738.7 MB** across ten sets (two leads x surface, 50 m, three caps) |
| per set | 71–78 MB |
| grids on `published` | 93 MB, 60 files |
| **this repository, whole** | **832 MB — 81% of the 1 GB Pages cap** |

Adding the four Navy products would reach **~982 MB, 96%**, about 42 MB spare
— less than one frame. So the owner's 2026-08-22 call to keep sst and sss in
`realtime-data-repo` stands, and was slightly optimistic at ~969 MB.

**Nothing about the machinery blocks the move.** Stages A, B and C landed
2026-08-21 and the currents move exercised all three against a real second
origin on 08-22; `espc_window.py` since retired the "a third consumer means a
third copy of `REFRESH_HOURS`" objection. What blocks it is disk.

**The headroom is the second lead, not the tiles' resolution.** One lead per
depth halves the tier and puts all four Navy products near 613 MB. That
reopens "two frames per window" — an owner decision, not an engineering one.
Staged, the arithmetic permits what the all-at-once move does not: ssh (no
tile tier at all, ~1 MB) could go today; the ice is 57.6 MB and is already the
designated canary; sst and sss are the expensive pair at 89 MB together.

**A caution the measurement itself taught**: a broken tile build makes this
repository read as 93 MB, which looks like room that is not there. Measure the
tier from a successful run's byte log, never from the current tree.

## 2026-08-28: the doc sweep's first catch — a dispatch that could only fail

The `workflow_dispatch` dropdown offered `full | light` and passed the choice
straight to `orchestrate.py plan --mode`. The light path was retired on
2026-08-27 and `--mode` has accepted only `full` since, so **dispatching
`light` failed the run outright** — and the three tile caches' `restore-keys`
were light-only expressions that could never evaluate to anything else.

Nobody would have found it from either side alone. The orchestrator that
dropped the mode lives in `realtime-data-repo`; the dropdown that still
offered it lives here; and nothing pointed from one to the other. It was found
by the cross-repository sweep the new doc doctrine asks for, on the sweep's
first outing — which is the argument for the doctrine better than the doctrine
makes it.

The input, the mode step and the dead `restore-keys` are gone; the plan step
passes `--mode full` outright. A second pass over the same workflow found the
comment that had justified those `restore-keys` still standing over their
absence, and claiming they were "kept identical to the sibling's" — the
sibling has none either.

## 2026-08-28: one poisoned time step, and it does not answer the same twice

Reported by the owner at about 17:40Z as "the map is out of date", with a
share hash carrying `Currents 0-200m mean (ESPC)`. The surface was `fresh` at
15:00Z throughout; both depth products were `held`.

**The timeline, from the run logs rather than from `status.json`** — which
freezes `checked` on a held product and so cannot tell you this:

| | |
| --- | --- |
| 11:11Z run | depth grids clean, 0.60-1.10 anisotropy, published. The last good one. |
| 15:32Z run | **first rejection**, at 2.3e14. |
| 16:21Z run | rejected again, at 1.1e14 — a different number for the same field. |
| 16:21Z-18:40Z | no runs at all; GitHub dispatched none for two hours. |
| ~17:40Z | the owner reports. |

So at the report the hold was 2.1 h old, the last depth publish was 6.5 h
back, and the newest published depth frame was valid 12:00Z.

**What a reader actually saw was 5.7 h old, not nine.** The product `hour` is
the base frame; each depth product also publishes its `+3h` frame — valid
12:00Z — and the map opens on whichever published valid time is nearest the
reader's clock. Read off the live page at 18:34Z, the credit line said *"valid
2026-08-28 12Z (-6 h), 2026-08-27 12Z run"*. The visible gap between the
surface layer and the depth layers was three hours. Quoting `hour` as "what
the map is showing" overstates the age by one frame, every time.

The gate's readings from the 16:23Z run, against a threshold of 2.0: the 50 m
domain 6.5e13 to 1.1e14 across all four grids; the caps 3.72 / 4.00 / 4.26
global, 2.74 / 2.87 / 2.99 Atlantic, 38.4 / 62.0 / 126.1 Antarctic, and the
Arctic passing at 0.99 / 1.07 / 1.58. The 11:11Z run before it read 0.60 to
1.10 everywhere. So the fault arrived between 11:11Z and 15:30Z, and it is
not a threshold that wants moving.

**Probed straight off `tds.hycom.org`**, no pipeline in the way:

- The dataset shape had not moved (`depth[40]`, `lat[4251]`, `lon[4500]`),
  `depth[14]` is still exactly 50.0, and `PROFILE_DEPTHS` still matches the
  axis it hardcodes. Nothing about our indices was wrong.
- **Exactly one time index is poisoned: 73, valid 2026-08-28T15:00Z.**
  Indices 70, 71, 72, 74 and 76 read clean at 0, 20, 50 and 200 m. Index 71
  is the 09:00Z hour still being served, and it is clean — what is published
  is good water, merely old.
- **Five identical requests for (t=73, z=14) returned four different
  answers.** Three constant down every latitude; one carrying 17.22 and
  25.701 m/s where `water_u`'s own `actual_range` tops out at 5.505; one
  "varying" at a maximum of 0.037 m/s, which is not ocean either. At z=8 the
  answer was the surface field's own last latitude row broadcast down every
  row; at z=13 it was byte-identical to z=14.
- **The `best` aggregation is interleaved**, which is the likely cause: run
  ownership ping-pongs across the forecast range — 64-73 to the 08-27T12Z
  run, 74-76 back to 08-26T12Z, 77-84 to 08-27 again — and index 73 sits at
  the first seam. No 08-28T12Z run existed yet.

The three-product split did exactly what it was built for: a clean surface
published while both depth domains held, with the reason in `status.json`.

**The pipeline could not step off the poisoned hour, and now it can.**
`serves()` probed `LEVELS[0]` only, on a docstring asserting that a step
serving 0 m serves 50 m — the assumption this step falsifies. Replaying
`pick_nearest` with the real constants (`REFRESH_HOURS` 3, `MAX_FROM_ANCHOR`
6.0, offsets [0, 3]) against the live axis: index 73 stayed selected at the
18:00Z and 21:00Z anchors and would only have dropped out at the 00:00Z
anchor on 08-29 — about six hours of hold from the report.

**Fixed the same evening, on the owner's instruction.** The probe now reads
every depth a run reads and judges what comes back:

- `probe_depths()` — the surface, 50 m and the deepest profile level
  (`[0, 14, 32]` as configured). **Deliberately not the `--only` domain's
  depth**: every `--only` invocation re-resolves `frames()` after the shared
  fetch has written the grids, so a per-domain answer would let
  `--tiles --only=surface` build tiles for an hour the grid was never
  fetched at, under the grid's own filename.
- `probe_verdict()` — pure, and the part that earns the extra reads, because
  the corrupt reads were HTTP 200 full of plausible floats. It refuses a
  window whose latitude rows are all identical, or whose speeds exceed
  10 m/s against a dataset `actual_range` topping out at 5.505. Below 20 wet
  cells it declines to judge rather than condemning a step for being over
  land.
- The window went from 3x3 at one depth to **10x10 at three**, because two
  latitude rows 0.08 deg apart are identical often enough at 0.001 m/s that a
  small window could not carry the test. Measured over 21 clean readings —
  seven steps at three depths — every one gave **10 distinct rows out of
  10**; the poisoned step gave **1**. No near miss in that sample.

Nine self-test cases, all five mutations killed (drop either refusal, probe
the surface only, drop the wet-cell floor, refuse everything). **Driven
against live HYCOM while the fault was still up**: `serves(73)` False,
`serves(71/72/74)` True, and `pick_nearest` walked the 08-27T12Z run past
and returned T+54 valid 18:00Z and T+57 valid 21:00Z from the 08-26T12Z run
— **-1 h from the clock**, against the -6.6 h the held layers were showing.

What it costs: a poisoned depth can walk the **surface** back a run too,
since selection has one answer for the whole run. That is the deliberate
trade — the fault domains still stop a bad depth from HOLDING a clean
surface, which is what they are for, while all three move together to an
hour that serves, which also keeps the consumer's ESPC hour rule satisfied.
`MAX_FROM_ANCHOR` bounds how far back that can reach, and here the older run
held fresher hours than the newer one's poisoned tail. The unbounded half is
the run's own age — T+54 off a 55 h run — which is exactly the `runAgeHours`
signal proposed and deferred in `realtime-data-repo`'s PLAN.

**Two instruments, two audiences, and only one had been taught about this
origin.** The site's watchdog caught it correctly and sixteen minutes before
the owner did — GitHub issue #2 at 17:24Z, *"espc-model-repo published
behind: currents-50m (4.4 h), currents-caps (4.4 h). Newer data existed and
was not published — this one is ours."* Its origin sweep was fixed on 08-27.
The map's own pipeline health line was not: it read `dataBase`'s status
document alone, so this origin's statement never reached the page the owner
was actually looking at. Fixed in `oceansensing.github.io` the same evening,
with its record there.

## 2026-08-29: the depth probe made this repository's tile keys unstable

A regression from the morning's probe change, found the same night by
reading a run dispatched to prove something else, and fixed. Recorded here
because the damage landed on this repository's tiles.

The orchestrator runs `fetch-currents.py` **six times per publish** —
`--tile-key` once per product, then the fetch, `--tiles`, `--namespace` and
`--quality` — each a fresh process that re-resolves the step selection and
re-probes HYCOM. Three depth reads per candidate step instead of one is
three times the exposure to a server that fails per request, and
`pick_nearest` truncates its frame list on a probe failure rather than
rejecting the run. So the six invocations stopped agreeing.

Before (2026-08-28T16:22Z) and after (2026-08-29T03:17Z), same three calls:

```
plan: currents-surface -> ...-r20260827T12-f20260828T15     |  probe failed (exit 1) — unplanned
plan: currents-50m     -> ...-r20260827T12-f20260828T15     |  ...-r20260827T12-f20260829T03
plan: currents-caps    -> ...-r20260827T12-f20260828T15     |  ...-r20260827T12-f20260829T03-f20260829T06
```

**Measured cost, live at 03:35Z:** `tiles-50m/index.json`,
`tiles-avg200m`, `tiles-avg350m` and `tiles-avg1000m` all **404**, correctly
withheld with `no readable index.json` while `tiles/index.json` (the
surface) served 200. Four of the five depth layers drew at the 0.96° global
grid at every zoom — the "staleness looks worse at high zoom" symptom, from
a cause that had nothing to do with staleness.

Fixed in `oceansensing.github.io`, which owns the fetcher: one selection per
publishing slot, shared through `RUNNER_TEMP`, so the first invocation
resolves it and the other five reuse it. Ten cases, five mutations killed;
the full record is in that repository's PLAN.

**The tier figures above are unaffected.** 738.7 MB across ten sets was
measured from a successful build's byte log and is a dated measurement, not
a claim about the current tree. What was wrong was the tree, for a few
hours, and it says so in `receipt.json` the whole time — which is the
withholding doing its job.

## 2026-08-29: a tier may now have holes, and it names them

The tile-key fix above was necessary and not sufficient — the keys agreed and
every tier was still 404. The cause was older and separate, and it is this
repository's layers that paid for it:

```
  ! tile -40_80 failed: water_v[77][0] failed after 4 tries: HTTP Error 500
! currents unavailable: 1 of 162 tiles failed after 4 tries each (87 not attempted)
--- tiles currents-surface: exit 1
withheld  currents-surface/tiles: no readable index.json
```

**One corner of 162 refused, and the whole tier went.** Every part behaved as
designed, the withholding included — a tier that is not there is correctly
declared absent rather than advertised. What was wrong is that 161 good tiles
were discarded for one transient 500, and 87 corners were never attempted.

Fixed in `oceansensing.github.io`, which owns the fetchers: a tier tolerates
`gap_budget()` refused corners (**6%** since later that day, shared in `espc_window.py` so the two
pipelines cannot drift), publishes with `gaps` naming each one, and stops
early only once that budget is exceeded — past it, the previous complete set
is kept, which is the case the original all-or-nothing rule was really for.

**What this repository should expect to see.** A tier can now publish with a
handful of holes; `receipt.json` and the run log say how many, and the site's
contract check emits a `note` rather than a failure. A tier that is *absent*
still means absent. If you are reading a tier's byte count against the 738.7
MB measurement, a tier with gaps is legitimately smaller and says so.

## 2026-08-29: the surface's +18h tier was absent and nothing said so

Reported as "how come coarse resolution current data is still served", at
zoom 9 over the Chesapeake, minutes after the tile tiers came back.

Both halves were true. `tiles/index.json` was complete and serving 0.08°
tiles — and the map was not using it, because this repository publishes two
frames (03:00Z and 06:00Z) and the map opens each layer on the one nearest
the reader's clock. At 05:30Z that is the +18h frame, and `tiles-f18h` was
404, so the layer fell back to `currents-atlantic-f18h.json` at 0.24°.

**The orchestrator could not see that tier was missing.** Its owed-tier list
filtered out every starred directory pattern, so `tiles-f*h` was excluded by
construction from the build trigger, the produced-nothing check and the
withheld accounting alike. This repository's `receipt.json` accordingly
reported `"currents-surface": {}` — nothing withheld — while a tier the map
was actively asking for did not exist. Fixed in `realtime-data-repo`
(`expected_tiers`), which owns the orchestrator; its PLAN carries the
mechanism.

**What to read differently here.** A `receipt.json` that names no withheld
tier now means what it always claimed to mean. Before this, it meant "no
BASE tier is missing", which is a much weaker statement and was not the one
written down.

## 2026-08-29: this repository published tomorrow morning

Reported as the currents being hours out. They were, in the other direction:
all three products were `fresh` and every one carried
**2026-08-30T03:00:00Z — seven hours into the reader's future** — with
nothing near the present on the map, while HYCOM had 09:00, 12:00, 15:00 and
18:00 that day available.

From the run log at 19:50Z:

```
T+39: valid 2026-08-30T03:00:00Z from the 2026-08-28T12:00:00Z run (32 h old, +7 h from now)
```

The step picker asked "which run is newest?" before "which hour is closest?".
The `best` aggregation stitches runs unevenly, so the newest run's coverage
began after the reader while the run before it carried the anchor hour
exactly. Fixed in `oceansensing.github.io` (`run_order`), which owns the
fetcher: closeness first, recency as the tiebreak, one-run invariant intact.

**What to watch for here.** `ageHours` counts distance to the nearest valid
time in EITHER direction, so a forecast published too far ahead reads as
stale exactly like data published too far behind — all three products showed
`stale: true` with `ageHours 7.15` and nothing said which side of now they
were on. If a currency alarm fires and the data looks fresh, check the sign.

## DECIDED: the fields/currents split, and this repository's legacy name — 2026-08-30

The owner's proposal: rename this repository `espc-model-currents-repo`,
create a NEW `espc-model-repo` holding SST, SSS, SSH, SIC, SIT and eventually
OHC, and make that dual shape the pattern for every model added later
(MERCATOR, ECCOFS).

### Storage: yes, and it is not close

Computed from this repository's own recorded byte log rather than estimated:

| | | |
| --- | --- | --- |
| today, currents only | 831.7 MB | **81.2%** |
| today, if the Navy scalars moved in — **blocked** | 982.0 MB | 95.9% |
| proposed: currents repo, unchanged | 831.7 MB | **81.2%** |
| proposed: scalars repo (sst+sss+sic+sit+ssh) | 150.3 MB | **14.7%** |
| proposed: scalars repo, with OHC at sst's size | 195.3 MB | **19.1%** |

**The split does not merely fit — it unblocks something currently blocked.**
Moving the Navy scalars into this repository as it stands lands at 96% with
about 42 MB spare, less than one current frame, which is why they have stayed
in `realtime-data-repo`. Split, they sit at 19% with **828 MB of headroom**,
and the two-lead question this file has been holding open as the headroom
lever stops being forced.

The asymmetry is the whole reason it works: **the currents' tile tier is 89%
of this repository's bytes.** Two leads across five depths is what costs; a
2-D scalar field is 44–58 MB. Separating the expensive half from the cheap
half is a real structural axis, not an arbitrary cut, and it is why the
pattern generalizes to any model with both.

### What the owner decided, 2026-08-30

**The split goes ahead. The rename does not.**

- A new **`espc-model-fields-repo`** takes SST, SSS, SSH, SIC, SIT and
  eventually OHC — moving them out of `realtime-data-repo`.
- **This repository keeps its name and its contents**: the currents, at every
  depth. It is the currents repository in everything but its name.
- **Every model added later gets the two-repo shape with both halves named**:
  `<model>-model-currents-repo` and `<model>-model-fields-repo`. MERCATOR,
  ECCOFS and anything after them.
- **`espc-model-repo` is a deliberate legacy exception**, accepted as a
  compromise rather than overlooked. Under the convention it would be
  `espc-model-currents-repo`.

**Nothing moves yet.** The decision is recorded; the migration is a separate
sitting, and this section is what it will be executed from.

### Why the rename was dropped — the expensive half, and it is separable

`https://oceansensing.org/espc-model-repo/map/` is a **live origin**,
hardcoded in the site's `src/config.ts`, answering 200 right now. Renaming
the repository changes that path, and **GitHub Pages project sites do not
reliably redirect after a rename** — old URLs 404. A reader holding a stale
bundle fetches the old paths and gets nothing until they reload.

That is survivable with a coordinated deploy, and it buys only a name.

**The cheaper route to the identical structure**: leave this repository as
the currents repository under its current name, and add
`espc-model-fields-repo` for the scalars. Same split, same storage answer,
**no URL migration and no 404 window**. It also names both halves by what
they hold rather than one by its content and the other by its absence, which
is the better generalization: `<model>-model-currents-repo` and
`<model>-model-fields-repo`.

The owner took the compromise: the structure without the migration. That
leaves one repository whose name does not match the convention it belongs to,
which is a real cost paid knowingly — **a reader meeting
`espc-model-currents-repo` for MERCATOR and `espc-model-repo` for ESPC will
wonder whether they are the same kind of thing.** They are. `CLAUDE.md` says
so at the top, which is the mitigation.

The rename stays available at any time and will never be cheaper than it is
now, so deferring it has its own price. That is understood, not overlooked.

### What the split does NOT fix, and cannot

**The ESPC hour rule spans ten roots and would still span two repositories.**
The contract requires one model run to publish one hour across all ten
members; those ten come from two repositories, on two crons, behind two
independent publish gates, and neither can see the other's hour at publish
time. Only the site, reading both origins, can.

The one arrangement that would fix it — all ten in one repository — is
**exactly what storage forbids**, at 96% of the cap. So cross-origin
enforcement of that rule is permanent, not a stage on the way to something
tidier. Worth knowing before the split is read as a simplification.

### A gain the proposal does not claim for itself

Moving the Navy scalars out of `realtime-data-repo` gives that repository
**one subject rather than two**: observations, and not observations plus one
model's output. That is the same argument that created this repository, and
it applies again.

## Planned: upper-ocean heat content — 2026-08-30

The owner asked for an ocean heat content layer focused on the **upper**
ocean, for **hurricane intensity forecasting**, and asked whether it wants a
new repository — `espc-model-derived-repo` was the proposed name.

**The recommendation is no: a fourth product HERE, not a sixth repository.**
The reasoning, and the one measurement that could overturn it:

**The split's own argument does not apply.** This repository exists because
one upstream is one fault domain — a HYCOM outage must not hold back the
site's other products. Heat content's upstream is **the same ESPC model**.
It fails when the currents fail, so separating them buys no isolation; it
would only move the same failure to a different Pages site.

**Per-product fault domains already exist inside this repository.** That is
what `currents-surface`, `currents-50m` and `currents-caps` are: three
products from one model, so a clean surface publishes while a faulty depth
holds. A fourth product gets the same containment on the same machinery,
with no new repository, no new cron, and no new doctrine surface.

**A name that groups by HOW a number was made is the wrong cut.**
"derived" describes provenance, not what fails together — which is the
opposite of the principle that produced this repository. By that logic the
depth-averaged caps, which are also derived, would belong there too.

**The one thing that could overturn it is storage, and it is unmeasured.**
This repository stood at 832 MB, 81% of the 1 GB Pages cap, on 2026-08-28.
Whether a heat-content field fits depends on whether it needs a tile tier;
the surface Navy products were 44.0 and 45.1 MB each, and a 2-D field of the
same shape would sit near that. **Measure before deciding.** The headroom
lever is already recorded above: one lead per depth halves the tile tier.

### What it actually costs, which is not what it looks like

**This repository fetches no temperature at any depth.** It reads currents —
surface, 50 m, and the three depth-averaged caps — and nothing else. The
Navy `sst` is surface-only and lives in `realtime-data-repo`. So heat content
is **not** a derivation from files already held; it needs a **new upstream
read of the 3-D temperature field**, which is the expensive part and the part
a "derived" framing hides.

Upper-ocean heat content in the hurricane sense is Tropical Cyclone Heat
Potential: the heat integrated between the surface and the 26 °C isotherm.
That needs temperature on enough vertical levels to find that isotherm and
integrate to it — not one level, and not an average.

### A question the ECCOFS work reopens

`eccofs-model-repo` carries `temp` on **50 vertical levels at 3 km**, which is
exactly the field this product needs, and the site's PLAN already measured
that source in detail (2026-08-05, "Queued: ECCOFS"). So there are two
candidate upstreams and they are not interchangeable:

| | ESPC (here) | ECCOFS |
| --- | --- | --- |
| coverage | global | Grand Banks to the Orinoco |
| vertical | needs a new 3-D read | `temp`, 50 levels, already the plan |
| horizontal | the model's own | 3 km |
| grid | regular lat/lon | **curvilinear, terrain-following, staggered** |

**Coverage is the deciding axis for hurricanes.** ECCOFS stops well short of
the main development region off West Africa, so a storm's heat potential
along most of its track would be missing. ESPC is global and does not have
that problem. Against that, ECCOFS already intends to carry the vertical
structure, and its grid work is the largest data task yet queued.

**Recorded as a genuine fork, not a settled question.** A basin-wide layer
argues for ESPC and a new 3-D read here; a shelf-and-Gulf-Stream layer at
3 km argues for ECCOFS. The owner's phrase was "hurricane intensity
forecasting", which leans global.

### Open, for the owner

1. Which upstream — ESPC here (global, new 3-D read) or ECCOFS (regional,
   vertical structure already planned).
2. Whether a heat-content field needs a tile tier, which is the storage
   question above.
3. Whether the published quantity is TCHP proper (integrate to the 26 °C
   isotherm) or a simpler fixed-depth heat content, which is cheaper to
   compute and easier to explain but is not what the hurricane literature
   means.

## Open

- **The ESPC hour-rule collision, and it is the owner's.** One model
  publishing two hours fails the consumer contract. **It no longer freezes
  the tree** — since `realtime-data-repo`'s `58a7207` a contract failure
  attributable only to held products deploys the rest, and the 2026-08-28
  hold rode exactly that path (`run: deploy=True` under four FAILs) — but it
  is still four FAILs in every run's log while a depth product is held, and
  the log is what a reader checks. Live on 2026-08-27/28 for hours.

  **The depth-aware step probe removes one CAUSE of it, not the rule.** A
  step poisoned below the surface no longer splits the products across two
  hours, because all three now move together to a step that serves. Any
  other per-domain hold still can: a held product keeps its previous files
  at the old hour while its siblings publish the new one. Options recorded
  in the site's PLAN: treat
  a held product at an older hour of the same run as a note, or accept
  freeze-until-heal. The rule lives in the site's `test-schema.mjs` and is the
  consumer's to own; nothing has been changed unilaterally.
- **The second lead**, above.
- **ESPC's 2026-08-27 12Z run was ~18 h late, not skipped.** Probed directly
  at 03:39Z on 08-28: HYCOM's `time_run` axis offered nothing newer than
  2026-08-26 12Z, so this repository served a +39 h forecast with every signal
  healthy — the valid time was current, which is what `ageHours` measures.
  It landed between the 04:28Z and 07:14Z runs that morning, about eighteen
  hours after its nominal time, so the original wording here ("skipped") was
  wrong and is corrected. Upstream, not ours, and the map's credit line said
  so. A `runAgeHours` signal was
  proposed and the owner deferred it; it is written up in
  `realtime-data-repo`'s PLAN, which owns `status/status.json`.
- **HYCOM's `.das` is intermittently slow rather than down**, and one run can
  rebuild two tiers while a third produces nothing. Whether that wants a
  longer per-try timeout or a fetch that tolerates one dead product is not
  yet decided.
