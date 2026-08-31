# CLAUDE.md

Guidance for working in this repository. The **design** — what this
repository is for, what it publishes, why it exists apart from
`realtime-data-repo` — lives in `README.md` and is not repeated here. This
file is the operator's half, and it is short because this repository holds no
code.

<!-- DOC-DOCTRINE v1 begin — identical in all four repositories; `check:docs` holds them equal. Edit one, sync all. -->
## Where truth lives, and what "update docs" means

Four repositories carry this project, and each carries the same four
documents: `oceanlet.js` (the engine), `oceansensing.github.io` (the site,
and every fetch script), `realtime-data-repo` (the orchestrator both data
repositories run, and most products), `espc-model-repo` (the ESPC currents).
Each document answers exactly one question.

| file | answers | tense | it is stale when |
| --- | --- | --- | --- |
| `README.md` | what this is, how to run it | present | a reader types a command or trusts a number and is wrong |
| `CLAUDE.md` | what must not be got wrong here | imperative | the next session is about to repeat a mistake |
| `PLAN.md` | what happened, measured, and what is open | dated past | "why is it like this?" has no answer here |
| `DECISIONS.md` | which one-way door closed, and when | dated | a reversal would cost a migration and nothing says so |
| `docs/` | contracts, ledgers and the guide | present | it describes an interface, a divergence or a concept that has moved on |

**`docs/` is a first-class part of "all docs", not an appendix** — the owner
asked for that explicitly on 2026-08-28, and the reason is that these are the
documents everything else points AT. A frozen contract, a divergence ledger
whose rows are pinned by tests, a guide that introduces the model: each is
the thing a reader is sent to when the short answer will not do, so each is
the worst place for a claim that has quietly stopped being true.

**"Update docs" means a sweep of all four repositories, not the one in hand.**
Docs are part of the change, never a follow-up and never a separate ask. Six
questions, asked of every repository the change touched:

1. Did a command, a path, a script name or a number a reader would type or
   trust move? → `README.md`
2. Did a rule, a trap, or a things-that-must-move-together change or come to
   light? → `CLAUDE.md`
3. Did something *happen* — a measurement, a defect, a yield, a mechanism, an
   open question opened or answered? → `PLAN.md`
4. Did a one-way door close? → `DECISIONS.md`
5. Did an interface, a deliberate divergence, or a concept the guide explains
   move? → the matching file under `docs/`
6. **Does a document in another repository now say something false because of
   this change?** → fix it there, in the same sitting.

**Question 6 is the one that gets missed, and it is why this block is
identical in four places.** Measured 2026-08-28: one tile-tier measurement
falsified `espc-model-repo`'s README, its `products.toml` header and the
site's README at once. Two were found; the third took a reminder from the
owner, who then asked for this doctrine.

**A FIFTH repository consumes this system and is deliberately NOT in the four
above**: `ocean-now`, the iOS port, which mirrors the site's published
contract. It is not swept by these six questions and does not carry this
block — it has a lighter mechanism instead, a pending list in its parity
ledger, and the two repositories whose changes can reach it (the engine and
the site) each say so in their own section. It is named here because "four"
was read as "all of them" for two weeks while that ledger drifted 176 commits
behind with nothing noticing — which is question 6 failing at the granularity
of a whole repository rather than a document.

A number in prose is only as good as its anchor. `check:docs` gates every
claim it can tie to a source constant and nothing else, so when a figure has
no anchor — a measurement, a live reading, a byte count off a build log —
write **where it was measured and when**, or the next reader cannot tell a
fact from a guess that aged.
<!-- DOC-DOCTRINE v1 end -->

## The thing to understand before changing anything

**A fault here is almost never fixed here.** The fetchers live in
`oceansensing.github.io/scripts/`, the orchestrator in
`realtime-data-repo/pipeline/`, and both are checked out at run time. What
this repository owns is `pipeline/products.toml` (the declaration), the
workflow, the crons, and the published tree. So:

- A change to how a product is fetched, judged or assembled lands here on the
  **next run**, from a push to the *other* repository. Nothing needs pushing
  here for it to take effect, and pushing here will not make it happen sooner.
- The corollary, and it has cost real Actions minutes: when a defect in this
  repository's output is traced to a fetcher or the orchestrator, the urgent
  push belongs to that repository — and even there it rides the next cron.
  Ask what ships the fix before reaching for an immediate deploy.

## Reading a run

Start with the data, not the Actions page:

```sh
curl -s https://oceansensing.org/espc-model-repo/status/status.json | python3 -m json.tool
```

Each product carries `fate`, `stale`, `ageHours`, `hour`, `modelRun` and, when
held, a one-sentence `reason`. `status/receipt.json` beside it says what was
**built** and what was **withheld**, which is the only place a missing tile
tier is visible: a tier that was never written cannot be withheld, so it is
named there with its reason rather than inferred from a directory walk.

**`deploy: false` freezes the whole tree, every product**, including ones that
fetched cleanly. That is the consumer contract refusing the tree, and while it
holds, `status.json` freezes with it — so a reason shown on the live tree can
be older than the fault a run is actually hitting. Read the run log, not only
the published status, when the two could disagree.

**A held product does not by itself mean `deploy: false`.** Since
`realtime-data-repo`'s `58a7207`, contract failures attributable only to
products this run already held deploy the rest — so the ordinary depth-hold
prints four `FAIL` lines and `run: deploy=True` in the same log. FAILs in the
log are not the same question as a frozen tree; check the `run:` line.

**And `checked` freezes for a held product — read `generated` first.**
`checked` means *the last time the pipeline successfully attempted this
product*, so one held every twenty minutes for six hours advertises a
`checked` six hours old. That reads as "nothing has looked at this since",
which is the opposite of what is happening, and it cost twenty minutes on
2026-08-28. `generated` at the top of the document is the run that actually
ran; a stale `checked` beside a current `generated` is the ordinary
signature of a hold. (`realtime-data-repo`'s `CLAUDE.md` says `checked`
going quiet **across products** means the pipeline is not completing — that
is the plural case, and it is a different reading from this one.)

## What has already gone wrong here

- **A tile build that produced nothing reported success** (2026-08-28). With
  HYCOM's `.das` timing out, `fetch-currents.py --tiles` returned 0 on the
  strength of `currents.json` — a *grid* the call was never asked to build —
  and four of the five current layers went on advertising a `tileIndex`
  against a 404. Fixed in both other repositories: the site's `outage_exit`
  refuses to degrade a tiles call, and the orchestrator now accounts for an
  absent tier whatever the product's fate. **Rebuilding is for fresh
  products; accounting is for all of them.**
- **This repository runs the fetcher SIX times per publish, and they have to
  agree.** `--tile-key` once per product, then the fetch, `--tiles`,
  `--namespace`, `--quality` — each a fresh process re-resolving the step
  selection and re-probing HYCOM. On 2026-08-29 the new depth probe made
  three `--tile-key` calls seconds apart return an outright failure, a
  one-frame key and a two-frame key; every depth tile tier went 404 and was
  withheld while the surface published. The fetcher now shares one selection
  per publishing slot through `RUNNER_TEMP`. **Anything that makes a probe
  costlier or stricter has to keep those six answers identical** — a tile
  cache key that does not identify its own content is worse than no cache.

- **The storage ceiling is real and it is latent.** Measured 2026-08-28: the
  tile tier is 738.7 MB and the grids 93 MB, so this repository is 832 MB of
  a 1 GB Pages cap — but a broken tile build makes it read as 93 MB, which
  looks like room that is not there. Measure the tier from a *successful*
  run's byte log, never from the current tree.
- **HYCOM fails partially, not cleanly.** One run can rebuild the caps and
  surface tiers and still produce nothing for 50 m. Do not read one product's
  success as the upstream being healthy.
- **A step can serve the surface and lie below it** (2026-08-28). `serves()`
  in `fetch-currents.py` read `LEVELS[0]` only, on a docstring asserting that
  a step serving 0 m serves 50 m. **FIXED the same day, on the owner's call**
  — it now probes every depth a run reads (`probe_depths()` = surface, 50 m
  and the deepest profile level) and refuses a window whose latitude rows are
  all identical or whose speeds no current reaches. Keep the history, because
  the assumption is the tempting one:
  Measured straight off `tds.hycom.org` that day: at time index 73 — valid
  15:00Z, the last hour the 08-27T12Z run owns in an interleaved `best`
  aggregation — the surface read clean and every deeper level returned
  garbage, **five identical requests giving four different answers** (17-25
  m/s against an `actual_range` topping out at 5.5; a neighboring level's row
  broadcast down every latitude; once a near-zero field). Its neighbors,
  indices 70-72 and 74-76, were all clean. So the probe passes, the surface
  publishes, and both depth domains fetch poison and hold — for six hours, as
  it turned out, because the anchor stays within `MAX_FROM_ANCHOR` of the
  poisoned hour that long. **The gate is the only thing standing between that
  and a published field; do not weaken it to clear a hold.**

  Two things about the fix that are easy to get wrong later. **The probe is
  not domain-aware and must not become so**: every `--only` invocation
  re-resolves `frames()` after the shared fetch has written the grids, so a
  per-domain answer would let `--tiles --only=surface` build tiles for an
  hour the grid was never fetched at, under the grid's own filename. And
  **bytes coming back is not the test** — the corrupt reads were HTTP 200
  full of plausible floats, so `probe_verdict` is the part that earns the
  extra reads. Cost measured: 0.4 s for three depths, 0.2 s when it refuses,
  against ~1 s for the old single read.
