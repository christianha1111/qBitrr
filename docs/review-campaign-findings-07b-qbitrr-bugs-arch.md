# Review campaign 2026-07 — prepared findings (fallback)

> **Why this file exists:** The **Issues tab is disabled** on `christianha1111/qBitrr`
> (the GitHub API returns `410 Issues has been disabled in this repository`), so the
> findings below could not be filed as GitHub issues. This document is the sanctioned
> fallback so no findings are lost.
>
> **To file them properly:** enable the Issues tab (repo **Settings → General →
> Features → check "Issues"**). Once enabled, each `### Issue:` block below can be
> filed verbatim (title, labels, body). The review session can also file them
> automatically on request.
>
> Covers **Phase 1** (baseline: fork delta + secrets) and **Phase 2** (bugs +
> architecture + fork strategy). Snapshot date: 2026-07-24.

---

## Labels to create (name → color → description)

| Name | Color | Description |
|---|---|---|
| `P0-critical` | `b60205` | Leaked secrets, exploitable vulnerabilities, data-loss bugs |
| `P1-high` | `d93f0b` | Significant bugs, security hardening, architectural blockers |
| `P2-medium` | `fbca04` | Architecture improvements, refactors, best-practice alignment |
| `P3-low` | `0e8a16` | Feature ideas, nice-to-haves |
| `security` | `ee0701` | Security-related; used together with a P label |

(`bug` and `enhancement` already exist in the repo.)

---

## Tracking issue to create

**Title:** `Review campaign 2026-07 — tracking`
**Labels:** `P2-medium`
**Body:**

```
Coordination issue for the July 2026 review campaign. Each phase posts a comment here; later phases read those comments for dedupe.

## Phases

- [ ] Phase 1/2 — baseline: fork delta + secrets (`07a-qbitrr-baseline.md`)
- [ ] Phase 2/2 — bugs + architecture + fork strategy (`07b-qbitrr-bugs-arch.md`)
```

---

## Phase 1 — baseline (summary; to post as a tracking-issue comment)

- **Fork delta:** `christianha1111/qBitrr` is an **unmodified, stale mirror** of
  `Feramance/qBitrr`. Fork `master` (`1be23b1`, 2025-06-03, v4.10.23) is a direct
  ancestor of upstream `master` (`f467cb0`, 2026-07-21, v5.13.1-1):
  **0 commits ahead, 0 files changed, 1254 commits behind.**
- **Secrets scan: CLEAN — no P0.** No leaked credentials in the working tree or the
  reachable history. All pattern hits were `CHANGE_ME` placeholders, config-key
  references in code, correct `${{ secrets.X }}` CI references, or package names
  (`tokenize-rt`). Shallow clone (53 reachable commits, boundary `b5adc80`) pickaxed
  clean; no `.env`/`.pem`/keys/`.netrc` ever committed or deleted.
- **Overview doc:** was missing → added `docs/OVERVIEW.md` in **PR #1**
  (`review/overview` → `master`).

---

## Phase 2 — bugs + architecture + fork strategy

Scope note: Phase 1 established the fork has **zero delta** (no fork-added/modified
code). Per campaign scope ("do not file style issues against upstream code"), Phase 2
does **not** file speculative bug/refactor/style issues against the unchanged upstream
code. The two issues below are the defensible findings: the fork-strategy
recommendation (centerpiece) and one concrete security exposure present in the frozen
snapshot (which upstream has already fixed — illustrating exactly why the sync matters).

### Issue: `[arch] Fork is 1254 commits behind upstream — adopt a sync strategy`

**Labels:** `P1-high`, `enhancement`

**Body:**

```
## Context
This repo (christianha1111/qBitrr) is a fork of Feramance/qBitrr. As of 2026-07-24:

- Fork `master` HEAD: `1be23b1` (2025-06-03, upstream version 4.10.23)
- Upstream `master` HEAD: `f467cb0` (2026-07-21, upstream version 5.13.1-1)
- Fork commits ahead of upstream: **0**
- Files the fork changes vs upstream: **0**
- Commits the fork is **behind** upstream: **1254**

The fork HEAD is a direct ancestor of upstream `master`: every tracked file is
byte-identical to upstream at the 2025-06-03 fork point. There is no divergence.

## Problem
The fork carries none of ~13 months of upstream development, including bug fixes and
security fixes. Concrete example: the credential-logging exposure tracked in the
security issue below was already fixed upstream but is still present in this fork.
Anyone deploying this fork runs stale, less-secure code.

## Recommendation (make a decision, then act)
Because the fork has **no local changes**, there is nothing to preserve and nothing to
contribute upstream. Choose one:

- **A. Consume upstream directly (recommended if no local changes are planned):**
  archive or delete the fork and install from source of truth —
  `pip install qBitrr2` or Docker `feramance/qbitrr`. Zero maintenance.
- **B. Keep the fork and sync:** fast-forward `master` to upstream now (no conflict
  risk, since there is no divergence):
  - GitHub UI: the repo's **"Sync fork"** button, or
  - CLI: `git fetch upstream && git push origin upstream/master:master`
  Then adopt a cadence (e.g., on each upstream release or monthly) to fast-forward,
  and keep any future local changes minimal and PR them upstream instead of letting
  the fork diverge.

## Acceptance criteria
- [ ] A decision is recorded: consume-upstream-directly (A) OR keep-and-sync (B).
- [ ] If B: `master` is fast-forwarded to a recent upstream commit and
      `git rev-list --count upstream/master ^origin/master` returns 0 (or near 0).
- [ ] If B: a documented sync cadence or automation (GitHub "Sync fork" / a scheduled
      workflow) exists so the delta does not silently regrow.
- [ ] If A: the fork is archived/deleted and deployment uses the upstream PyPI/Docker artifact.

## Reproduce the delta
    git remote add upstream https://github.com/Feramance/qBitrr
    git fetch upstream
    git log --oneline origin/master ^upstream/master     # empty: no fork-unique commits
    git rev-list --count upstream/master ^origin/master  # commits behind

## Architecture assessment (for context — not separate issues)
The codebase is 100% upstream, so these are upstream's concerns, best addressed by
syncing (and, if desired, contributing upstream) rather than diverging in this fork:
- One dominant module: `qBitrr/arss.py` is ~5,321 lines (~73% of the code); most logic
  (Arr + qBittorrent orchestration, search, tracker mgmt, request sources) lives there.
- No automated tests in the repo tree.
- Broad exception handling: ~21 `except Exception` / `contextlib.suppress` sites in
  `arss.py`, plus many `while True` loops — error-handling and loop-exit strategy would
  benefit from review upstream.
- Dependencies are pinned to exact versions (good); CI (CodeQL, pre-commit, release,
  nightly) exists upstream.
```

### Issue: `[security] qBittorrent password & Arr/Ombi/Overseerr API keys written to debug logs`

**Labels:** `P2-medium`, `security`, `bug`

**Body:**

```
Blocked by #<fork-strategy issue number>

## Summary
In this fork's frozen snapshot (v4.10.23), several log statements emit raw credentials.
When logging is set to DEBUG/TRACE (`Settings.ConsoleLevel` / `QBITRR_SETTINGS_CONSOLE_LEVEL`),
the plaintext qBittorrent password and Arr/Ombi/Overseerr API keys are written to the
console and to log files. Self-hosted users routinely enable debug logging and paste
logs into public GitHub issues / Discord, which would leak live credentials.

## Exact locations (this fork, commit 1be23b1)
- `qBitrr/main.py:59-65` — logs
  `"qBitTorrent Config: Host: %s Port: %s, Username: %s, Password: %s"` with
  `self.qBit_Password` (raw password) at DEBUG.
- `qBitrr/arss.py:407-423` — the config log block logs `"... API Key: %s ..."` with
  `self.apikey` (raw Arr API key) at DEBUG.
- `qBitrr/arss.py:466` — `self.logger.debug("Script Config:  OmbiAPIKey=%s", self.ombi_api_key)`.
- `qBitrr/arss.py:473` — `self.logger.debug("Script Config:  OverseerrAPIKey=%s", self.overseerr_api_key)`.

## Before -> after
- Before: the actual secret value is written to the log.
- After: log only whether the value is configured (a boolean), or a redacted/masked
  value. This is exactly what upstream already does:
  `upstream/master:qBitrr/main.py:115` logs
  `"... Username configured: %s, Password configured: %s"` (booleans), and upstream's
  `arss.py` no longer logs the API keys.

## Note
This is already fixed upstream — the cleanest resolution is to sync (see the
fork-strategy issue). If not syncing immediately, apply the masking locally.

## Acceptance criteria
- [ ] No log statement emits a raw credential (qBit password; Arr/Ombi/Overseerr API keys).
- [ ] `qBitrr/main.py` logs configured-or-not booleans instead of the password (matching upstream).
- [ ] `qBitrr/arss.py` no longer logs `APIKey` / `OmbiAPIKey` / `OverseerrAPIKey` values.
- [ ] `rg -n 'Password: %s|API Key: %s|APIKey=%s' qBitrr/` returns nothing.
```

---

## Phase 2 wrap-up notes (to post as a tracking-issue comment once filed)

- Scope actually covered: fork-delta-driven review. Priority areas inspected in
  `qBitrr/arss.py` (auth handling, `folder_cleanup`/`remove_and_maybe_blocklist` file
  deletion, retry `while True` loops), `qBitrr/config.py`, `qBitrr/env_config.py`,
  `qBitrr/gen_config.py`, `qBitrr/main.py`.
- No path-traversal/data-loss issue filed: deletion targets derive from qBittorrent's
  reported paths and are guarded against removing the completed-download root
  (`remove_and_maybe_blocklist`, `arss.py:2821`); nothing fork-specific or clearly
  exploitable was found worth filing against unchanged upstream code.
- Features / best-practice: none filed — explicitly scoped to the (empty) fork delta.
- Nothing cut short.
```
