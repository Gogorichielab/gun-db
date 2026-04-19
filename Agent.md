# Agent.md - Principal Architecture Guide for gun-db

## Executive Summary

This document serves as the authoritative architectural reference for AI agents, developers, and contributors working on the **gun-db** project. As a principal architect's guide, it provides deep insights into the database architecture, design patterns, data governance, and best practices that ensure consistency, scalability, and maintainability.

**Target Audience**: AI agents, automated systems, senior developers, and architects
**Maintained By**: Principal Architecture Team
**Last Updated**: 2026-04-14

---

## Table of Contents

1. [Architectural Overview](#architectural-overview)
2. [Core Design Principles](#core-design-principles)
3. [Data Architecture](#data-architecture)
4. [File System Architecture](#file-system-architecture)
5. [API and CLI Architecture](#api-and-cli-architecture)
6. [Data Governance](#data-governance)
7. [Extensibility Patterns](#extensibility-patterns)
8. [Testing and Validation Strategy](#testing-and-validation-strategy)
9. [CI/CD Pipeline Architecture](#cicd-pipeline-architecture)
10. [Security and Compliance](#security-and-compliance)
11. [Agent Development Workflow](#agent-development-workflow)
12. [Common Patterns and Anti-Patterns](#common-patterns-and-anti-patterns)
13. [Troubleshooting Guide](#troubleshooting-guide)
14. [Future Architecture Considerations](#future-architecture-considerations)

---

## Architectural Overview

### System Purpose

The **gun-db** is a hierarchical, JSON-based, public domain database designed to:
- Provide structured firearm data for identification and cataloging systems
- Enable de-duplication through serial number decoding schemes
- Serve as a reference database for firearm tracking applications
- Support GunClear.io's mission of responsible firearm ownership documentation

### Architecture Philosophy

```
Simplicity > Complexity
Files > Database Engines
JSON > Proprietary Formats
Convention > Configuration
Public Domain > Restrictions
```

### Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Data Storage** | File System (JSON) | Zero-dependency, git-friendly, human-readable |
| **Data Format** | JSON | Universal support, schema-flexible, version-controllable |
| **CLI Interface** | Python 3.8-3.12 | Wide compatibility, simple dependencies |
| **CLI Framework** | Click + click-shell | Industry standard, interactive REPL support |
| **Version Control** | Git | Distributed, traceable, branch-friendly |
| **CI/CD** | GitHub Actions | Integrated, automated, free for public repos |
| **Dependency Management** | Dependabot | Automated security updates |

### Architectural Constraints

1. **No Database Engine**: Must remain file-based for simplicity and portability
2. **Public Domain Only**: All data must be CC0 1.0 Universal compatible
3. **Zero-Config**: No database setup, connection strings, or server processes
4. **Git-Native**: All changes tracked via version control
5. **Python 3.8+ Compatible**: Support modern Python without cutting-edge features

---

## Core Design Principles

### 1. Hierarchical Data Organization

The database follows a strict three-level hierarchy:

```
Manufacturer → Model → Variant
     ↓           ↓        ↓
  Directory  Directory  JSON File
```

**Rationale**:
- Natural mental model matching real-world organization
- Enables efficient file system traversal
- Supports partial database cloning
- Allows manufacturer-level branding (logos)

### 2. Convention Over Configuration

**Naming Conventions**:
- Lowercase with hyphens for directories (e.g., `smith-and-wesson`)
- No corporate suffixes (remove `inc.`, `llc.`, etc.)
- Base model names for directories (e.g., `P320`, not `P320-M18`)
- Manufacturer SKUs for variant files (e.g., `320CA-9-M18-MS-10.json`)

**Why**: Eliminates configuration files, reduces decision fatigue, ensures consistency

### 3. Immutable Data Structure

Each JSON file represents an **immutable variant specification**. To update:
1. Modify the existing file (tracked in git)
2. Create a new variant file (for actual product variations)

**Never**: Delete historical data without justification

### 4. Single Source of Truth

Each variant file is the **authoritative source** for:
- Physical specifications
- Serial number decoding patterns
- Firearm classification

**Logo files** are the authoritative source for manufacturer branding.

### 5. Fail-Safe Defaults

```python
# Example: Missing logo.json fallback
logo = read_logo_json() or {"logo": "default-icon.svg"}

# Example: Missing "large" key in logo.json
large_logo = logo.get("large") or logo.get("small")
```

**Principle**: System should degrade gracefully, never crash on missing optional data

---

## Data Architecture

### Entity-Relationship Model

```
┌─────────────────┐
│  Manufacturer   │
│  (Directory)    │
│                 │
│  - Name (path)  │
│  - Logo (JSON)  │
└────────┬────────┘
         │ 1
         │
         │ *
┌────────▼────────┐
│     Model       │
│  (Directory)    │
│                 │
│  - Name (path)  │
└────────┬────────┘
         │ 1
         │
         │ *
┌────────▼────────┐
│    Variant      │
│  (JSON File)    │
│                 │
│  - SKU (name)   │
│  - Specs (data) │
└─────────────────┘
```

### Schema Specifications

#### Logo Schema (`logo.json`)

**Canonical Format** (Recommended):
```json
{
  "small": "https://cdn.example.com/logo-128.png",
  "large": "https://cdn.example.com/logo-256.png",
  "banner": "https://cdn.example.com/banner-128x512.png"
}
```

**Simplified Format** (Acceptable):
```json
{
  "logo": "https://cdn.example.com/logo.svg"
}
```

**Field Specifications**:

| Field | Type | Size | Required | Purpose |
|-------|------|------|----------|---------|
| `small` | URL | 128x128px | Yes (canonical) | Icon-size logo |
| `large` | URL | 256x256px | No | Detailed logo |
| `banner` | URL | 128x512px | No | Marketing banner |
| `logo` | URL | Flexible | Yes (simplified) | Single logo reference |

**Fallback Logic**:
```python
def get_logo_sizes(logo_json):
    if "logo" in logo_json:
        # Simplified format
        return {
            "small": logo_json["logo"],
            "large": logo_json["logo"],
            "banner": logo_json.get("banner")
        }
    else:
        # Canonical format
        return {
            "small": logo_json["small"],
            "large": logo_json.get("large", logo_json["small"]),
            "banner": logo_json.get("banner")
        }
```

#### Variant Schema (`{sku}.json`)

**Canonical Structure**:
```json
{
  "barrel length": "4.7 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "ABC-XYZ-*****"
}
```

**Field Specifications**:

| Field | Type | Required | Constraints | Example Values |
|-------|------|----------|-------------|----------------|
| `barrel length` | String | Yes | Include units | "4.7 inches", "16 inches" |
| `caliber` | String | Yes | Standard nomenclature | "9mm", ".45 ACP", "5.56x45mm NATO" |
| `style` | String | Yes | Lowercase | "handgun", "long gun", "shotgun", "rifle" |
| `fire mode` | String | Yes | Lowercase | "semi-auto", "bolt action", "pump action", "revolver" |
| `decoder` | String | Yes | Pattern with `*` | "G17G5-*****", "SW-MP9-******" |

**Validation Rules**:
1. All fields are required
2. Decoder pattern must be unique within the model
3. Caliber must use industry-standard nomenclature
4. Units must be included for measurements
5. Values must be publicly verifiable

### Data Normalization

**Anti-Pattern**: Storing redundant data
```json
// ❌ Don't do this
{
  "manufacturer": "Sig Sauer",  // Already in path
  "model": "P320",               // Already in path
  "sku": "320CA-9-M18-MS-10",   // Already in filename
  "caliber": "9mm"
}
```

**Correct Pattern**: Store only variant-specific data
```json
// ✅ Do this
{
  "barrel length": "3.9 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "SIG-P320-M18-*****"
}
```

**Rationale**:
- Prevents inconsistencies (e.g., filename says P320 but JSON says P226)
- Reduces file size
- Follows DRY (Don't Repeat Yourself) principle
- Path is the source of truth for identity

---

## File System Architecture

### Directory Structure

```
gun-db/
├── db/                              # Database root (sacred boundary)
│   ├── {manufacturer}/              # Manufacturer namespace
│   │   ├── logo.json                # Branding data
│   │   └── {model}/                 # Model namespace
│   │       ├── {variant-1}.json     # Variant specification
│   │       ├── {variant-2}.json
│   │       └── {variant-n}.json
│   ├── {manufacturer-2}/
│   └── ...
├── docs/                            # Extended documentation
│   └── AGENT.md                     # Detailed agent guide
├── .github/
│   ├── workflows/                   # CI/CD automation
│   │   ├── ci.yml                   # Continuous integration
│   │   ├── release.yml              # Release automation
│   │   └── weekly-release.yml       # Scheduled releases
│   ├── ISSUE_TEMPLATE/              # Issue templates
│   └── PULL_REQUEST_TEMPLATE.md     # PR template
├── gun_db.py                        # CLI entry point
├── requirements.txt                 # Python dependencies
├── README.md                        # Public documentation
├── CONTRIBUTING.md                  # Contribution guide
├── Agent.md                         # This file (architectural guide)
├── LICENSE                          # CC0 1.0 Universal
└── .gitignore                       # VCS exclusions
```

### Path Construction Patterns

#### Manufacturer Path
```python
def get_manufacturer_path(display_name: str) -> Path:
    """
    Convert display name to file system path.

    Examples:
        "Sig Sauer" → "sig-sauer"
        "Smith & Wesson" → "smith-and-wesson"
        "Heckler & Koch, Inc." → "heckler-and-koch"
    """
    normalized = display_name.lower()
    normalized = normalized.replace(' & ', '-and-')
    normalized = normalized.replace(' ', '-')
    normalized = normalized.replace(',', '')
    normalized = re.sub(r'\b(inc|llc|ltd|corp)\b\.?', '', normalized)
    normalized = normalized.strip('-')
    return Path('db') / normalized
```

#### Model Path
```python
def get_model_path(manufacturer: str, model: str) -> Path:
    """
    Get path to model directory.

    Note: Model names are NOT transformed (case-sensitive).
    Examples:
        ("Sig Sauer", "P320") → "db/sig-sauer/P320"
        ("Glock", "G17") → "db/glock/G17"
    """
    mfr_path = get_manufacturer_path(manufacturer)
    return mfr_path / model  # Model preserves original case
```

#### Variant Path
```python
def get_variant_path(manufacturer: str, model: str, sku: str) -> Path:
    """
    Get path to variant JSON file.

    Examples:
        ("Sig Sauer", "P320", "320CA-9-M18-MS-10")
        → "db/sig-sauer/P320/320CA-9-M18-MS-10.json"
    """
    model_path = get_model_path(manufacturer, model)
    return model_path / f"{sku}.json"
```

### File System Invariants

**Invariant 1**: Every manufacturer directory contains exactly one `logo.json`
```bash
# Validation check
for mfr in db/*/; do
  [ -f "$mfr/logo.json" ] || echo "Missing: $mfr/logo.json"
done
```

**Invariant 2**: Model directories contain only JSON files (no subdirectories)
```bash
# Validation check
find db/*/*/* -type d && echo "ERROR: Models should not have subdirectories"
```

**Invariant 3**: Variant files must have `.json` extension
```bash
# Validation check
find db/*/*/* -type f ! -name "*.json" && echo "ERROR: Non-JSON file in model directory"
```

---

## API and CLI Architecture

### CLI Design Pattern

The CLI uses a **REPL (Read-Eval-Print Loop)** pattern via `click-shell`:

```
┌─────────────────────────────────────┐
│  User Input                         │
│  🔫 ~ models "Sig Sauer"           │
└───────────────┬─────────────────────┘
                ↓
┌───────────────▼─────────────────────┐
│  Command Parser (Click)             │
│  - Argument validation              │
│  - Type coercion                    │
└───────────────┬─────────────────────┘
                ↓
┌───────────────▼─────────────────────┐
│  Name Transformer                   │
│  "Sig Sauer" → "sig-sauer"         │
│  (h2e / e2h functions)              │
└───────────────┬─────────────────────┘
                ↓
┌───────────────▼─────────────────────┐
│  File System Query                  │
│  Iterate db/sig-sauer/*             │
└───────────────┬─────────────────────┘
                ↓
┌───────────────▼─────────────────────┐
│  Response Formatter                 │
│  Transform paths back to display    │
└───────────────┬─────────────────────┘
                ↓
┌───────────────▼─────────────────────┐
│  Output to Terminal                 │
│  P210\nP220\nP226\nP320            │
└─────────────────────────────────────┘
```

### Core Functions

#### Name Transformation Functions

```python
def h2e(phrase: str) -> str:
    """
    Hyphen to English: Convert file system name to display name.

    Args:
        phrase: Hyphenated lowercase string

    Returns:
        Title-cased string with spaces

    Examples:
        "sig-sauer" → "Sig Sauer"
        "smith-and-wesson" → "Smith And Wesson"
    """
    return phrase.replace('-', ' ').title()


def e2h(phrase: str) -> str:
    """
    English to Hyphen: Convert display name to file system name.

    Args:
        phrase: Natural language manufacturer name

    Returns:
        Lowercase hyphenated string

    Examples:
        "Sig Sauer" → "sig-sauer"
        "Smith & Wesson" → "smith-&-wesson"  # Note: '&' preserved
    """
    return phrase.replace(' ', '-').lower()
```

**Important**: These functions are **symmetric but not perfect**:
- `h2e(e2h("Smith & Wesson"))` → "Smith-&-Wesson" (not original)
- Use for display purposes, not canonical name storage

#### CLI Commands Architecture

##### 1. `makes` Command
**Purpose**: List all manufacturers
**Implementation**:
```python
@gun_db.command()
def makes():
    """List all manufacturers in the database."""
    makes = sorted(h2e(p.name) for p in BASE_PATH.iterdir() if p.is_dir())
    click.echo("\n".join(makes))
```

**Algorithm**:
1. Scan `db/` directory
2. Filter for directories only
3. Transform directory names (h2e)
4. Sort alphabetically
5. Print one per line

**Edge Cases**:
- Empty database → No output
- Non-directory files in `db/` → Ignored

##### 2. `models` Command
**Purpose**: List models for a manufacturer
**Implementation**:
```python
@gun_db.command()
@click.argument('make')
def models(make):
    """List all models for a given manufacturer."""
    models = sorted(h2e(p.name) for p in (BASE_PATH / e2h(make)).iterdir() if p.is_dir())
    click.echo("\n".join(models))
```

**Algorithm**:
1. Convert manufacturer name (e2h)
2. Scan manufacturer directory
3. Filter for directories (exclude logo.json)
4. Transform directory names (h2e)
5. Sort and print

**Edge Cases**:
- Manufacturer not found → FileNotFoundError
- Only logo.json in directory → Empty output

##### 3. `variants` Command
**Purpose**: List variants for a model
**Implementation**:
```python
@gun_db.command()
@click.argument('make')
@click.argument('model')
def variants(make, model):
    """List all variants for a given model."""
    model_path = (BASE_PATH / e2h(make) / model)
    variants = sorted(p.name.replace('.json', '') for p in model_path.iterdir() if p.is_file())
    click.echo("\n".join(variants))
```

**Algorithm**:
1. Construct model directory path
2. Scan for files only
3. Strip `.json` extension
4. Sort and print

**Edge Cases**:
- Model directory doesn't exist → FileNotFoundError
- Empty model directory → Empty output

##### 4. `get_info` Command
**Purpose**: Retrieve variant specifications
**Implementation**:
```python
@gun_db.command()
@click.argument('make')
@click.argument('model')
@click.argument('variant')
def get_info(make, model, variant):
    """Get detailed information about a specific variant."""
    file_path = (BASE_PATH / e2h(make) / model / variant).with_suffix('.json')
    with open(file_path, 'r') as f:
        click.echo(json.loads(f.read() or '{}'))
```

**Algorithm**:
1. Construct variant file path
2. Open and read JSON
3. Parse and print (Python dict format)

**Edge Cases**:
- File not found → FileNotFoundError
- Empty file → `{}`
- Invalid JSON → JSONDecodeError

##### 5. `set_info` Command
**Purpose**: Update variant parameters
**Implementation**:
```python
@gun_db.command()
@click.argument('make')
@click.argument('model')
@click.argument('variant')
@click.argument('param')
@click.argument('value')
def set_info(make, model, variant, param, value):
    """Update a parameter for a specific variant."""
    file_path = (BASE_PATH / e2h(make) / model / variant).with_suffix('.json')

    # Read current data
    with open(file_path, 'r') as f:
        parameters = json.loads(f.read() or '{}')

    # Display before state
    click.echo(f"[{make}] {model} ({variant}):")
    click.echo("Before")
    click.echo(parameters)

    # Update parameter
    parameters[param] = value

    # Display after state
    click.echo("After")
    click.echo(parameters)

    # Write back
    with open(file_path, 'w') as f:
        f.write(json.dumps(parameters))
```

**Algorithm**:
1. Read existing JSON
2. Display current state
3. Update parameter (add or modify)
4. Display new state
5. Write back to file

**Edge Cases**:
- New parameter → Added to dict
- Existing parameter → Overwritten
- Empty file → Creates new dict with single parameter

**Security Concerns**:
- ⚠️ No validation on `param` or `value`
- ⚠️ Could introduce non-standard fields
- ⚠️ Could set invalid values (e.g., caliber = "banana")

---

## Data Governance

### Data Quality Tiers

#### Tier 1: Manufacturer Data
**Requirements**:
- ✅ Publicly traded company OR established manufacturer (>10 years)
- ✅ Website or public documentation available
- ✅ Logo images are public domain, CC0, or properly licensed
- ✅ Manufacturer still in business OR historical significance

**Rejection Criteria**:
- ❌ Startup companies (<2 years old) without established products
- ❌ Copyrighted logos without permission
- ❌ Companies under active litigation for IP disputes

#### Tier 2: Model Data
**Requirements**:
- ✅ Commercially available model OR historical model with documentation
- ✅ Model name matches manufacturer's official nomenclature
- ✅ At least one variant with complete specifications available

**Rejection Criteria**:
- ❌ Concept models never manufactured
- ❌ Military-only models with no public specifications
- ❌ Custom one-off builds

#### Tier 3: Variant Data
**Requirements**:
- ✅ All five required fields present and accurate
- ✅ Decoder pattern is unique within the model
- ✅ Specifications match manufacturer documentation
- ✅ Publicly available information (no NDA-restricted specs)

**Rejection Criteria**:
- ❌ Missing required fields
- ❌ Decoder pattern collision with existing variant
- ❌ Specifications from rumor/speculation
- ❌ Proprietary or confidential information

### Data Verification Workflow

```
┌─────────────────────┐
│  New Data Proposed  │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  Source Check       │
│  - Public info?     │
│  - Verifiable?      │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  Format Validation  │
│  - JSON valid?      │
│  - Schema correct?  │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  Uniqueness Check   │
│  - Decoder unique?  │
│  - No duplicates?   │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  License Check      │
│  - CC0 compatible?  │
│  - Public domain?   │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  Manual Review      │
│  (for edge cases)   │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  Merge to Main      │
└─────────────────────┘
```

### Decoder Pattern Design

**Purpose**: Unique identification for de-duplication

**Pattern Rules**:
1. Use manufacturer's actual serial number format when known
2. Use `*` for variable characters
3. Ensure pattern uniquely identifies THIS variant
4. Keep pattern as specific as possible without false negatives

**Examples**:

```
Good Decoder Patterns:
- "SIG-P320-M18-*****"     # Manufacturer prefix + model + variant + serial
- "G17G5-*********"        # Model + generation + serial
- "SW-MP9-2.0-******"      # Prefix + model + version + serial

Problematic Patterns:
- "*****"                  # Too generic, not unique
- "SIG-****-****-****"     # Too vague, matches multiple models
- "EXACTLY-12345"          # No wildcards, only matches one serial number
```

**Testing Decoders**:
```python
def test_decoder_uniqueness():
    """Ensure all decoders in a model are unique."""
    for model_dir in get_all_models():
        decoders = []
        for variant in model_dir.glob("*.json"):
            data = json.loads(variant.read_text())
            decoders.append(data["decoder"])

        # Check for duplicates
        if len(decoders) != len(set(decoders)):
            raise ValueError(f"Duplicate decoders in {model_dir}")
```

### Caliber Nomenclature Standards

Use **industry-standard** caliber designations:

**Preferred Formats**:
- Metric: `9mm`, `7.62mm`, `5.56mm`
- Imperial: `.45 ACP`, `.308 Winchester`, `.223 Remington`
- NATO: `5.56x45mm NATO`, `7.62x51mm NATO`
- Full designation: `9x19mm Parabellum`

**Avoid**:
- Colloquialisms: ~~"nine mil"~~, ~~"forty-five"~~
- Ambiguous: ~~"9mm"~~ when multiple exist (specify `9x19mm` vs `9x18mm`)
- Marketing names without specs: ~~"Magnum"~~ (specify `.44 Magnum`)

---

## Extensibility Patterns

### Adding New Manufacturers

**Pattern**: Scaffold structure before adding data

```bash
# 1. Create manufacturer directory
mkdir -p db/manufacturer-name

# 2. Create logo file FIRST (required)
cat > db/manufacturer-name/logo.json << EOF
{
  "logo": "https://example.com/logo.svg"
}
EOF

# 3. Now safe to add models
mkdir -p db/manufacturer-name/ModelName

# 4. Add variants
cat > db/manufacturer-name/ModelName/variant-sku.json << EOF
{
  "barrel length": "...",
  "caliber": "...",
  "style": "...",
  "fire mode": "...",
  "decoder": "..."
}
EOF
```

### Adding New Models

**Pattern**: Model directory with multiple variants

```bash
# 1. Create model directory under manufacturer
mkdir -p db/sig-sauer/P365

# 2. Add all known variants
# Variant 1: Standard
cat > db/sig-sauer/P365/365-9-BXR3.json << EOF
{
  "barrel length": "3.1 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "SIG-P365-STD-*****"
}
EOF

# Variant 2: XL
cat > db/sig-sauer/P365/365XL-9-BXR3.json << EOF
{
  "barrel length": "3.7 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "SIG-P365-XL-*****"
}
EOF
```

### Adding New Variants

**Pattern**: Ensure uniqueness and completeness

```python
def add_variant(manufacturer, model, sku, specifications):
    """Add a new variant with validation."""
    # 1. Validate path exists
    model_path = get_model_path(manufacturer, model)
    if not model_path.exists():
        raise ValueError(f"Model directory does not exist: {model_path}")

    # 2. Validate variant doesn't exist
    variant_path = model_path / f"{sku}.json"
    if variant_path.exists():
        raise ValueError(f"Variant already exists: {variant_path}")

    # 3. Validate specifications
    required = {"barrel length", "caliber", "style", "fire mode", "decoder"}
    if not required.issubset(specifications.keys()):
        missing = required - specifications.keys()
        raise ValueError(f"Missing required fields: {missing}")

    # 4. Validate decoder uniqueness
    for existing in model_path.glob("*.json"):
        existing_data = json.loads(existing.read_text())
        if existing_data["decoder"] == specifications["decoder"]:
            raise ValueError(f"Decoder collision with {existing.name}")

    # 5. Write variant file
    variant_path.write_text(json.dumps(specifications, indent=4))
    print(f"✅ Added variant: {variant_path}")
```

### Extending CLI Commands

**Pattern**: Add new commands following Click conventions

```python
@gun_db.command()
@click.argument('manufacturer')
def count_variants(manufacturer):
    """Count total variants for a manufacturer."""
    mfr_path = BASE_PATH / e2h(manufacturer)
    count = 0

    for model_dir in mfr_path.iterdir():
        if model_dir.is_dir():
            count += len(list(model_dir.glob("*.json")))

    click.echo(f"{manufacturer}: {count} variants")
```

---

## Testing and Validation Strategy

### Validation Levels

#### Level 1: Syntax Validation
```python
def validate_json_syntax():
    """Ensure all JSON files are valid."""
    for json_file in Path("db").rglob("*.json"):
        try:
            json.loads(json_file.read_text())
        except json.JSONDecodeError as e:
            print(f"❌ Invalid JSON: {json_file}\n   {e}")
            return False
    return True
```

#### Level 2: Schema Validation
```python
def validate_variant_schema(variant_data, file_path):
    """Validate variant against schema."""
    required_fields = {
        "barrel length", "caliber", "style", "fire mode", "decoder"
    }

    # Check for required fields
    missing = required_fields - set(variant_data.keys())
    if missing:
        print(f"❌ Missing fields in {file_path}: {missing}")
        return False

    # Check for unknown fields
    unknown = set(variant_data.keys()) - required_fields
    if unknown:
        print(f"⚠️  Unknown fields in {file_path}: {unknown}")

    # Validate decoder pattern
    if "*" not in variant_data["decoder"]:
        print(f"❌ Decoder missing wildcard in {file_path}")
        return False

    return True
```

#### Level 3: Business Logic Validation
```python
def validate_decoder_uniqueness():
    """Ensure decoders are unique within each model."""
    for model_dir in Path("db").glob("*/*"):
        if not model_dir.is_dir():
            continue

        decoders = {}
        for variant_file in model_dir.glob("*.json"):
            data = json.loads(variant_file.read_text())
            decoder = data.get("decoder")

            if decoder in decoders:
                print(f"❌ Duplicate decoder '{decoder}':")
                print(f"   {decoders[decoder]}")
                print(f"   {variant_file}")
                return False

            decoders[decoder] = variant_file

    return True
```

### Testing Checklist

**For New Manufacturers**:
- [ ] Directory name is lowercase with hyphens
- [ ] `logo.json` exists and is valid JSON
- [ ] Logo URLs are accessible (HTTP 200)
- [ ] Logo images are appropriate size (or scalable SVG)
- [ ] At least one model exists

**For New Models**:
- [ ] Model directory name matches manufacturer nomenclature
- [ ] At least one variant exists
- [ ] All variants have unique decoders

**For New Variants**:
- [ ] All five required fields present
- [ ] Decoder pattern includes wildcards (`*`)
- [ ] Decoder is unique within model
- [ ] Caliber uses standard nomenclature
- [ ] Barrel length includes units
- [ ] Style is lowercase
- [ ] Fire mode is lowercase
- [ ] JSON is properly formatted (indented)

---

## CI/CD Pipeline Architecture

### Workflow Overview

```
┌─────────────────────┐
│  Developer Push/PR  │
└──────────┬──────────┘
           ↓
┌──────────▼──────────┐
│  ci.yml Triggered   │
│  - Syntax check     │
│  - Dependency test  │
│  - CLI test         │
└──────────┬──────────┘
           ↓
    ┌──────▼──────┐
    │  All Pass?  │
    └──┬───────┬──┘
       │       │
    ❌ │       │ ✅
       ↓       ↓
  ┌────▼───┐ ┌▼────────────┐
  │ Reject │ │ Allow Merge │
  └────────┘ └┬────────────┘
              ↓
    ┌─────────▼──────────┐
    │  Merge to Main     │
    └─────────┬──────────┘
              ↓
    ┌─────────▼──────────┐
    │ release.yml        │
    │ - Generate notes   │
    │ - Create release   │
    │ - Tag version      │
    └────────────────────┘
```

### CI Workflow (`ci.yml`)

**Purpose**: Validate all changes before merge

**Key Steps**:
1. **Matrix Testing**: Python 3.8, 3.9, 3.10, 3.11, 3.12
2. **Dependency Installation**: `pip install -r requirements.txt`
3. **Syntax Check**: `python -m py_compile gun_db.py`
4. **CLI Test**: Verify tool loads without errors

**Configuration**:
```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    strategy:
      matrix:
        python-version: [3.8, 3.9, '3.10', 3.11, 3.12]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: python -m py_compile gun_db.py
```

### Release Workflow (`release.yml`)

**Purpose**: Automate versioning and release notes

**Trigger**: Merge to main branch

**Key Steps**:
1. Generate release notes from merged PRs
2. Auto-increment version tag
3. Create GitHub release
4. Publish release notes

**Version Strategy**: Semantic versioning (v1.0.0, v1.0.1, v1.1.0, etc.)

### Dependabot Configuration

**Purpose**: Automated dependency updates

**Schedules**:
- Python dependencies: Weekly (Monday)
- GitHub Actions: Weekly (Monday)

**Auto-Merge Policy**:
- Patch updates (1.0.x): Auto-merge if CI passes
- Minor updates (1.x.0): Manual review required
- Major updates (x.0.0): Manual review required

---

## Security and Compliance

### Threat Model

**Threat 1: Malicious Data Injection**
- **Risk**: Attacker submits PR with malicious JSON
- **Mitigation**: Manual PR review, JSON validation in CI
- **Impact**: Low (no code execution from JSON data)

**Threat 2: Copyrighted Content**
- **Risk**: Logo images or data violate copyright
- **Mitigation**: Manual review, source verification
- **Impact**: Medium (legal liability)

**Threat 3: Dependency Vulnerabilities**
- **Risk**: `click` or `click-shell` have security issues
- **Mitigation**: Dependabot auto-updates
- **Impact**: Low (CLI tool, no network operations)

**Threat 4: Path Traversal**
- **Risk**: Malicious manufacturer name like `../../../etc/passwd`
- **Mitigation**: Path validation, use of `Path` library
- **Impact**: Low (file system operations are read-only in CLI)

### Security Best Practices

#### Input Validation
```python
def validate_manufacturer_name(name: str) -> bool:
    """Validate manufacturer name for security."""
    # Reject path traversal attempts
    if ".." in name or "/" in name or "\\" in name:
        return False

    # Reject absolute paths
    if name.startswith("/") or ":" in name:
        return False

    # Only allow alphanumeric, hyphens, spaces
    if not re.match(r'^[a-zA-Z0-9\s\-&]+$', name):
        return False

    return True
```

#### Safe File Operations
```python
def safe_read_json(path: Path) -> dict:
    """Safely read JSON file with error handling."""
    # Ensure path is within db/ directory
    if not path.is_relative_to(Path("db")):
        raise ValueError("Path outside database directory")

    # Check file exists
    if not path.exists():
        raise FileNotFoundError(f"File not found: {path}")

    # Read and parse
    try:
        return json.loads(path.read_text())
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON in {path}: {e}")
```

### Compliance (CC0 1.0 Universal)

**Requirements**:
- ✅ All data must be public domain or CC0 compatible
- ✅ No copyrighted images without explicit permission
- ✅ No proprietary specifications or trade secrets
- ✅ No personal information (PII)

**Verification Process**:
1. PR submitter attests data is public domain
2. Maintainer verifies sources are public
3. For questionable cases, contact manufacturer for permission

---

## Agent Development Workflow

### Workflow for AI Agents

**Phase 1: Analysis**
```
1. Read README.md for high-level understanding
2. Read Agent.md (this file) for architectural details
3. Read CONTRIBUTING.md for contribution guidelines
4. Explore db/ structure to understand current state
```

**Phase 2: Planning**
```
5. Identify exact task (new manufacturer, model, variant, or code change)
6. Verify data sources are public and CC0-compatible
7. Check for duplicates or conflicts
8. Plan file/directory structure
```

**Phase 3: Implementation**
```
9. Create directories if needed (manufacturer, model)
10. Create or update JSON files
11. Validate JSON syntax
12. Test with CLI tool (./gun_db.py)
```

**Phase 4: Validation**
```
13. Run syntax validation (python -m json.tool file.json)
14. Run CLI commands to verify data is queryable
15. Check decoder uniqueness
16. Verify logo URLs are accessible
```

**Phase 5: Submission**
```
17. Create commit with descriptive message
18. Push to branch
19. Create PR with detailed description
20. Address review feedback
```

### Common Agent Tasks

#### Task 1: Add a New Manufacturer

```bash
# 1. Research manufacturer
# - Verify they exist and are publicly known
# - Find publicly accessible logo
# - Verify CC0 compatibility

# 2. Create structure
MANUFACTURER="ruger"  # lowercase, hyphens
mkdir -p db/$MANUFACTURER

# 3. Add logo
cat > db/$MANUFACTURER/logo.json << 'EOF'
{
  "logo": "https://example.com/ruger-logo.svg"
}
EOF

# 4. Validate
python -m json.tool db/$MANUFACTURER/logo.json

# 5. Test
./gun_db.py
# 🔫 ~ makes
# (verify manufacturer appears in list)
```

#### Task 2: Add a New Model

```bash
# 1. Identify manufacturer and model name
MANUFACTURER="sig-sauer"
MODEL="P365"

# 2. Create model directory
mkdir -p db/$MANUFACTURER/$MODEL

# 3. Add first variant (establishes the model)
cat > db/$MANUFACTURER/$MODEL/365-9-BXR3.json << 'EOF'
{
  "barrel length": "3.1 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "SIG-P365-STD-*****"
}
EOF

# 4. Validate
python -m json.tool db/$MANUFACTURER/$MODEL/365-9-BXR3.json

# 5. Test
./gun_db.py
# 🔫 ~ models "Sig Sauer"
# (verify P365 appears)
# 🔫 ~ variants "Sig Sauer" P365
# (verify variant appears)
```

#### Task 3: Add a Variant to Existing Model

```bash
# 1. Research variant specifications
# - Find official manufacturer SKU
# - Verify specs (barrel length, caliber, etc.)
# - Determine serial number decoder pattern

# 2. Create variant file
MANUFACTURER="glock"
MODEL="G17"
VARIANT="G17-Gen5-MOS"

cat > db/$MANUFACTURER/$MODEL/$VARIANT.json << 'EOF'
{
  "barrel length": "4.49 inches",
  "caliber": "9x19mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "G17G5M-*****"
}
EOF

# 3. Validate JSON
python -m json.tool db/$MANUFACTURER/$MODEL/$VARIANT.json

# 4. Check decoder uniqueness
grep -r "G17G5M-\*\*\*\*\*" db/$MANUFACTURER/$MODEL/
# (should only appear once)

# 5. Test
./gun_db.py
# 🔫 ~ get_info "Glock" G17 "G17-Gen5-MOS"
```

#### Task 4: Update CLI Tool

```bash
# Example: Add a search command

# 1. Edit gun_db.py
# Add new command following Click conventions

@gun_db.command()
@click.argument('search_term')
def search(search_term):
    """Search for models containing the search term."""
    results = []
    for mfr_dir in BASE_PATH.iterdir():
        if not mfr_dir.is_dir():
            continue

        for model_dir in mfr_dir.iterdir():
            if not model_dir.is_dir():
                continue

            if search_term.lower() in model_dir.name.lower():
                results.append(f"{h2e(mfr_dir.name)} - {model_dir.name}")

    click.echo("\n".join(sorted(results)))

# 2. Test
python -m py_compile gun_db.py
./gun_db.py
# 🔫 ~ search "320"
# (should show Sig Sauer - P320, etc.)

# 3. Update documentation if needed
```

### Error Handling Patterns

**Pattern 1: Graceful Degradation**
```python
def get_logo(manufacturer):
    """Get logo with fallback."""
    try:
        logo_path = get_manufacturer_path(manufacturer) / "logo.json"
        logo_data = json.loads(logo_path.read_text())
        return logo_data.get("logo") or logo_data.get("small") or "default.svg"
    except (FileNotFoundError, json.JSONDecodeError):
        return "default.svg"
```

**Pattern 2: User-Friendly Errors**
```python
@gun_db.command()
@click.argument('make')
def models(make):
    """List models for a manufacturer."""
    try:
        mfr_path = BASE_PATH / e2h(make)
        if not mfr_path.exists():
            click.echo(f"❌ Manufacturer '{make}' not found.")
            click.echo(f"💡 Try: makes (to see all manufacturers)")
            return

        models = sorted(p.name for p in mfr_path.iterdir() if p.is_dir())
        if not models:
            click.echo(f"No models found for {make}")
            return

        click.echo("\n".join(models))
    except Exception as e:
        click.echo(f"❌ Error: {e}")
```

---

## Common Patterns and Anti-Patterns

### ✅ Design Patterns (Best Practices)

#### Pattern 1: Path Construction
**Good**:
```python
from pathlib import Path

# Use Path library for safety and portability
variant_path = Path("db") / e2h(mfr) / model / f"{variant}.json"
```

**Bad**:
```python
# String concatenation leads to path traversal vulnerabilities
variant_path = f"db/{mfr}/{model}/{variant}.json"
```

#### Pattern 2: JSON Handling
**Good**:
```python
# Read with error handling
try:
    data = json.loads(file.read_text())
except json.JSONDecodeError as e:
    logger.error(f"Invalid JSON in {file}: {e}")
    data = {}
```

**Bad**:
```python
# No error handling
data = json.loads(open(file).read())
```

#### Pattern 3: Name Transformation
**Good**:
```python
# Use provided utility functions
display_name = h2e(directory_name)
file_name = e2h(display_name)
```

**Bad**:
```python
# Manual string manipulation
display_name = directory_name.replace("-", " ").title()
file_name = display_name.lower().replace(" ", "-")
```

### ❌ Anti-Patterns (Avoid)

#### Anti-Pattern 1: Data Redundancy
**Bad**:
```json
{
  "manufacturer": "Sig Sauer",
  "model": "P320",
  "sku": "320CA-9-M18-MS-10",
  "barrel length": "3.9 inches",
  ...
}
```
**Why**: Manufacturer and model are already in the file path. This creates inconsistency risk.

**Good**:
```json
{
  "barrel length": "3.9 inches",
  "caliber": "9mm",
  ...
}
```

#### Anti-Pattern 2: Magic Strings
**Bad**:
```python
if firearm["style"] == "handgun":  # What are valid values?
    ...
```

**Good**:
```python
VALID_STYLES = {"handgun", "long gun", "shotgun", "rifle"}

if firearm["style"] in VALID_STYLES:
    ...
else:
    raise ValueError(f"Invalid style: {firearm['style']}")
```

#### Anti-Pattern 3: Hardcoded Paths
**Bad**:
```python
logo = json.load(open("db/sig-sauer/logo.json"))
```

**Good**:
```python
logo_path = get_manufacturer_path("Sig Sauer") / "logo.json"
logo = json.loads(logo_path.read_text())
```

---

## Troubleshooting Guide

### Issue 1: "Manufacturer Not Found"

**Symptom**:
```
🔫 ~ models "Sig Sauer"
FileNotFoundError: db/sig-sauer does not exist
```

**Diagnosis**:
```bash
# Check exact directory name
ls db/ | grep -i sig

# Possible output: "sig sauer" (with space, not hyphen)
```

**Solution**:
```bash
# Directory names must use hyphens
mv "db/sig sauer" db/sig-sauer

# Or if truly missing:
mkdir db/sig-sauer
cat > db/sig-sauer/logo.json << 'EOF'
{"logo": "https://..."}
EOF
```

### Issue 2: "Invalid JSON"

**Symptom**:
```
JSONDecodeError: Expecting ',' delimiter: line 4 column 5
```

**Diagnosis**:
```bash
# Validate JSON syntax
python -m json.tool db/sig-sauer/P320/variant.json
```

**Common Causes**:
1. Trailing comma: `{"a": 1,}` ❌
2. Missing quotes: `{key: "value"}` ❌
3. Single quotes: `{'key': 'value'}` ❌

**Solution**:
```bash
# Use proper JSON syntax
cat > file.json << 'EOF'
{
  "barrel length": "3.9 inches",
  "caliber": "9mm"
}
EOF
```

### Issue 3: "Decoder Collision"

**Symptom**:
Two variants have the same decoder pattern.

**Diagnosis**:
```bash
# Find duplicate decoders
cd db/sig-sauer/P320
jq -r '.decoder' *.json | sort | uniq -d
```

**Solution**:
Make decoder more specific:
```json
# Before (too generic)
{"decoder": "SIG-P320-*****"}

# After (variant-specific)
{"decoder": "SIG-P320-M18-*****"}
```

### Issue 4: "CLI Command Not Found"

**Symptom**:
```
🔫 ~ get_info "Sig Sauer" P320 "320CA-9-M18-MS-10"
Error: No such command "get_info"
```

**Diagnosis**:
```bash
# Check available commands
./gun_db.py
🔫 ~ help
```

**Solution**:
- Verify command is defined in `gun_db.py`
- Check for typos in command name
- Ensure command is decorated with `@gun_db.command()`

---

## Future Architecture Considerations

### Scalability Considerations

**Current Limitations**:
- File system scans don't scale beyond ~10,000 manufacturers
- No indexing for fast lookups
- No query optimization

**Potential Solutions** (if needed):
1. **SQLite index**: Build on-demand index for fast queries
2. **JSON Schema**: Formal schema validation
3. **API layer**: REST API on top of file system

**Decision Point**: Only add complexity if:
- Database exceeds 1,000 manufacturers OR
- Query performance becomes user-impacting (>1 second) OR
- New use cases require querying capabilities

### Extension Opportunities

#### 1. Variant Images
**Proposal**: Add optional `images` field to variants
```json
{
  "barrel length": "3.9 inches",
  "caliber": "9mm",
  "style": "handgun",
  "fire mode": "semi-auto",
  "decoder": "SIG-P320-M18-*****",
  "images": {
    "profile": "https://...",
    "top": "https://...",
    "detail": "https://..."
  }
}
```

**Pros**: Enhanced visual reference
**Cons**: Increases data size, licensing complexity

#### 2. Historical Data
**Proposal**: Track specification changes over time
```json
{
  "current": {
    "barrel length": "3.9 inches",
    ...
  },
  "history": [
    {
      "valid_from": "2017-01-01",
      "valid_to": "2020-12-31",
      "barrel length": "4.0 inches",
      ...
    }
  ]
}
```

**Pros**: Captures product evolution
**Cons**: Complexity, harder to query

#### 3. Ammunition Compatibility
**Proposal**: Add optional ammunition cross-reference
```json
{
  "caliber": "9mm",
  "compatible_ammunition": [
    "9x19mm Parabellum",
    "9mm NATO",
    "9mm Luger"
  ]
}
```

**Pros**: Useful for users
**Cons**: Difficult to maintain accuracy

### Migration Strategy (if needed)

**Scenario**: Database grows beyond file system limits

**Phase 1: Add Index (No Breaking Changes)**
```python
# Build on-demand index
def build_index():
    """Create SQLite index for fast queries."""
    conn = sqlite3.connect("gun_db.index")
    conn.execute("""
        CREATE TABLE IF NOT EXISTS variants (
            manufacturer TEXT,
            model TEXT,
            variant TEXT,
            caliber TEXT,
            style TEXT,
            fire_mode TEXT,
            decoder TEXT
        )
    """)

    # Populate from JSON files
    for variant_file in Path("db").rglob("*/*/*.json"):
        data = json.loads(variant_file.read_text())
        # Insert into index...
```

**Phase 2: Dual Read (JSON + Index)**
```python
def get_variant_info(manufacturer, model, variant):
    """Get variant info, preferring index if available."""
    if index_exists():
        return query_index(manufacturer, model, variant)
    else:
        return read_json_file(manufacturer, model, variant)
```

**Phase 3: Deprecate Direct JSON Reads** (only if necessary)

---

## Conclusion

This architectural guide provides a comprehensive foundation for AI agents and developers working on the gun-db project. Key takeaways:

1. **Simplicity First**: File-based architecture is intentional and sufficient
2. **Convention Over Configuration**: Follow established patterns
3. **Data Quality**: Verify sources, ensure uniqueness, validate schemas
4. **Extensibility**: Add features incrementally without breaking changes
5. **Documentation**: Keep this guide updated as architecture evolves

### Quick Reference

**Essential Files**:
- `gun_db.py` - CLI implementation
- `db/` - Database directory
- `README.md` - User documentation
- `CONTRIBUTING.md` - Contribution guide
- `docs/AGENT.md` - Detailed technical guide
- `Agent.md` - This architectural reference (you are here)

**Essential Commands**:
```bash
# Install dependencies
pip install -r requirements.txt

# Start CLI
./gun_db.py

# Validate JSON
python -m json.tool file.json

# Test CLI
python -m py_compile gun_db.py
```

**Essential Patterns**:
```python
# Name transformation
display = h2e(filesystem_name)
filesystem = e2h(display_name)

# Path construction
path = Path("db") / e2h(mfr) / model / f"{variant}.json"

# Safe JSON read
data = json.loads(path.read_text())
```

### Support and Feedback

- **Issues**: https://github.com/Gogorichielab/gun-db/issues
- **Email**: admin@gunclear.io
- **Documentation**: See `/docs` directory

---

**Document Metadata**:
- **Version**: 1.0.0
- **Last Updated**: 2026-04-14
- **Maintainer**: Principal Architecture Team
- **License**: CC0 1.0 Universal (Public Domain)
