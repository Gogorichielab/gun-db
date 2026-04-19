# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**gun-db** is a hierarchical JSON database of firearm makes, models, and variants, with a Python CLI for interacting with it. Released under CC0 1.0 Universal (public domain). Data covers manufacturer logos, model specs, and serial number decoding schemes.

## Development Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Start interactive CLI shell
./gun_db.py

# Validate Python syntax (what CI runs)
python -m py_compile gun_db.py

# Test CLI loads correctly
python gun_db.py --help
```

**Interactive shell commands** (run inside `./gun_db.py`):
```
🔫 ~ makes
🔫 ~ models "Sig Sauer"
🔫 ~ variants "Sig Sauer" P320
🔫 ~ get_info "Sig Sauer" P320 "320CA-9-M18-MS"
🔫 ~ set_info "Sig Sauer" P320 "320CA-9-M18-MS" caliber "9mm"
```

There is no formal test suite — CI validates only syntax and CLI startup.

## Architecture

The project is two things: a file-system database (`db/`) and a thin CLI over it (`gun_db.py`).

### Database Hierarchy

```
db/
└── {manufacturer}/          # e.g. sig-sauer, smith-and-wesson
    ├── logo.json
    └── {model}/             # e.g. P320, G17
        └── {variant}.json   # e.g. 320CA-9-M18-MS.json
```

### CLI (`gun_db.py`, 83 lines)

Built with Click + click-shell. `BASE_PATH = Path('db/')` is hardcoded. Two name-conversion helpers bridge display names and filesystem paths:
- `h2e()`: hyphen → English title case (`sig-sauer` → `Sig Sauer`)
- `e2h()`: English → hyphen lowercase (`Sig Sauer` → `sig-sauer`)

Note: model directories are **not** lowercased — `e2h()` is only applied to manufacturer names. Model names like `G17`, `P320` are stored as-is.

## Naming Conventions

| Level | Convention | Example |
|---|---|---|
| Manufacturer dir | Lowercase, hyphens, no corporate suffixes | `smith-and-wesson` |
| Model dir | Exact base model name (case-sensitive) | `G17`, `P320` |
| Variant file | Manufacturer SKU + `.json` | `320CA-9-M18-MS.json` |

## Data Formats

### `logo.json`
```json
{
    "small": "https://...",    // 128x128 px (required)
    "large": "https://...",    // 256x256 px (optional, falls back to small)
    "banner": "https://..."    // 128x512 px (optional, omitted if missing)
}
```
Existing entries often use a simplified single-key `{"logo": "..."}` or `{"banner": "..."}` format; the three-key format is preferred for new entries.

### `{variant}.json`
```json
{
    "barrel length": "3.41 inches",
    "caliber": "9x19mm",
    "style": "handgun",
    "fire mode": "semi-auto",
    "decoder": "G43X-*****"
}
```
All five fields are required. `decoder` uses `*` as a wildcard and must uniquely identify the variant for de-duplication.

## CI / Release Automation

- **CI** (`.github/workflows/ci.yml`): Runs syntax check and `--help` across Python 3.8–3.12 on every PR and push to main.
- **Auto-release** (`.github/workflows/release.yml`): Triggers on merge to main; auto-increments patch version and generates release notes from merged PRs.
- **Weekly release** (`.github/workflows/weekly-release.yml`): Scheduled every Sunday at 00:00 UTC.
- **Dependabot**: Weekly checks for Python deps and GitHub Actions updates.
