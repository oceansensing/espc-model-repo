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
share hash carrying `Currents 0-200m mean (ESPC)`. Both depth products were
`held` and had been since 11:11Z, stuck at valid `2026-08-28T09:00Z` and last
published at 09:39Z — nine hours old by the time it was reported. The surface
was `fresh` at 15:00Z throughout.

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

**What this repository cannot currently do is step off the poisoned hour.**
`serves()` in `fetch-currents.py` probes `LEVELS[0]` only, on a docstring
asserting that a step serving 0 m serves 50 m — which is the assumption this
step falsifies. Replaying `pick_nearest` with the real constants
(`REFRESH_HOURS` 3, `MAX_FROM_ANCHOR` 6.0, offsets [0, 3]) against the live
time axis: index 73 stays selected at the 18:00Z and 21:00Z anchors and only
drops out at the 00:00Z anchor on 08-29, so the hold self-heals about six
hours after the report. Making the probe depth-aware would walk the depth
domains to the 08-26T12Z run and publish 18:00Z depth water beside 15:00Z
surface water — the hour rule that is already the owner's open decision
below, so nothing was changed.

**Two instruments, two audiences, and only one had been taught about this
origin.** The site's watchdog caught it correctly and sixteen minutes before
the owner did — GitHub issue #2 at 17:24Z, *"espc-model-repo published
behind: currents-50m (4.4 h), currents-caps (4.4 h). Newer data existed and
was not published — this one is ours."* Its origin sweep was fixed on 08-27.
The map's own pipeline health line was not: it read `dataBase`'s status
document alone, so this origin's statement never reached the page the owner
was actually looking at. Fixed in `oceansensing.github.io` the same evening,
with its record there.

## Open

- **The ESPC hour-rule collision, and it is the owner's.** One model
  publishing two hours fails the consumer contract. **It no longer freezes
  the tree** — since `realtime-data-repo`'s `58a7207` a contract failure
  attributable only to held products deploys the rest, and the 2026-08-28
  hold rode exactly that path (`run: deploy=True` under four FAILs) — but it
  is still four FAILs in every run's log while a depth product is held, and
  the log is what a reader checks. Live on 2026-08-27/28 for hours. Options
  recorded in the site's PLAN: treat
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
