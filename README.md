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

## What is not set up yet

- **`pipeline/products.toml` declares no products.** Until the ESPC products
  move here, a run would assemble an empty tree, which is why the scheduled
  crons in the workflow are commented out rather than live.
- **`PIPELINES_SSH_KEY` is not set.** The site repository is private and holds
  the fetchers, so this needs the private half of a **read-only deploy key**
  whose public half sits on that repository's Deploy keys page.

  **Read-only: leave "Allow write access" unchecked.** A deploy key's *scope*
  is by construction — one repository, no other — but read-only is a checkbox.
  Nothing here pushes to the site repository, and this job runs `pip install`,
  `npm ci` and four fetch scripts against public servers, so write access
  would put the repository that deploys the public site inside that blast
  radius.

  **Its own key, not the one `realtime-data-repo` uses.** A repository may
  hold many deploy keys, and one per consumer means revoking or rotating one
  does not take the other down.

  **Order matters, and it has been measured the hard way:** put the public
  half on the site repository *first*, then set the secret here. A non-empty
  `ssh-key` sends `actions/checkout` down SSH immediately and the HTTPS
  fallback is not consulted, so setting the secret first breaks every run in
  between.

- **No `published` branch yet.** The first run creates it. Until then the
  orchestrator starts cold and says so.

## The published contract

Every root this repository publishes must be declared here **and** agree with
`test-schema.mjs --roots` in the site repository, which is the one derived
list of published files. That contract is three-sided now — this repository,
`realtime-data-repo` and the site — and a root declared on one side and not
another stops the publish rather than shipping a tree nobody can read.

`status/status.json` is the routing document consumers read: per product, the
`roots` this origin serves, its `source`, its `hour`, the `hours` it offers
and its `modelRun`. The map knows *origins*; origins know *products*.
