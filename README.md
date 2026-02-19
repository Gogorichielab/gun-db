# gun-db
List of Firearm Makes, Models, and Variants

## Structure

Database is structured as JSON documents, organized according to the following:
```
{manufacturer}/
  logo.json
  {model}/
    {model number}.json
```

`{manufacturer}` is the full legal name of the maker of the particular model,
without any corporation-specific identification such as `inc.` or `llc.`
Links to their logos are included via the `logo.json` files.

`{model}` is the *base model* of a particular class of models that a manufacturer makes.
Variants such as caliber types, barrel lengths, fire modes, etc. go inside the `json` files
that describe them via their `{model number}` (e.g. the manufacturer's SKU).

## Data Formats

### `logo.json`

This file contains links to a small square, and large square, and a banner photo.

Small square photos should just contain the manufacturer's logo as simply as possible.
They should be 128x128 px in size.

Large square photos can either be the manufacturer's simple logo, or a logo and a name below it.
This photo can show more detail than the small one, or it can just be the small one expanded.
If this key doesn't exist, the large is just the small one expanded.
They should be 256x256 px in size.

Banner photos should contain the manufacturer's full logo and name, and can contain a slogan or catchphrase.
This photo will be displayed above their large square photo, so it should be different.
It can also just show a picture of their most popular model, or any other marketing material.
If this key doesn't exist, the banner is not displayed.
They should be 128x512 px in size.

The file should be in JSON format, with the following layout:
```json
{
    "small":"https://path.to.host/path-to-file.ext",
    "large":"https://path.to.host/path-to-file.ext",
    "banner":"https://path.to.host/path-to-file.ext"
}
```

### `{model number}.json`

This contains information about a particular variant of a firearm made by the manufacturer.
Information should include chambered caliber type, barrel length, type (handgun, long gun, etc.),
fire modes (semi-auto, bolt action, etc.), and any other relevant identifying information.
It should also contain the serial number decoding scheme used to determine the make and model
of a particular serial number (used for de-duplication detection).
The format of the decoding scheme should uniquely map to that variant,
using the `*` character to denote a wildcard.

The file should be in JSON format, with the following layout:
```json
{
    "barrel length":"23 inches",
    "caliber":".223 Remington",
    "style":"long gun",
    "fire mode":"bolt action",
    "decoder":"RM-LN-*****"
}
```

## Automation & Maintenance

This repository uses automated tools to maintain code quality and dependencies:

### Dependabot
- **Python Dependencies**: Automated weekly checks for updates to Python packages
- **GitHub Actions**: Automated weekly checks for updates to workflow actions

### GitHub Actions Workflows
- **CI**: Runs on all pull requests and pushes to main
  - Tests Python compatibility (3.8 - 3.12)
  - Validates dependency installation
  - Checks Python syntax
  - Tests CLI tool functionality

- **Release Notes**: Automatically triggered on merge to main
  - Generates release notes from merged PRs
  - Creates a new GitHub release with auto-incrementing version tags
  - Aggregates all changes since the last release

## Contributing

We welcome contributions! Please see our [CONTRIBUTING.md](CONTRIBUTING.md) guide for detailed information on:
- How to add manufacturers, models, and variants
- Development setup and testing
- Pull request guidelines
- Data quality standards

For quick questions or concerns about licensing, contact us at [admin@gunclear.io](mailto:admin@gunclear.io).
