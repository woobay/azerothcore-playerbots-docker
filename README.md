# azerothcore-playerbots-docker

Automated Docker image builds for an AzerothCore worldserver/authserver with
the [mod-playerbots](https://github.com/mod-playerbots/mod-playerbots) module
and the [mod-multibot-bridge](https://github.com/Wishmaster117/mod-multibot-bridge)
module built in, published to `ghcr.io/woobay/*`.

## Why this repo exists

`mod-playerbots` requires a custom fork of AzerothCore
([mod-playerbots/azerothcore-wotlk](https://github.com/mod-playerbots/azerothcore-wotlk),
`Playerbot` branch) — it cannot be added as a drop-in module to the standard
`acore/*` images. This repo does **not** vendor or fork that source. It's a
thin CI wrapper that:

1. Checks out the upstream core fork + modules at pinned commits
2. Builds them using the upstream repo's own `apps/docker/Dockerfile`
   (targets: `authserver`, `worldserver`, `db-import`)
3. Pushes the resulting images to GHCR

`mod-multibot-bridge` is a normal (non-forking) AzerothCore module - the
server-side companion for the
[MultiBot Chatless](https://github.com/Wishmaster117/MultiBot-Chatless) client
addon, letting it control playerbots (roster, inventory, talents, loot rules,
etc.) via structured protocol messages instead of chat commands. It has no
database/SQL component of its own.

## Pinning strategy: always one commit behind

`versions.env` pins exact commit SHAs for the core fork (`Playerbot` branch),
`mod-playerbots` (`master` branch), and `mod-multibot-bridge` (`main` branch,
no tagged releases). We intentionally lag **one commit behind the tip** of
each branch — this gives upstream a chance to catch obviously broken commits
(CI failures, reverts, etc.) before we ever build against them.

This is resolved statelessly: on every check, we ask GitHub for the two most
recent commits and take the second one. There's no persisted "last seen"
state to drift or get out of sync — regardless of how many commits land
between checks, we're always exactly one commit behind current tip.

## Fully automated pipeline

```
weekly schedule (check-updates.yaml)
  -> resolve 1-behind-latest SHAs for core + both modules
  -> if changed: open a PR updating versions.env
  -> validate-pr.yaml runs (build-only, no push) as a required status check
       - builds all 3 targets: authserver, worldserver, db-import
  -> if validation passes: PR auto-merges
  -> if validation fails: PR stays open, auto-merge is cancelled
  -> merge to main triggers build.yaml
  -> build.yaml builds + pushes images to GHCR, tagged with the pinned SHAs
```

No human involvement is required end-to-end. The one-commit lag plus the
build-only validation gate are the two safety mechanisms protecting the
images that your live worldserver actually pulls.

Note: `mod-multibot-bridge` is a smaller, single-maintainer project with deep
integration into `mod-playerbots` internals (not just standard AzerothCore
hooks), so it carries more risk of a compile break against a given
`mod-playerbots` pin than the core/mod-playerbots pair does. It participates
in the same fully-automated pin-bump pipeline (by design/choice) rather than
requiring manual review - the `validate-pr.yaml` build-only gate is what
catches breakage before anything reaches `main`/GHCR.

## Images produced

- `ghcr.io/woobay/ac-wotlk-playerbots-authserver`
- `ghcr.io/woobay/ac-wotlk-playerbots-worldserver`
- `ghcr.io/woobay/ac-wotlk-playerbots-dbimport`

Each is tagged with:
- `core-<short-sha>_module-<short-sha>_mbbridge-<short-sha>` — traceable,
  reproducible tag
- `latest` — always points at the most recently built pin

## Manually triggering a rebuild

Both workflows support `workflow_dispatch` from the Actions tab:
- **Check for Upstream Updates** — force a check/PR right now instead of
  waiting for the weekly schedule
- **Build and Push** — rebuild and push images from the current
  `versions.env` without waiting for a new pin

## Build time

Full AzerothCore compiles can take well over an hour per target. Both
`validate-pr.yaml` and `build.yaml` set an explicit `timeout-minutes: 240`
on their build jobs so a runaway build fails clearly with a known budget
instead of relying on GitHub's implicit default (360 minutes).

## Repository settings required

- **Allow auto-merge** must be enabled in repo settings (Settings ->
  General -> Pull Requests) for the automated PR flow to work
- Branch protection on `main` should require the `Validate` status check
  (from `validate-pr.yaml`) before merging
- GHCR package visibility should be set to **public** after the first
  successful push so the homelab cluster can pull without an
  `imagePullSecret`

