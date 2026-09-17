# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.1.0] - 2026-09-17

### Changed

- **Fails closed when `systemd-run` is missing.** Previously the command ran
  uncapped after a stderr warning — a safety wrapper that becomes a no-op while
  still exiting like a success. Now it refuses with exit 69 (`EX_UNAVAILABLE`)
  unless `HEAVY_RUN_UNCAPPED=1` gives explicit consent. Found by an independent
  review panel on 2026-09-15.

### Added

- CI now tests **the cap itself**, not just argument parsing: a Python process
  allocating 3× the cap must be killed, and one allocating well under it must
  exit 0 — in the same job, so a broken harness cannot read as "blocked". On
  runners without user-cgroup delegation the step says so loudly instead of
  passing silently. The fail-closed path and the `HEAVY_RUN_UNCAPPED=1` override
  are tested everywhere.

### Fixed

- `parse_size` now handles fractional sizes (e.g. `HEAVY_RUN_MEM=10.5G`). The
  previous bash integer-arithmetic path aborted with an arithmetic syntax error
  on any decimal value under `set -euo pipefail`. Sizes are now converted via
  `awk`, which accepts fractions while still rounding to whole bytes.

### Changed

- Resolved ShellCheck findings (SC2295, SC2086); `shellcheck heavy-run` now
  exits clean.

### Added

- GitHub Actions CI (`.github/workflows/ci.yml`) running ShellCheck plus a smoke
  test that asserts usage/exit-code behavior and `parse_size` conversions.
