# mina-archive-testbed

A Podman/Docker Compose testbed for deploying a Mina Archive node combo locally. The default configuration targets the **Mina Devnet** network on **macOS with Apple Silicon**.

## Components

1. **PostgreSQL Database** - Stores blockchain data
2. **Dump Loader** _(recommended)_ - Pre-loads the archive database from a GCS dump
3. **Mina Archive Node** - Processes and stores block data
4. **Mina Plain Node** - Connects to the network and feeds data to the archive

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Podman Host (macOS)                             │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │   Plain     │───▶│   Archive   │───▶│    PostgreSQL            │  │
│  │   Node      │    │   Node      │    │                          │  │
│  │ Port: 3085  │    │ Port: 3086  │    │  Port: 5432              │  │
│  │ (GraphQL)   │    │ (Archive)   │    │  (internal)              │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘  │
│        │                                   ▲               │        │
│        │                              ┌────┘               │        │
│        │                              │                    │        │
│        │                    ┌─────────────────┐            │        │
│        │                    │  Dump Loader     │            │        │
│        │                    │  (optional,      │            │        │
│        │                    │   one-shot)      │            │        │
│        │                    └─────────────────┘            │        │
│        ▼                                                   ▼        │
│  ┌─────────────┐                        ┌─────────────────────────┐ │
│  │   Volume    │                        │      Volume              │ │
│  │  ./data/    │                        │   ./data/postgres        │ │
│  │    mina     │                        │                          │ │
│  └─────────────┘                        └─────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

Host Port Mappings:
  - 5433 → PostgreSQL (5432 internal)
  - 3086 → Archive Node
  - 3085 → GraphQL API
  - 10003 → P2P
  - 8081 → Daemon Metrics
  - 10002 → Archive Metrics
```

## Resource Requirements

| Component | CPU (Request) | CPU (Limit) | Memory (Request) | Memory (Limit) | Storage |
|-----------|---------------|-------------|------------------|----------------|---------|
| PostgreSQL | 2 | 3 | 3Gi | 10Gi | 12Gi |
| Archive Node | 4 | 6 | 4Gi | 8Gi | - |
| Plain Node | 6 | 8 | 8Gi | 12Gi | (Optional) 15Gi |
| **Total** | **12** | **17** | **15Gi** | **30Gi** | **27Gi** |

For production or long-running environments, increase PostgreSQL memory to 16Gi and storage to 50Gi+.

## Quick Start

### Prerequisites

1. **Podman Desktop** or **Docker Desktop** installed
2. Container runtime running (`podman machine start` or equivalent)
3. **podman-compose** or **docker-compose** installed

### Setup

```bash
# Clone the repo
git clone https://github.com/o1-labs/mina-archive-testbed.git
cd mina-archive-testbed

# Create data directories
mkdir -p data/postgres data/mina shared

# Download the archive database schema
curl -o init-schema.sql https://raw.githubusercontent.com/MinaProtocol/mina/refs/heads/compatible/src/app/archive/create_schema.sql

# Start all services
podman-compose up -d

# View logs
podman-compose logs -f
```

### Initializing from an Archive Dump (Recommended)

The archive only records blocks added to the transition frontier **after** the daemon finishes bootstrapping. Without a dump, the archive starts with only the genesis block and misses all historical data. **Starting from a dump is strongly recommended** to ensure the archive has complete block history.

The compose file defaults to loading the latest devnet dump. To use a different dump, override `ARCHIVE_DUMP_URL` before starting:

```bash
# Devnet
export ARCHIVE_DUMP_URL=https://storage.googleapis.com/mina-archive-dumps/devnet-archive-dump-2025-09-11_1104.sql.tar.gz

# Then start services
podman-compose up -d
```

Available dumps: `https://storage.googleapis.com/mina-archive-dumps/`

The `dump-loader` service runs once at startup, loads the SQL dump into PostgreSQL, and exits. Data persists across restarts — subsequent launches detect the existing data and skip the import.

## Configuration

### Environment Overrides (.env)

Create a `.env` file to override defaults:

```bash
# Platform (for x86_64 hosts, change to linux/amd64)
PLATFORM=linux/arm64

# Container Images (override for different networks/versions)
DAEMON_IMAGE=gcr.io/o1labs-192920/mina-daemon:3.3.1-7b34378-noble-devnet-arm64
ARCHIVE_IMAGE=europe-west3-docker.pkg.dev/o1labs-192920/euro-docker-repo/mina-archive:3.3.0-compatible-cc77620-noble-devnet-arm64

# Network Configuration
# Devnet (default):
PEER_LIST_URL=https://bootnodes.minaprotocol.com/networks/devnet.txt

# Mainnet:
# PEER_LIST_URL=https://bootnodes.minaprotocol.com/networks/mainnet.txt

# Database (use secrets management in production!)
POSTGRES_PASSWORD=mina_dev_password

# Archive Database Initialization (recommended — enabled by default)
# Pre-loads the archive DB from a dump so it has complete block history.
# Dumps available at: https://storage.googleapis.com/mina-archive-dumps/
ARCHIVE_DUMP_URL=https://storage.googleapis.com/mina-archive-dumps/devnet-archive-dump-2026-02-13_0900.sql.tar.gz
```

### Default Images

| Component | Default Image |
|-----------|---------------|
| PostgreSQL | `postgres:15-alpine` |
| Archive | `europe-west3-docker.pkg.dev/o1labs-192920/euro-docker-repo/mina-archive:3.3.0-compatible-cc77620-noble-devnet-arm64` |
| Daemon | `gcr.io/o1labs-192920/mina-daemon:3.3.1-7b34378-noble-devnet-arm64` |

### Network-Specific Configurations

**Mainnet:**

```bash
PEER_LIST_URL=https://bootnodes.minaprotocol.com/networks/mainnet.txt
DAEMON_IMAGE=gcr.io/o1labs-192920/mina-daemon:3.3.1-7b34378-noble-mainnet-arm64
ARCHIVE_IMAGE=europe-west3-docker.pkg.dev/o1labs-192920/euro-docker-repo/mina-archive:3.3.0-compatible-cc77620-noble-mainnet-arm64
```

**x86_64 / Intel Mac:**

```bash
PLATFORM=linux/amd64
DAEMON_IMAGE=gcr.io/o1labs-192920/mina-daemon:3.3.1-7b34378-noble-devnet
ARCHIVE_IMAGE=europe-west3-docker.pkg.dev/o1labs-192920/euro-docker-repo/mina-archive:3.3.0-compatible-cc77620-noble-devnet
```

## Verification

### Check Node Sync Status

```bash
curl -s http://localhost:3085/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query": "{syncStatus}"}' | jq
```

Expected sync statuses:
- `BOOTSTRAP` - Initial sync, downloading blocks
- `CATCHUP` - Catching up to the network
- `SYNCED` - Fully synchronized

### Check Archive Database

```bash
# Connect to PostgreSQL (note: host port 5433)
psql -h localhost -p 5433 -U mina -d archive

# Check block count
SELECT COUNT(*) FROM blocks;

# Check latest block
SELECT state_hash, height, timestamp FROM blocks ORDER BY height DESC LIMIT 5;
```

### Monitor Resources

```bash
podman stats
```

## Operations

### Stop Services

```bash
# Stop services (preserves data)
podman-compose down

# Stop and remove volumes (clean slate)
podman-compose down -v
rm -rf data/
```

### Container Shell Access

```bash
podman-compose exec mina-daemon mina client status
```

### Archive Metrics

```bash
curl http://localhost:10002/metrics
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Archive can't connect to PostgreSQL | Ensure postgres-uri uses internal port `5432`, not host port `5433`. Check `podman-compose logs postgres`. |
| Plain node not syncing | Verify peer list URL is accessible: `curl -I $PEER_LIST_URL`. Check image tag matches network. |
| Out of memory | Increase memory limits or allocate more memory to Podman machine (`podman machine set --memory 32768`). |
| Slow sync | Increase CPU allocation. Use SSD storage for `./data/` volumes. |
| "Genesis state not found" | Ensure image tag matches target network (e.g., `-devnet` suffix for devnet). |
| Platform mismatch | On Intel Macs, change `PLATFORM=linux/amd64` in `.env`. |
| Archive only has 1 block (genesis) | The daemon must fully sync before archiving new blocks. Ensure `ARCHIVE_DUMP_URL` is set (default for devnet) to pre-load historical data, or wait for `syncStatus: SYNCED`. |

### Podman Machine Resources

If containers are being OOM-killed:

```bash
podman machine stop
podman machine set --cpus 10 --memory 32768
podman machine start
```

## References

- [Mina Protocol Documentation](https://docs.minaprotocol.com/)
- [Mina Archive Node Guide](https://docs.minaprotocol.com/node-operators/archive-node)
- [Mina Protocol GitHub](https://github.com/MinaProtocol/mina)
- [Podman Desktop](https://podman-desktop.io/)

## License

See [LICENSE](LICENSE).
