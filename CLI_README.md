# FedCore Platform CLI

A Rust-based command-line tool that replicates and enhances all the functionality of the bash scripts in the `scripts/` directory for managing Kubernetes platform deployments.

## Features

The CLI provides the following commands:

- **bootstrap** - Generate and deploy bootstrap configuration (Flux + Components)
- **build** - Build component artifacts for clusters
- **helm-manage** - Manage Helm charts (download, version discovery, updates, OCI push)
- **matrix** - Discover build matrix from cluster configs
- **validate** - Validate ytt templates, schemas, and cluster configs
- **mirror-flux** - Mirror Flux images to target registry
- **init** - Initialize new FedCore projects, clusters, or components

### Global Options

- `--verbose` or `-v` - Enable verbose output showing detailed progress and file operations

By default, the CLI uses minimal output showing only essential information. Use `--verbose` to see detailed progress indicators, file creation messages, configuration details, and step-by-step instructions.

**Example:**
```bash
# Minimal output (default)
fedcore build --cluster platform/clusters/mycluster

# Verbose output with details
fedcore -v build --cluster platform/clusters/mycluster
```

## Installation

### Prerequisites

- Rust toolchain (1.75 or later)
- Required external tools (same as bash scripts):
  - `ytt` - For YAML templating
  - `flux` - For Flux CLI operations
  - `kubectl` - For Kubernetes operations
  - `crane` - For container image mirroring (optional)
  - `helm` - For Helm operations (optional)

### Install from OCI Registry (recommended)

The fedcore bundle includes the CLI and all required tools (crane, flux, helm, kbld, kubectl, ytt).

**Linux:**
```bash
sudo oras pull ecp-non-prod.nexus-ecp.web.irs.gov/fedcore/cli-linux:latest -o /usr/local/bin/
sudo chmod +x /usr/local/bin/{fedcore,crane,flux,helm,kbld,kubectl,ytt}
```

**Windows (PowerShell):**
```powershell
oras pull ecp-non-prod.nexus-ecp.web.irs.gov/fedcore/cli-windows:latest -o C:\development\scoop\shims
```

### Build from Source

```bash
cd /home/ns2nb/repos/app-factory
cargo build --release

# The binary will be at: target/release/fedcore
```

### Install from Source

```bash
# Install to your PATH
cargo install --path .

# Or copy the binary
sudo cp target/release/fedcore /usr/local/bin/fedcore
```

## Usage

### Bootstrap Command

Generate and optionally deploy bootstrap configuration for a cluster:

```bash
# Generate bootstrap config (prints to stdout)
fedcore bootstrap -c platform/clusters/aws-csb-usgw1-dev-app

# Generate and deploy
fedcore bootstrap -c platform/clusters/aws-csb-usgw1-dev-app --deploy

# Save to file
fedcore bootstrap -c platform/clusters/aws-csb-usgw1-dev-app > bootstrap.yaml
```

**Options:**
- `-c, --cluster <PATH>` - Cluster directory (required)
- `-d, --deploy` - Deploy after generation

**Equivalent to:** `scripts/bootstrap.sh`

### Build Command

Build component artifacts for clusters:

```bash
# Build all artifacts
fedcore build --all

# Build single artifact
fedcore build -a platform/components/capsule -c platform/clusters/mycluster

# Build and push to OCI registry
export OCI_REGISTRY_USER="github-user"
export OCI_REGISTRY_PASS="$GITHUB_TOKEN"
fedcore build --all --push \
  -r ghcr.io/myorg \
  -v v1.0.0 \
  --repo-url https://github.com/org/repo \
  --ref main \
  --sha abc123
```

**Options:**
- `-a, --artifact <PATH>` - Artifact path
- `-c, --cluster <PATH>` - Cluster directory
- `--all` - Build all artifacts (default)
- `-p, --push` - Push to OCI registry
- `-r, --registry <URL>` - OCI registry URL (for push)
- `-v, --version <VERSION>` - Artifact version tag (for push)
- `--repo-url <URL>` - Git repository URL (for push)
- `--ref <NAME>` - Git ref name (for push)
- `--sha <HASH>` - Git commit SHA (for push)

**Environment Variables (for push):**
- `OCI_REGISTRY_USER` - Registry username
- `OCI_REGISTRY_PASS` - Registry password

**Equivalent to:** `scripts/build.sh`

### Helm Manage Command

Manage Helm charts - download, version discovery, updates, and OCI push:

```bash
# Download charts using current versions
fedcore helm-manage

# Download specific component
fedcore helm-manage -a platform/components/kyverno/

# Download and push to Nexus
fedcore helm-manage --push

# Full workflow: discover latest, update, download, and push
fedcore helm-manage --latest --update --push

# Mirror container images from charts
fedcore helm-manage --push --mirror-images
```

**Options:**
- `-a, --artifact <PATH>` - Specific artifact(s) (comma-separated, default: "all")
- `-d, --dir <PATH>` - Output directory (default: "./helm-charts")
- `-l, --latest` - Discover latest versions
- `-u, --update` - Update component YAML files (requires --latest)
- `-p, --push` - Push charts to OCI registry
- `-m, --mirror-images` - Mirror container images from charts

**Environment Variables:**
- `CLOUD_PROVIDER` - Set to 'all' (default), 'aws', 'azure', or 'none'
- `NEXUS_OCI_URL` - OCI registry URL for Helm charts (for --push)
- `NEXUS_USER` - Registry username (for --push)
- `NEXUS_PASS` - Registry password (for --push)
- `IMAGE_REGISTRY_URL` - Container image registry (for --mirror-images)

**Equivalent to:** `scripts/helm-manage.sh`

### Matrix Command

Discover the build matrix for OCI artifacts from cluster configs:

```bash
# Discover all components and clusters
fedcore matrix

# Output is JSON format with build_matrix and cluster_matrix
fedcore matrix | jq .
```

**Output:**
- JSON with `build_matrix` (component artifacts) and `cluster_matrix` (clusters for bootstrap)

**Equivalent to:** `scripts/matrix.sh`

### Validate Command

Validate ytt templates, schemas, and cluster configs:

```bash
# Run all validations
fedcore validate
```

**Checks:**
- Required tools (ytt, yq, yamllint)
- Cluster schema validity
- Component template validity
- Cluster configuration validity

**Equivalent to:** `scripts/validate.sh`

### Mirror Flux Command

Mirror Flux images to target registry (for airgapped environments):

```bash
# Mirror to default registry
fedcore mirror-flux

# Mirror to custom registry
fedcore mirror-flux -r my-registry.example.com
```

**Options:**
- `-r, --registry <URL>` - Target registry URL (default: "ecp-non-prod.nexus-ecp.web.irs.gov")

**Requirements:**
- `flux` CLI installed
- `crane` installed for image mirroring

**Equivalent to:** `scripts/mirror-flux-images.sh`

## Advantages over Bash Scripts

✅ **Better Performance** - Compiled binary, faster execution
✅ **Type Safety** - Rust's type system prevents many runtime errors
✅ **Better Error Handling** - Clear, structured error messages
✅ **Cross-platform** - Works on Linux, macOS, and Windows
✅ **Single Binary** - No need for multiple script files
✅ **Colored Output** - Better visual feedback
✅ **Structured Data** - JSON output for programmatic use
✅ **Progress Indicators** - Visual progress for long-running operations
✅ **Parallel Execution** - Potential for concurrent operations (future enhancement)
✅ **Easy Testing** - Rust's testing framework for unit/integration tests

## Development

### Project Structure

```
src/
├── main.rs              # CLI entry point and command definitions
├── commands/            # Command implementations
│   ├── mod.rs
│   ├── bootstrap.rs     # Bootstrap command
│   ├── build.rs         # Build command
│   ├── helm_manage.rs   # Helm manage command
│   ├── matrix.rs        # Matrix discovery
│   ├── validate.rs      # Validation command
│   └── mirror_flux.rs   # Mirror Flux images
└── utils/               # Shared utilities
    └── mod.rs
```

### Building

```bash
# Debug build
cargo build

# Release build (optimized)
cargo build --release

# Run tests
cargo test

# Run with logging
RUST_LOG=debug cargo run -- validate
```

### Adding New Features

1. Add command variant to `Commands` enum in `main.rs`
2. Create new module in `src/commands/`
3. Implement the `execute()` function
4. Add module to `src/commands/mod.rs`
5. Wire up in `main()` match statement

## Migration from Bash Scripts

The Rust CLI is a drop-in replacement for the bash scripts:

| Bash Script | Rust CLI Command |
|-------------|------------------|
| `bootstrap.sh` | `fedcore bootstrap` |
| `build.sh` | `fedcore build` |
| `helm-manage.sh` | `fedcore helm-manage` |
| `matrix.sh` | `fedcore matrix` |
| `validate.sh` | `fedcore validate` |
| `mirror-flux-images.sh` | `fedcore mirror-flux` |

All command-line arguments are preserved with the same semantics.

## Troubleshooting

### Tool Not Found Errors

Ensure all required tools are installed and in your PATH:
```bash
which ytt flux kubectl crane helm
```

### Permission Denied

Make the binary executable:
```bash
chmod +x target/release/fedcore
```

### Compilation Errors

Update Rust toolchain:
```bash
rustup update
```

## License

Same as the parent repository.

## Contributing

Contributions welcome! Please:
1. Run `cargo fmt` to format code
2. Run `cargo clippy` to check for common mistakes
3. Add tests for new functionality
4. Update this README with new features
