# Contributing to gun-db

Thank you for your interest in contributing to the gun-db project! This document provides guidelines and instructions for contributing to our firearm database.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [How to Contribute](#how-to-contribute)
  - [Contributing Data](#contributing-data)
  - [Contributing Code](#contributing-code)
- [Development Setup](#development-setup)
- [Data Quality Standards](#data-quality-standards)
- [Pull Request Process](#pull-request-process)
- [Testing Your Changes](#testing-your-changes)
- [Reporting Issues](#reporting-issues)
- [License](#license)

## Code of Conduct

We are committed to providing a welcoming and inclusive environment for all contributors. Please be respectful and professional in all interactions within this project.

## Getting Started

The gun-db repository is a structured database of firearm makes, models, and variants organized in a hierarchical JSON format. Before contributing, please:

1. Read the [README.md](README.md) to understand the project structure
2. Review the [docs/AGENT.md](docs/AGENT.md) for detailed technical documentation
3. Familiarize yourself with the existing data format and conventions

## How to Contribute

There are two main ways to contribute to this project:

1. **Contributing Data**: Adding or updating firearm information, manufacturer logos, or specifications
2. **Contributing Code**: Improving the Python CLI tool or adding new features

### Contributing Data

#### Adding a New Manufacturer

1. Create a new directory in `db/` using the manufacturer's full legal name:
   ```bash
   mkdir -p db/manufacturer-name
   ```
   - Use lowercase letters
   - Replace spaces with hyphens
   - Omit corporate identifiers (inc., llc., etc.)
   - Example: `smith-and-wesson`, `sig-sauer`

2. Create a `logo.json` file with manufacturer branding:
   ```json
   {
       "small": "https://path.to.host/128x128-logo.ext",
       "large": "https://path.to.host/256x256-logo.ext",
       "banner": "https://path.to.host/128x512-banner.ext"
   }
   ```
   
   **Logo specifications:**
   - `small`: Simple manufacturer logo, 128x128 px (required)
   - `large`: Detailed logo or logo with name, 256x256 px (optional)
   - `banner`: Full logo with name/slogan, 128x512 px (optional)

#### Adding a New Model

Create a model directory under the manufacturer:
```bash
mkdir -p db/manufacturer-name/model-name
```

Use the base model name (e.g., `P320`, `G19`, `AR-15`), not full variant names.

#### Adding a New Variant

1. Create a JSON file named after the manufacturer's SKU or model number:
   ```bash
   db/manufacturer-name/model-name/variant-sku.json
   ```

2. Include all required specifications:
   ```json
   {
       "barrel length": "4.7 inches",
       "caliber": "9mm",
       "style": "handgun",
       "fire mode": "semi-auto",
       "decoder": "ABC-123-*****"
   }
   ```

   **Required fields:**
   - `barrel length`: Physical barrel measurement
   - `caliber`: Ammunition type/caliber
   - `style`: Firearm category (handgun, long gun, shotgun, etc.)
   - `fire mode`: Operating mechanism (semi-auto, bolt action, pump action, etc.)
   - `decoder`: Serial number pattern using `*` as wildcard for unique identification

### Contributing Code

If you want to improve the Python CLI tool (`gun_db.py`) or add new features:

1. Fork the repository
2. Create a new branch for your feature
3. Make your changes following Python best practices
4. Ensure your code works with Python 3.8 through 3.12
5. Test your changes thoroughly using the CLI
6. Submit a pull request with a clear description

#### Code Style Guidelines

- Follow PEP 8 Python style guidelines
- Use descriptive variable and function names
- Add docstrings for new functions
- Keep functions focused and single-purpose
- Maintain consistency with existing code style

## Development Setup

### Prerequisites

- Python 3.8 or higher
- Git

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/Gogorichielab/gun-db.git
   cd gun-db
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. Test the CLI tool:
   ```bash
   ./gun_db.py
   ```

### Using the CLI Tool

The CLI provides several commands to interact with the database:

```bash
# Start interactive shell
./gun_db.py

# List all manufacturers
🔫 ~ makes

# List models for a manufacturer
🔫 ~ models "Sig Sauer"

# List variants for a model
🔫 ~ variants "Sig Sauer" P320

# Get variant information
🔫 ~ get_info "Sig Sauer" P320 "320CA-9-M18-MS"

# Update variant information
🔫 ~ set_info "Sig Sauer" P320 "320CA-9-M18-MS" caliber "9mm"
```

## Data Quality Standards

### ✅ Do

- Use accurate, publicly available information
- Follow the JSON structure exactly as documented
- Use consistent terminology and formatting
- Include complete and accurate specifications
- Provide unique decoder patterns for serial numbers
- Test additions using the CLI tool before submitting
- Use properly licensed images and resources
- Cite sources when possible

### ❌ Don't

- Include proprietary or restricted information
- Use copyrighted images without permission
- Create duplicate entries
- Leave required fields empty or use placeholder values
- Include personal opinions or subjective information
- Add data that is not publicly available
- Include information that violates CC0 1.0 Universal license terms

## Pull Request Process

1. **Fork and Branch**: Fork the repository and create a descriptive branch name
   ```bash
   git checkout -b add-manufacturer-name
   ```

2. **Make Changes**: Add or update data following the structure guidelines

3. **Test Locally**: Verify your changes work correctly
   ```bash
   ./gun_db.py
   # Test relevant commands
   ```

4. **Commit**: Write clear, descriptive commit messages
   ```bash
   git add .
   git commit -m "Add [Manufacturer] [Model] variants"
   ```

5. **Push**: Push your branch to your fork
   ```bash
   git push origin add-manufacturer-name
   ```

6. **Create PR**: Open a pull request with:
   - Clear title describing the change
   - Detailed description of what was added/modified
   - Any relevant context or sources
   - Screenshots (if adding logos or visual elements)

7. **Review**: Address any feedback from maintainers

8. **CI/CD**: Ensure all automated checks pass:
   - Python syntax validation
   - Dependency installation
   - CLI functionality tests

### Pull Request Guidelines

- Keep PRs focused on a single change or related set of changes
- Provide context and rationale for changes
- Reference any related issues
- Be responsive to review feedback
- Ensure all CI checks pass before requesting review

## Testing Your Changes

### Manual Testing

Always test your changes before submitting:

1. **Data additions**: Use the CLI to query your new data
   ```bash
   ./gun_db.py
   🔫 ~ get_info "Manufacturer" Model "Variant"
   ```

2. **Code changes**: Run the CLI and test all affected commands
   ```bash
   python gun_db.py --help
   python -m py_compile gun_db.py
   ```

### Validation Checklist

Before submitting a PR, verify:

- [ ] File and directory names follow naming conventions
- [ ] JSON files are properly formatted and valid
- [ ] All required fields are present and accurate
- [ ] Logo URLs are accessible and properly sized
- [ ] Decoder patterns are unique and use correct format
- [ ] CLI commands work correctly with new data
- [ ] No duplicate entries exist
- [ ] Changes follow licensing requirements

## Reporting Issues

If you find problems or have suggestions:

### For Data Issues

- Missing manufacturer, model, or variant
- Incorrect specifications or information
- Broken logo links
- Duplicate entries

**Create a GitHub Issue** with:
- Clear description of the problem
- Location in the database (path to file)
- Suggested correction (if applicable)
- Source references

### For Code Issues

- Bugs in the CLI tool
- Feature requests
- Performance issues
- Documentation improvements

**Create a GitHub Issue** with:
- Clear description of the issue
- Steps to reproduce (for bugs)
- Expected vs actual behavior
- Python version and environment details

### For Licensing Concerns

If you believe any data or images may not be public information suitable for CC0 1.0 Universal license:

- Email: [admin@gunclear.io](mailto:admin@gunclear.io)
- Or open a GitHub issue marked as confidential

## License

By contributing to gun-db, you agree that your contributions will be licensed under the **CC0 1.0 Universal (Public Domain)** license. This means:

- Your contributions become part of the public domain
- Anyone can use the data for any purpose
- No attribution is required (though appreciated)
- You must have the right to contribute the information

All data should be publicly available information suitable for common use.

## Questions and Support

- **Technical questions**: Open a GitHub issue
- **General inquiries**: Email [admin@gunclear.io](mailto:admin@gunclear.io)
- **Documentation**: See [README.md](README.md) and [docs/AGENT.md](docs/AGENT.md)

---

Thank you for contributing to gun-db! Your efforts help maintain a comprehensive, accurate, and freely available firearm database.
