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
passes `--mode full` outright.

## Open

- **The ESPC hour-rule collision, and it is the owner's.** One model
  publishing two hours fails the consumer contract, so a partial upstream
  outage freezes the whole tree — including products that fetched cleanly.
  Live on 2026-08-27/28 for hours. Options recorded in the site's PLAN: treat
  a held product at an older hour of the same run as a note, or accept
  freeze-until-heal. The rule lives in the site's `test-schema.mjs` and is the
  consumer's to own; nothing has been changed unilaterally.
- **The second lead**, above.
- **HYCOM's `.das` is intermittently slow rather than down**, and one run can
  rebuild two tiers while a third produces nothing. Whether that wants a
  longer per-try timeout or a fetch that tolerates one dead product is not
  yet decided.
