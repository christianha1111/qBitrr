# qBitrr — Repository Overview

> Scope note: this is a **fork** of [`Feramance/qBitrr`](https://github.com/Feramance/qBitrr).
> Per the project review, this document focuses on **this fork's delta** and gives a
> compact map of the codebase for orientation. It is not upstream's documentation.

## Fork status (the delta)

**This fork currently adds nothing to upstream — it is an unmodified, stale mirror.**

| | |
|---|---|
| Fork `master` HEAD | `1be23b1` — 2025-06-03 (upstream version `4.10.23`) |
| Upstream `master` HEAD | `f467cb0` — 2026-07-21 (upstream version `5.13.1-1`) |
| Commits the fork adds on top of upstream | **0** |
| Files the fork changes vs upstream | **0** |
| Commits the fork is **behind** upstream | **1254** |

The fork HEAD is a direct ancestor of `Feramance/qBitrr@master`: every commit and
every file tracked here is byte-identical to upstream at the 2025-06-03 fork point.
There are no fork-specific features, fixes, or configuration. The practical
consequence is that the fork lags ~13 months of upstream development (v4.10.x → v5.13.x).
A fork sync/rebase strategy is tracked separately in the review campaign.

To reproduce this delta:

```bash
git remote add upstream https://github.com/Feramance/qBitrr
git fetch upstream
git log --oneline origin/master ^upstream/master   # (empty: no fork-unique commits)
git rev-list --count upstream/master ^origin/master # commits behind
```

## What the project does

qBitrr is a Python daemon/CLI that monitors [qBittorrent](https://github.com/qbittorrent/qBittorrent)
and drives [Radarr](https://github.com/Radarr/Radarr) / [Sonarr](https://github.com/Sonarr/Sonarr)
(plus optional [Overseerr](https://github.com/sct/overseerr) / [Ombi](https://github.com/Ombi-app/Ombi)
request sources). Core responsibilities:

- Detect stalled/bad torrents, remove and blacklist them on the Arr, optionally re-search.
- Tell the Arr to import completed downloads; clean up the completed folder.
- Validate media with `ffprobe`; skip files by extension/folder/regex.
- Trigger missing/upgrade searches on Radarr/Sonarr (by movie availability, by
  series/episode, year-by-year, custom-format/quality-unmet).
- Manage per-tracker settings, free-space thresholds, and temporary quality-profile swaps.

## Key components (entry points)

| Module | Lines | Role |
|---|---:|---|
| `qBitrr/main.py` | 233 | **Entry point** `run()`; bootstraps config, spawns a child process per Arr instance (via `pathos`), supervises them. |
| `qBitrr/arss.py` | 5,321 | The core (~73% of the code): Arr + qBittorrent orchestration, search logic, tracker management, Overseerr/Ombi requests. |
| `qBitrr/gen_config.py` | 789 | Generates the default/interactive `config.toml`. |
| `qBitrr/config.py` | 175 | Loads and accesses TOML config (`get`, `get_or_raise`). |
| `qBitrr/env_config.py` | 61 | Environment-variable config schema (`environ-config`). |
| `qBitrr/tables.py` | 72 | `peewee` ORM models for the SQLite state DBs. |
| `qBitrr/ffprobe.py` | 105 | Fetches/wraps `ffprobe` for media validation. |
| `qBitrr/logger.py` | 169 | Logging setup (`coloredlogs`). |
| `qBitrr/utils.py` | 207 | Shared helpers. |
| `home_path.py`, `bundled_data.py`, `errors.py`, `__init__.py` | <45 each | Small support modules. |

**Total:** 14 Python modules, ~7,216 lines; 51 tracked files overall.

## How to run

- **PyPI:** `pip install qBitrr2` then run `qbitrr` (console entry point
  `qBitrr.main:run`, or `python -m qBitrr.main`). On first run it writes a
  `config.toml` in the working directory from `config.example.toml`.
- **Docker:** image `feramance/qbitrr`; mount a `/config` volume. The container
  sets `QBITRR_DOCKER_RUNNING=69420`, which switches qBitrr to Docker paths.
- **Config:** TOML file (`config.example.toml`, ~35 KB) with optional
  environment-variable overrides (`QBITRR_*`). Credentials (`Password`, `APIKey`,
  `OmbiAPIKey`, `OverseerrAPIKey`) default to the placeholder `CHANGE_ME`.

## Requirements

- qBittorrent ≥ 4.5.x (latest fully supported: 4.6.7; v5 behind a config flag).
- Radarr (v4/v5) and Sonarr (v4) configured to tag downloads.
- Python ≥ 3.8.3, < 4. Key deps: `qbittorrent-api`, `pyarr`, `peewee`,
  `ffmpeg-python`, `pathos`, `requests`, `tomlkit`.

## Key numbers at a glance

- Codebase: ~7.2k LOC Python, one dominant module (`arss.py`, 5.3k LOC).
- Fork: **0** commits ahead, **1254** behind upstream (as of 2026-07-21).
- Frozen at qBitrr `4.10.23` (2025-06-03).
