# Decisions

Dated, irreversible-leaning decisions, one entry each, newest last. The
reasoning lives in `PLAN.md`; this file is the index of what was decided and
when, so a future reader never re-derives whether a door was walked through.

**Started 2026-08-30**, later than this repository itself — the doctrine says
every repository carries one and, at that moment, three did not. **The last
two arrived on 2026-08-31**: the site's and `realtime-data-repo`'s, and the
site's `check:docs` has required one in every repository since. Entries before
2026-08-30 are reconstructed from `PLAN.md` and the site's, and are marked as
such.

**What counts as one-way in a data repository**: a decision that puts bytes in
readers' hands under a shape they will code against; a decision about which
repository owns a product, since moving one costs a migration in two places;
and a decision that forecloses an upstream. Tuning a threshold is none of
those.

## D1 — 2026-08-22 — This repository, and the currents move into it

*(Reconstructed 2026-08-30 from `pipeline/products.toml` and the site's
`PLAN.md`, which carried this repository's record before it had one.)*

The ESPC currents move out of `realtime-data-repo` into their own
repository: **one upstream, one fault domain, one gigabyte.** A HYCOM outage
must not hold back the site's observations, and the observations' failures
must not hold back the currents.

**The Navy temperature and salinity stayed behind**, on the owner's call the
same day, after the storage was measured rather than estimated — the combined
figure lands at 96% of the 1 GB Pages cap, less than one current frame of
margin. **The ice stayed behind too, deliberately, as a live test that moving
one product between repositories is cheap.**

One-way in the practical sense: moving a product between repositories is
cheap in machinery and expensive in everything that points at it — roots in
the contract, origins in the site's config, the union `check:docs` holds
across origins.

## D2 — 2026-08-30 — The fields/currents split, and a legacy name kept

Every model gets **two repositories**, split along the axis that actually
costs bytes:

- **`<model>-model-currents-repo`** — the tiled vector fields. Expensive.
  This repository's tile tier is **89% of its own bytes**: two forecast leads
  across five depths.
- **`<model>-model-fields-repo`** — the scalar fields. Cheap. 44–58 MB each.

Measured, from this repository's own byte log: currents alone are 831.7 MB
(81.2% of the cap); the scalars land at 150.3 MB (14.7%), or 195.3 MB (19.1%)
with a heat-content layer. **The split does not merely fit — it unblocks the
Navy scalars, which cannot move anywhere today at 96%.**

**`espc-model-fields-repo` is created for SST, SSS, SSH, SIC, SIT and
eventually OHC**, taking them out of `realtime-data-repo` — which then has
one subject, observations, rather than two.

**This repository keeps the name `espc-model-repo` as a deliberate legacy
exception.** Under the convention it would be `espc-model-currents-repo`. The
rename was dropped because its URL is a **live origin**, hardcoded in the
site's config and answering now, and GitHub Pages project sites do not
reliably redirect after a repository rename: old paths 404 and a reader
holding a stale bundle gets nothing until reload. The owner took the
structure without the migration, knowingly.

**One-way in the naming, not the bytes.** Every model added after this one
inherits the two-repo shape, and each new pair makes the exception here
slightly more surprising. The mitigation is that `CLAUDE.md` says at the top
what this repository actually holds.

**What this decision does NOT change**: the ESPC hour rule still spans ten
roots in two repositories, and the only arrangement that would fix it — all
ten in one repository — is exactly what storage forbids. Cross-origin
enforcement of that rule is permanent, not a stage toward something tidier.

**Nothing has moved.** The decision is recorded; the migration is a separate
sitting.
