# AGENTS.md

This is a local fork of an upstream open-source project (govee2mqtt), used to
run H1310 ceiling-fan support that upstream has not merged.

## AI harness

Read [../ai-docs/harness.md](../ai-docs/harness.md) (general) and
[../ai-docs/claude-harness.md](../ai-docs/claude-harness.md) (Claude Code)
before starting — shared token-efficiency/delegation rules across all
projects.

## Verification

Run before every commit — this mirrors what CI (`.github/workflows/pr.yml`)
executes:

```bash
cargo build --all && cargo test --all && cargo fmt --check
```

`cargo clippy` is *not* part of CI. It reports pre-existing findings in
`src/platform_api.rs`, `src/hass_mqtt/work_mode.rs` and `src/rest_api.rs`
under current toolchains; those are inherited from upstream, so judge a
branch by whether it adds new ones, not by a clean run.

## Remotes and branches

| Remote | Points at | Role |
|---|---|---|
| `origin` | `wez/govee2mqtt` | upstream, read-only — no write access |
| `fork` | `Kuro4S/govee2mqtt` | our own line, push here |

- `main` tracks `origin/main` but has deliberately diverged: it carries the
  merged H1310 work plus fork-only commits. Syncing upstream is therefore a
  `git merge origin/main`, never a fast-forward.
- `feature/h1310-ceiling-fan` is the head of
  [wez/govee2mqtt#698](https://github.com/wez/govee2mqtt/pull/698). Do not
  commit to it except to address review feedback — anything else changes the
  open PR.

## Planned work

### Expose the ceiling fan as a real hass `fan` entity

Today an H1310 shows up in hass as a light plus a `Fan Toggle` switch and a
`Fan Speed` select. Hass does not recognize that combination as a fan, so
there is no fan card, no `fan.set_percentage`, no useful voice control, and
automations have to coordinate two entities. The MQTT fan platform models
exactly this: `fanToggle` as state/command, `fanSpeedMode` as `preset_modes`
or a `percentage` across the six speeds.

There is no `src/hass_mqtt/fan.rs` yet. [humidifier.rs](src/hass_mqtt/humidifier.rs)
is the closest template, since it already combines a toggle with a mode
select. This is a feature in its own right and belongs on a separate branch,
not in the open PR #698.

### `DeviceType::Fan` is inert — do not "fix" it casually

`Device::device_type()` prefers `http_device_info` over the quirk, and Govee
reports the H1310/H1370 as `devices.types.light`. So the `DeviceType::Fan` in
their quirk never takes effect in practice, and `DeviceType::Fan` is matched
nowhere else in the tree. Two consequences:

- Behavior for these devices must hang off explicit quirk flags (as
  `empty_platform_state` does), never off the device type.
- The `mdi:fan` icon works *because* the resolved type is `Light`:
  [light.rs](src/hass_mqtt/light.rs) only applies a quirk icon for
  `DeviceType::Light`. Making `device_type()` honor the quirk would silently
  drop the icon, and would change resolution for every other quirked device
  as well. If it is ever worth changing, it needs its own branch and a sweep
  over all quirks.

## Fork-only changes

Some commits on `main` must never reach an upstream contribution branch,
because they repoint the build at our own registry or drop upstream jobs:

- `.github/workflows/build.yml` — `IMAGE: ghcr.io/kuro4s/govee2mqtt`, and the
  removed add-on jobs (see below)
- `addon/Dockerfile`, `addon/config.yaml`, `addon/build.yaml` — image name,
  project URL, and the cosign identity

Upstream's hardcoded `ghcr.io/wez/govee2mqtt` makes every push on this fork
fail with `denied: permission_denied`. Keep these out of PR branches.

## Distribution: container only

`main` publishes a multi-arch image on every push:

```bash
docker pull ghcr.io/kuro4s/govee2mqtt:latest
```

The package is public, so no `docker login` is needed. Tags of the form
`YYYY.MM.DD-<hash>` mark releases and publish an identically named image tag.

The Home Assistant **add-on line is disabled**. `home-assistant/builder` is
deprecated: releases that still publish a builder image pin cosign v2.5.3,
which can no longer verify the base image signatures (`no signatures found`),
and the one release with a usable cosign (2026.06.0) publishes no image at all
(ghcr 404). No combination works, so both `addon` and `test-addon` were
removed from the workflow. The `addon/` sources remain in the tree; reviving
the add-on means migrating to the builder's successor first.

## Pushing

Push to `fork` over **SSH**. The macOS Keychain token used for HTTPS lacks
the `workflow` scope, so any push touching `.github/workflows/**` is rejected
outright. See [../ai-docs/harness.md](../ai-docs/harness.md) → Working rules →
Commit.
