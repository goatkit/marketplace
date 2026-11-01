# GoatFlow Plugin Marketplace

The curated plugin index for [GoatFlow](https://github.com/goatkit/goatflow), the open-source ticketing system.

This repository hosts `marketplace.json` — a single JSON file that the `gk` CLI fetches to discover, install, and update community plugins. No server infrastructure; GitHub Releases provides the download backend.

## Installing Plugins

```bash
# Search for a plugin
gk search knowledge-base

# Install by name
gk install goat-kb

# Check for updates on all installed plugins
gk update
```

### Signature Verification

GoatFlow runs only signed plugin code (from the release after 0.10.0). Each package needs two signatures from a trusted publisher:

- the package itself (`<name>.zip.sig`, published next to the ZIP), checked before anything is installed;
- the plugin binary (gRPC) or module (WASM), as `<binary>.sig` inside the ZIP, checked at install and every time the plugin starts.

The GoatKit publisher key is built into GoatFlow, so first-party plugins install with no configuration. For a plugin from another publisher, an administrator adds that publisher's public key first. The `public_key` in the index entry is shown in the error message, but is not trusted on its own:

```bash
# Hex public key, comma-separated for multiple
export GOATFLOW_TRUSTED_KEYS=<64-char-hex-public-key>

# Or load from a file (one per line, # for comments)
export GOATFLOW_TRUSTED_KEYS_FILE=/etc/goatflow/trusted_keys.txt
```

Only on a development system that runs plugins built from source, `GOATFLOW_REQUIRE_SIGNATURES=0` loads unsigned plugins.

### Custom Marketplace

Point `gk` at a private or self-hosted index:

```bash
export GOATFLOW_MARKETPLACE_URL=https://internal.example.com/marketplace.json
```

## Publishing a Plugin

### 1. Build the Plugin

For **WASM/template** plugins:

```bash
cd my-plugin/
gk build              # → dist/my-plugin-1.0.0.zip
```

For **gRPC** plugins (compiled binary), use your project's build pipeline. The ZIP must contain at minimum:
- `plugin.yaml` — the plugin manifest
- The compiled binary (name must match the `binary` field in `plugin.yaml`)

### 2. Generate a Signing Key

```bash
gk keys generate
# Private key: <hex>   — keep this secret, use it to sign releases
# Public key:  <hex>   — put this in your marketplace entry and README
```

Signing is required: GoatFlow refuses unsigned plugins.

### 3. Sign the Binary, Then the Package

Sign the binary (gRPC) or module (WASM) **before** packaging, so its `.sig` goes into the ZIP next to it. Then sign the ZIP:

```bash
gk sign my-plugin --key <hex-private-key>        # → my-plugin.sig (my-plugin.wasm.sig for WASM)
# package plugin.yaml, my-plugin and my-plugin.sig into my-plugin.zip
gk sign my-plugin.zip --key <hex-private-key>    # → my-plugin.zip.sig
```

Template and theme plugins have no binary; only the ZIP is signed.

### 4. Create a GitHub Release

```bash
gh release create v1.0.0 \
  my-plugin.zip \
  my-plugin.zip.sig \
  --title "my-plugin v1.0.0" \
  --notes "Initial release"
```

The release tag must match the version in `plugin.yaml`. Asset naming convention: `{plugin-name}.zip` and `{plugin-name}.zip.sig` (matching the `name` field in `plugin.yaml`).

#### GitHub Actions CI

Automate the build-sign-release pipeline on tag push (a gRPC plugin; for WASM, sign the `.wasm` module instead):

```yaml
name: Release Plugin
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/checkout@v5
        with:
          repository: goatkit/goatflow
          ref: v0.11.0          # pin to the GoatFlow release you build against
          path: goatflow
      - uses: actions/setup-go@v6
        with:
          go-version: '1.26.10'
      - name: Build, sign and package
        env:
          SIGNING_KEY: ${{ secrets.SIGNING_KEY }}
        run: |
          (cd goatflow && go build -o ../gk ./cmd/gk)
          go build -o my-plugin .
          ./gk sign my-plugin --key "$SIGNING_KEY"
          zip my-plugin.zip plugin.yaml my-plugin my-plugin.sig
          ./gk sign my-plugin.zip --key "$SIGNING_KEY"
      - uses: softprops/action-gh-release@v2
        with:
          files: |
            my-plugin.zip
            my-plugin.zip.sig
```

Store your private signing key as a repository secret named `SIGNING_KEY`. Never commit it.

### 5. Submit to the Marketplace

Open a pull request adding your plugin to [`marketplace.json`](marketplace.json):

```json
{
  "name": "my-plugin",
  "description": "One-line description of what it does",
  "author": "Your Name",
  "licence": "Apache-2.0",
  "homepage": "https://github.com/yourname/my-plugin",
  "repo": "yourname/my-plugin",
  "category": "utility",
  "tags": ["automation", "workflow"],
  "latest_version": "1.0.0",
  "min_host_version": "0.11.0",
  "runtime": "wasm",
  "verified": true,
  "dependencies": [],
  "public_key": "<your-64-char-hex-public-key-from-gk-keys-generate>"
}
```

Set `"verified": true` and include `"public_key"`. GoatFlow does not trust that key on its own: it names it when it refuses your plugin, so an administrator can check it against the key you publish (in your README, say) and add it to `GOATFLOW_TRUSTED_KEYS`. Without that, installs of your plugin are refused.

## Index Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Plugin name (must match `plugin.yaml` `name` field) |
| `description` | string | yes | Short description shown in search results |
| `author` | string | yes | Author or organisation name |
| `licence` | string | yes | SPDX licence identifier (e.g. `Apache-2.0`, `MIT`) |
| `homepage` | string | yes | Project URL |
| `repo` | string | yes | GitHub `owner/repo` for release downloads |
| `category` | string | yes | One of: `business`, `integration`, `theme`, `utility` |
| `tags` | string[] | yes | Search keywords |
| `latest_version` | string | yes | Semantic version (e.g. `1.2.0`) |
| `min_host_version` | string | no | Minimum GoatFlow version required |
| `runtime` | string | yes | One of: `wasm`, `grpc`, `template`, `theme` |
| `verified` | bool | yes | Whether the release is ed25519-signed |
| `dependencies` | string[] | no | Plugin names this depends on (installed first) |
| `public_key` | string | no | Ed25519 public key (hex, 64 chars) of the publisher; shown when an install is refused, so an administrator can choose to trust it (not trusted by itself) |

### Categories

| Category | Description |
|----------|-------------|
| `business` | Domain-specific business logic (KB, analytics, CRM) |
| `integration` | Third-party service integrations (email, SMS, webhooks) |
| `theme` | Visual themes and CSS overrides |
| `utility` | General-purpose tools (importers, exporters, maintenance) |

## Contributing

1. Fork this repository
2. Add or update your entry in `marketplace.json`
3. Open a pull request

PRs are reviewed for:
- Correct schema and field values
- Working GitHub Release with the tagged version
- Consistent naming between `plugin.yaml`, release assets, and the index entry

## Licence

Apache-2.0 — same as GoatFlow. Plugin entries retain the licence specified by their authors.
