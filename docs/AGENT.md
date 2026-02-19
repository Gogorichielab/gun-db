# Agent Guide: gun-db Repository

## Overview

The **gun-db** repository is a structured database of firearm makes, models, and variants. It organizes information about firearms from various manufacturers in a hierarchical JSON-based format. This guide is designed to help developers, contributors, and automated agents understand and work with this repository effectively.

## Repository Purpose

This database serves as a comprehensive reference for:
- Firearm manufacturer information and logos
- Firearm models and their variants
- Technical specifications (caliber, barrel length, fire mode, etc.)
- Serial number decoding schemes for de-duplication

## Technology Stack

- **Language**: Python 3.x
- **CLI Framework**: Click and click-shell
- **Data Format**: JSON
- **Version Control**: Git

## Repository Structure

```
gun-db/
├── db/                          # Main database directory
│   ├── {manufacturer}/          # One directory per manufacturer
│   │   ├── logo.json           # Manufacturer logo links
│   │   └── {model}/            # Model directories
│   │       └── {variant}.json  # Variant specification files
├── gun_db.py                   # Python CLI tool
├── requirements.txt            # Python dependencies
├── README.md                   # Project documentation
└── LICENSE                     # CC-1.0 Universal license
```

### Directory Naming Conventions

- **Manufacturer names**: Full legal name in lowercase with hyphens replacing spaces
  - Example: `smith-and-wesson`, `sig-sauer`
  - Excludes corporate identifiers like `inc.`, `llc.`

- **Model names**: Base model name (not the full variant)
  - Example: Under `glock/`: models like `G17`, `G19`, etc.

- **Variant files**: Manufacturer SKU or specific model number
  - Example: `210A-9-TGT.json`, `320CA-9-M18-MS.json`

## Data Format Specifications

### 1. Logo Files (`logo.json`)

Located at the root of each manufacturer directory.

**Structure**:
```json
{
    "small": "https://path.to.host/128x128-logo.ext",
    "large": "https://path.to.host/256x256-logo.ext",
    "banner": "https://path.to.host/128x512-banner.ext"
}
```

**Guidelines**:
- **small** (required): Simple manufacturer logo, 128x128 px
- **large** (optional): Detailed logo or logo with name, 256x256 px
  - If missing, small logo is expanded
- **banner** (optional): Full logo with name/slogan or popular model image, 128x512 px
  - If missing, banner is not displayed

**Example**:
```json
{
    "logo": "https://us.glock.com/.../glock-logo.svg"
}
```

### 2. Variant Files (`{model-number}.json`)

Located in `{manufacturer}/{model}/` directories.

**Structure**:
```json
{
    "barrel length": "23 inches",
    "caliber": ".223 Remington",
    "style": "long gun",
    "fire mode": "bolt action",
    "decoder": "RM-LN-*****"
}
```

**Fields**:
- **barrel length**: Physical barrel measurement
- **caliber**: Ammunition type/caliber
- **style**: Firearm category (e.g., "handgun", "long gun", "rifle")
- **fire mode**: Operating mechanism (e.g., "semi-auto", "bolt action", "pump action")
- **decoder**: Serial number pattern using `*` as wildcard for unique identification

## CLI Tool: gun_db.py

The repository includes a Python CLI for interacting with the database.

### Installation

```bash
pip install -r requirements.txt
```

### Available Commands

Start the interactive shell:
```bash
./gun_db.py
```

**Commands**:

1. **makes** - List all manufacturers
   ```bash
   🔫 ~ makes
   ```

2. **models** - List models for a manufacturer
   ```bash
   🔫 ~ models "Sig Sauer"
   ```

3. **variants** - List variants for a specific model
   ```bash
   🔫 ~ variants "Sig Sauer" P320
   ```

4. **get_info** - Get details about a specific variant
   ```bash
   🔫 ~ get_info "Sig Sauer" P320 "320CA-9-M18-MS"
   ```

5. **set_info** - Update a parameter for a variant
   ```bash
   🔫 ~ set_info "Sig Sauer" P320 "320CA-9-M18-MS" caliber "9mm"
   ```

### Command Behavior

- Name conversion: The tool automatically converts between display names and file system names
  - Display: "Sig Sauer" → Filesystem: `sig-sauer`
  - Uses `h2e()` (hyphen-to-English) and `e2h()` (English-to-hyphen) functions

## Contributing Guidelines

### Adding New Data

1. **New Manufacturer**:
   ```bash
   mkdir -p db/{manufacturer-name}
   # Create logo.json with manufacturer logo links
   ```

2. **New Model**:
   ```bash
   mkdir -p db/{manufacturer-name}/{model-name}
   ```

3. **New Variant**:
   ```bash
   # Create {variant-sku}.json in the model directory
   # Include all relevant specifications
   ```

### Data Quality Standards

- ✅ **Do**:
  - Use accurate, publicly available information
  - Follow the JSON structure exactly
  - Use consistent terminology
  - Include decoder patterns for serial number identification
  - Test additions using the CLI tool

- ❌ **Don't**:
  - Include proprietary or restricted information
  - Use copyrighted images without permission
  - Create duplicate entries
  - Leave required fields empty

### Pull Request Process

1. Fork the repository
2. Add or update data following the structure guidelines
3. Test your changes with `gun_db.py`
4. Submit a PR with a clear description of additions/changes
5. Address any review feedback

### Reporting Issues

If you find:
- Missing information
- Incorrect data
- Licensing concerns
- Non-public information

Contact: [admin@gunclear.io](mailto:admin@gunclear.io) or open a GitHub issue

## License

This database is released under **CC-1.0 Universal (Public Domain)**. All data should be publicly available information suitable for common use.

## Current Manufacturers

As of this documentation, the database includes:

- Armalite
- Barrett
- Benelli
- Beretta
- Browning
- Bushmaster Firearms
- Colt
- Glock
- Mossberg
- Remington
- Ruger
- Sig Sauer
- Smith & Wesson
- Springfield Armory
- Walther
- Winchester

## For Automated Agents

### Quick Facts

- **Primary Language**: Python
- **Data Format**: JSON (structured)
- **Entry Point**: `gun_db.py` (CLI)
- **Database Location**: `db/` directory
- **Naming**: Lowercase with hyphens, no corporate suffixes

### Working with This Repository

1. **Reading Data**:
   - Navigate hierarchically: manufacturer → model → variant
   - Parse JSON files for structured data
   - Use `logo.json` for manufacturer branding

2. **Adding Data**:
   - Follow directory structure strictly
   - Validate JSON format before committing
   - Ensure decoder patterns are unique

3. **Validation**:
   - Use the CLI to test database queries
   - Verify file paths match naming conventions
   - Check JSON syntax validity

### Common Tasks

**Task**: Add a new firearm variant
```python
# 1. Determine the correct path
# db/{manufacturer}/{model}/{variant}.json

# 2. Create JSON with required fields
{
    "barrel length": "...",
    "caliber": "...",
    "style": "...",
    "fire mode": "...",
    "decoder": "..."
}

# 3. Test with CLI
./gun_db.py
🔫 ~ get_info "{Manufacturer}" {Model} "{Variant}"
```

**Task**: Update manufacturer logo
```bash
# Edit db/{manufacturer}/logo.json
# Ensure URLs are accessible and properly sized
```

## Support and Questions

For technical questions about the repository structure or contributions:
- Open a GitHub issue
- Email: [admin@gunclear.io](mailto:admin@gunclear.io)

---

*This agent guide provides comprehensive documentation for working with the gun-db repository. For end-user documentation, see the main [README.md](../README.md).*
