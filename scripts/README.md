# Build & Development Scripts

## Purpose

Automation scripts for common development tasks:
- Building and packaging
- Development workflow
- Testing and coverage
- Release automation
- Utilities and maintenance

## Scripts

### Development

**`dev.sh`** - Start development server with hot reload

### Building

**`build.sh`** - Build app for production (all platforms)

**`build-mac.sh`** - Build macOS-specific package

### Testing

**`test.sh`** - Run all tests (unit + E2E)

**`coverage.sh`** - Generate coverage report

### Release

**`release.sh`** - Create and publish release
- Bump version
- Generate changelog
- Build packages
- Create GitHub release

### Utilities

**`clean.sh`** - Remove build artifacts and caches

**`setup.sh`** - Initial project setup (install deps, etc.)

## Usage

All scripts should be executable and follow consistent patterns:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Script description
# Usage: ./script-name.sh [args]

# Implementation
```

## Implementation Notes

Scripts will be implemented by component agents as needed. Coordinate with package.json scripts.

## See Also

- [AGENTS.md](../AGENTS.md) - Build commands
- [package.json](../package.json) - npm scripts
