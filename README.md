# Sirr (سر)

[![License: BSL 1.1](https://img.shields.io/badge/License-BSL%201.1-blue.svg)](LICENSE)
[![CI](https://github.com/yourorg/sirr/actions/workflows/ci.yml/badge.svg)](https://github.com/yourorg/sirr/actions/workflows/ci.yml)

**Ephemeral secret management for developers who believe secrets should have expiration dates.**

Sirr is a self-hosted secret vault where every secret knows when to die. Set expiration times, limit read counts, and let your secrets clean themselves up. Single binary, zero runtime dependencies.

---

## Why Sirr?

Traditional secret managers treat everything as permanent. Sirr treats secrets as **temporary by default**:

| Use case | Config |
|---|---|
| API key for a contractor | `--ttl 30d` |
| CI/CD deploy token | `--reads 1` |
| Debug database credentials | `--ttl 1h --reads 3` |
| Credentials shared with an AI assistant | `--ttl 2h --reads 1` |

**The rule**: if you can't name a date when a secret should stop working, it's already a liability.

---

## The AI-Era Problem

Ever pasted a database URL into Claude to debug a query? With traditional secrets:

- ❌ The credential lives in AI training data forever
- ❌ Manually revoking after every session is tedious
- ❌ Creating temp accounts takes 10+ minutes

**With Sirr:**

```bash
# Create a one-time database credential
sirr push DEBUG_DB="postgres://user:pass@host/db" --reads 1 --ttl 2h

# Share the key name with Claude
# → "Claude, use DEBUG_DB from my sirr vault"

# After Claude reads it once: gone. Not in logs, not in training data.
```

Zero paranoia. Maximum productivity.

---

## Quick Start

### Server

**Docker (recommended):**

```bash
docker run -d \
  -p 8080:8080 \
  -v ./sirr-data:/data \
  -e SIRR_MASTER_KEY=your-secret-token \
  ghcr.io/yourorg/sirr
```

**Binary:**

```bash
# Download from releases
curl -L https://github.com/yourorg/sirr/releases/latest/download/sirr-x86_64-unknown-linux-musl.tar.gz | tar xz
SIRR_MASTER_KEY=your-token ./sirr serve
```

**From source:**

```bash
git clone https://github.com/yourorg/sirr
cd sirr
cargo build --release --bin sirr
SIRR_MASTER_KEY=your-token ./target/release/sirr serve
```

### CLI

**Homebrew:**

```bash
brew tap sirr/sirr https://github.com/yourorg/sirr
brew install sirr
```

**npx (Node.js):**

```bash
npx @sirr/sdk push API_KEY="sk-..." --ttl 1h
```

---

## Usage

### Store Secrets

```bash
# Single key-value with TTL
sirr push API_KEY="sk-..." --ttl 7d

# One-time read (burns after retrieval)
sirr push DB_PASSWORD="secret" --reads 1

# Both constraints
sirr push TOKEN="xyz" --ttl 1h --reads 3

# Push entire .env file
sirr push .env --ttl 24h
```

### Retrieve Secrets

```bash
# Single secret (prints value to stdout)
sirr get API_KEY

# Pull all secrets into .env
sirr pull .env

# Run a command with secrets injected as env vars
sirr run -- node app.js
sirr run -- docker-compose up

# Print shareable reference URL
sirr share API_KEY
# → http://localhost:8080/secrets/API_KEY
```

### Manage

```bash
sirr list                  # Show all active secrets (metadata only)
sirr delete API_KEY        # Burn a secret immediately
sirr prune                 # Delete all expired secrets now
```

---

## Claude Code Integration (MCP)

Sirr ships an MCP server so Claude can read and write secrets directly:

```bash
# Install
npm install -g @sirr/mcp

# Configure in Claude Code (~/.claude/settings.json or project .mcp.json)
```

**.mcp.json:**

```json
{
  "mcpServers": {
    "sirr": {
      "command": "sirr-mcp",
      "env": {
        "SIRR_SERVER": "http://localhost:8080",
        "SIRR_TOKEN": "your-secret-token"
      }
    }
  }
}
```

Once configured, Claude can:

```
You: "Get me the DATABASE_URL from sirr"
Claude: [calls get_secret("DATABASE_URL")] → returns the value

You: "Push my Stripe key to sirr, burn after 1 read"
Claude: [calls push_secret("STRIPE_KEY", "sk_live_...", reads=1)]

You: "What secrets do I have expiring today?"
Claude: [calls list_secrets()] → filters by expires_at
```

You can also reference secrets inline using the `sirr:KEYNAME` syntax:

```
"Deploy using sirr:DEPLOY_TOKEN — it should already be there"
```

See [`packages/mcp/README.md`](packages/mcp/README.md) for full configuration.

---

## HTTP API

All routes except `/health` require `Authorization: Bearer <SIRR_MASTER_KEY>`.

### `GET /health`
```json
{ "status": "ok" }
```

### `POST /secrets`
```json
// Request
{ "key": "API_KEY", "value": "secret", "ttl_seconds": 86400, "max_reads": 1 }

// Response 201
{ "key": "API_KEY" }
```

### `GET /secrets/:key`
Retrieves the value and increments the read counter. Deletes the record if the read limit is reached.
```json
{ "key": "API_KEY", "value": "secret" }
// 404 if expired, burned, or not found
```

### `GET /secrets`
Returns metadata only — values are never included.
```json
{
  "secrets": [
    { "key": "API_KEY", "created_at": 1700000000, "expires_at": 1700086400, "max_reads": 1, "read_count": 0 }
  ]
}
```

### `DELETE /secrets/:key`
```json
{ "deleted": true }
```

### `POST /prune`
Immediately removes all expired secrets.
```json
{ "pruned": 3 }
```

---

## Node.js SDK

```bash
npm install @sirr/sdk
```

```typescript
import { SirrClient } from '@sirr/sdk'

const sirr = new SirrClient({
  server: 'http://localhost:8080',
  token: process.env.SIRR_TOKEN!,
})

// Push a secret
await sirr.push('API_KEY', 'sk-...', { ttl: 3600, reads: 1 })

// Retrieve
const value = await sirr.get('API_KEY')  // null if burned/expired

// Inject all secrets as env vars for the duration of a function
await sirr.withSecrets(async () => {
  // process.env.API_KEY is set here
  await runTests()
})

// Pull all to a map
const secrets = await sirr.pullAll()
```

---

## Configuration

All configuration is via environment variables — no config files.

| Variable | Default | Description |
|---|---|---|
| `SIRR_MASTER_KEY` | **required** | Bearer token + encryption key seed |
| `SIRR_LICENSE_KEY` | — | License key (required for >100 secrets) |
| `SIRR_PORT` | `8080` | HTTP port |
| `SIRR_HOST` | `0.0.0.0` | Bind address |
| `SIRR_DATA_DIR` | Platform default¹ | Storage directory |
| `SIRR_LOG_LEVEL` | `info` | Log level (`trace`/`debug`/`info`/`warn`/`error`) |

**CLI variables:**

| Variable | Default | Description |
|---|---|---|
| `SIRR_SERVER` | `http://localhost:8080` | Server base URL |
| `SIRR_TOKEN` | — | Same value as `SIRR_MASTER_KEY` on the server |

¹ Platform defaults: `~/.local/share/sirr/` (Linux), `~/Library/Application Support/sirr/` (macOS), `%APPDATA%\sirr\` (Windows). Override with `SIRR_DATA_DIR`. Docker: mount `/data` and set `SIRR_DATA_DIR=/data`.

---

## Architecture

```
CLI (Rust) / Node SDK / MCP Server
           ↓  HTTP (bearer token)
      axum REST API (Rust)
           ↓
    redb embedded database (single file: sirr.db)
           ↓
  ChaCha20Poly1305 encrypted values
  (key derived via Argon2id from SIRR_MASTER_KEY + sirr.salt)
```

**Security model:**
- `SIRR_MASTER_KEY` seeds Argon2id (64 MiB, 3 iterations) → 32-byte encryption key
- Per-record random 12-byte nonce; ChaCha20Poly1305 encrypts the value field
- Metadata (TTL, read count) stored plaintext — required for efficient expiry scans
- `SIRR_MASTER_KEY` also serves as the bearer token (constant-time comparison)
- `sirr.salt` — 32 random bytes generated on first run, stored alongside `sirr.db`

**On-disk files:**

| File | Contents |
|---|---|
| `sirr.db` | redb database (encrypted values) |
| `sirr.salt` | Argon2id salt (not secret — just persistent) |

```bash
# Confirm no plaintext secrets in the database
xxd sirr.db | head -20
```

---

## Licensing

**Business Source License 1.1**

| | |
|---|---|
| ✅ Free for production | Up to **100 secrets per instance** |
| ✅ Unlimited non-production | Dev, staging, CI |
| ✅ Source available | Forks and modifications allowed |
| ✅ Converts to Apache 2.0 | On **February 20, 2028** |
| 💼 Commercial license | Required for >100 secrets |

**Get a license key** at [secretdrop.app/sirr](https://secretdrop.app/sirr) — free tier included.

Set it before starting the server:

```bash
SIRR_LICENSE_KEY=sirr_lic_... SIRR_MASTER_KEY=... ./sirr serve
```

---

## Installation Methods

### Docker

```bash
# With data persistence
docker run -d \
  --name sirr \
  -p 8080:8080 \
  -v ./data:/data \
  -e SIRR_DATA_DIR=/data \
  -e SIRR_MASTER_KEY="$(openssl rand -hex 32)" \
  ghcr.io/yourorg/sirr

# docker-compose
cat > docker-compose.yml << 'EOF'
services:
  sirr:
    image: ghcr.io/yourorg/sirr
    ports: ["8080:8080"]
    volumes: ["./sirr-data:/data"]
    environment:
      SIRR_DATA_DIR: /data
      SIRR_MASTER_KEY: ${SIRR_MASTER_KEY}
      SIRR_LICENSE_KEY: ${SIRR_LICENSE_KEY}
    restart: unless-stopped
EOF
```

### Homebrew

```bash
brew tap sirr/sirr https://github.com/yourorg/sirr
brew install sirr
```

### npm

```bash
# CLI
npm install -g @sirr/sdk
sirr --help

# MCP server
npm install -g @sirr/mcp
```

---

## Use Cases

### Development Teams

```bash
# Sync .env across team — expires in 24h so no stale credentials
sirr push .env --ttl 24h
# Team member pulls:
sirr pull .env
```

### CI/CD Pipelines

```bash
# GitHub Actions: one-time deploy token
- run: |
    sirr push DEPLOY_TOKEN="${{ secrets.DEPLOY_TOKEN }}" --reads 1
    sirr run -- ./deploy.sh
```

### AI-Assisted Development

```
# You: "Claude, use DATABASE_URL from sirr — it's set to burn after this session"
# Claude reads it once via MCP, does the work, credential is gone.
```

### Contractor Access

```bash
# Create time-limited access
sirr push STAGING_DB="postgres://..." --ttl 30d
# Access auto-revokes after 30 days — no manual cleanup needed
```

---

## The AI Workflow (Detailed)

```bash
# 1. You need Claude to analyze your production schema
sirr push PROD_DB="postgresql://user:pass@host/db" --reads 1 --ttl 1h

# 2. Tell Claude (with MCP configured)
# "Analyze the schema at PROD_DB and suggest indexes"

# 3. Claude uses MCP to fetch the credential — read counter → 1
# 4. Read limit reached → credential deleted automatically
# 5. Even if the conversation is stored/trained on, the credential is dead
```

---

## Roadmap

- [ ] Web UI for secret management
- [ ] Webhooks on expiration / burn
- [ ] Team namespaces (shared secrets within org)
- [ ] Kubernetes secrets sync operator
- [ ] Terraform provider
- [ ] Audit log (who accessed what, when)
- [ ] Secret versioning
- [ ] Browser extension
- [ ] LDAP/SSO integration

---

## Contributing

Contributions welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md).

By contributing, you agree to license your contributions under the same BSL 1.1 terms.

---

## Why "Sirr"?

*Sirr* (سر) means "secret" in Arabic. Short, memorable, and fits the theme of secrets that whisper and disappear.

---

**Made with ❤️ for developers who believe secrets should expire.**

*"The best secret is the one that destroys itself."*
# sirr
