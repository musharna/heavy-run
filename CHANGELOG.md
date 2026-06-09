# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
