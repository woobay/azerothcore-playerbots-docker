# azerothcore-playerbots-docker

Automated Docker image builds for an AzerothCore worldserver/authserver with
the [mod-playerbots](https://github.com/mod-playerbots/mod-playerbots) module,
the [mod-multibot-bridge](https://github.com/Wishmaster117/mod-multibot-bridge)
module, the [mod-ah-bot-plus](https://github.com/NathanHandley/mod-ah-bot-plus)
auction house bot module, and the
[mod-individual-xp](https://github.com/azerothcore/mod-individual-xp) module, and the
[mod-dungeon-clear](https://github.com/jrad7/mod-dungeon-clear) module
built in, published to `ghcr.io/woobay/*`.

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

`mod-ah-bot-plus` is a normal (non-forking) AzerothCore module that populates
the in-game auction house with bot-driven listings and (optionally) bot
buyers, using file-based configuration instead of SQL. **Operational note:**
per its own README, the character(s) used as the AH-bot's listing identity
must be regular, non-playerbot characters (create a plain player account +
character and add its GUID to `AuctionHouseBot.GUIDs` in the module config).
This is a runtime/deployment concern on the live server, not something this
repo's CI manages.

`mod-individual-xp` is a normal (non-forking) AzerothCore module that lets
per-character XP rate multipliers be set on top of the server-wide
`Rate.XP.*` settings, via the `OnPlayerGiveXP` hook (persisted in a new
`characters.individualxp` table). It ships player-facing chat commands
(`.xp view` / `.xp set <rate>` / `.xp default` / `.xp enable` / `.xp disable`)
plus admin config (`IndividualXp.MaxXPRate`, `IndividualXp.DefaultXPRate`).
**Operational note:** it doesn't check for bot-controlled characters, so
playerbot characters will also get an `individualxp` row (harmless — it just
sits at `IndividualXp.DefaultXPRate` since bots can't run chat commands
themselves, and mod-playerbots' own separate `AiPlayerbot.RandomBotXPRate`
multiplier still applies independently on top).

## Pinning strategy: always one commit behind

`versions.env` pins exact commit SHAs for the core fork (`Playerbot` branch),
`mod-playerbots` (`master` branch), `mod-multibot-bridge` (`main` branch,
no tagged releases), `mod-ah-bot-plus` (`master` branch), and
`mod-individual-xp` (`master` branch), and
`mod-dungeon-clear` (`master` branch). We intentionally lag **one commit
behind the tip** of each branch — this gives upstream a chance to catch
obviously broken commits (CI failures, reverts, etc.) before we ever build
against them.

This is resolved statelessly: on every check, we ask GitHub for the two most
recent commits and take the second one. There's no persisted "last seen"
state to drift or get out of sync — regardless of how many commits land
between checks, we're always exactly one commit behind current tip.

## Fully automated pipeline

```
weekly schedule (check-updates.yaml)
  -> resolve 1-behind-latest SHAs for core + all four modules
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

Note: `mod-ah-bot-plus` has a known history of compile issues against
playerbot-forked cores in older/other AH-bot variants (mismatched
`WorldSession` constructor signatures introduced by playerbots' core changes).
It's untested against this specific pin at the time it was added; the same
build-only validation gate is relied on to catch any breakage rather than a
manual precheck.

## Images produced

- `ghcr.io/woobay/ac-wotlk-playerbots-authserver`
- `ghcr.io/woobay/ac-wotlk-playerbots-worldserver`
- `ghcr.io/woobay/ac-wotlk-playerbots-dbimport`

Each is tagged with:
- `core-<short-sha>_module-<short-sha>_mbbridge-<short-sha>_ahbot-<short-sha>_ixp-<short-sha>_dclear-<short-sha>` —
  traceable, reproducible tag
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
- **Allow GitHub Actions to create and approve pull requests** must be
  enabled (Settings -> Actions -> General -> Workflow permissions). Without
  this, `check-updates.yaml` fails at the PR-creation step with
  `GitHub Actions is not permitted to create or approve pull requests` even
  though the workflow already declares `permissions: pull-requests: write` -
  this is a separate, repo-level toggle that overrides workflow-level
  permissions
- Branch protection on `main` should require the `Validate` status check
  (from `validate-pr.yaml`) before merging
- GHCR package visibility should be set to **public** after the first
  successful push so the homelab cluster can pull without an
  `imagePullSecret`

