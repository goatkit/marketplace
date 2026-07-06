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

Plugins can be signed with ed25519 keys. When a plugin entry includes a `public_key` field, `gk install` verifies the download automatically — no user configuration needed.

To enforce signatures for all installs (reject unsigned plugins):

```bash
export GOATFLOW_REQUIRE_SIGNATURES=1
```

To trust additional keys beyond those in the index (e.g. for private plugins):

```bash
# Hex public key, comma-separated for multiple
export GOATFLOW_TRUSTED_KEYS=<64-char-hex-public-key>

# Or load from a file (one per line, # for comments)
export GOATFLOW_TRUSTED_KEYS_FILE=/etc/goatflow/trusted_keys.txt
```

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

### 2. Generate a Signing Key (optional but recommended)

```bash
gk keys generate
# Private key: <hex>   — keep this secret, use it to sign releases
# Public key:  <hex>   — publish this so users can verify your plugins
```

### 3. Sign the Package

```bash
gk sign dist/my-plugin-1.0.0.zip --key <hex-private-key>
# → dist/my-plugin-1.0.0.zip.sig
```

### 4. Create a GitHub Release

```bash
gh release create v1.0.0 \
  dist/my-plugin-1.0.0.zip \
  dist/my-plugin-1.0.0.zip.sig \
  --title "my-plugin v1.0.0" \
  --notes "Initial release"
```

The release tag must match the version in `plugin.yaml`. Asset naming convention: `{plugin-name}.zip` (matching the `name` field in `plugin.yaml`).

#### GitHub Actions CI

Automate the build-sign-release pipeline on tag push:

```yaml
name: Release Plugin
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.25.10'
      - run: gk build
      - run: gk sign dist/*.zip --key ${{ secrets.SIGNING_KEY }}
      - uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/*.zip
            dist/*.zip.sig
```

Store your private signing key as a repository secret named `SIGNING_KEY`.

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
  "min_host_version": "0.8.0",
  "runtime": "wasm",
  "verified": true,
  "dependencies": [],
  "public_key": "<your-64-char-hex-public-key-from-make-keygen>"
}
```

Set `"verified": true` and include `"public_key"` when you have signed your release. Users with `GOATFLOW_REQUIRE_SIGNATURES=1` will reject entries without a valid key.

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
| `public_key` | string | no | Ed25519 public key (hex, 64 chars) for automatic signature verification |

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
