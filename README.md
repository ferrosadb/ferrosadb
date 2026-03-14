<p align="center">
  <img src="https://ferrosadb.com/logo.svg" alt="Ferrosa" width="280">
</p>

<h3 align="center">Cassandra Reimagined in Rust, Backed by S3</h3>

<p align="center">
  <a href="https://ferrosadb.com">Website</a> &bull;
  <a href="https://github.com/ferrosadb/ferrosadb/releases">Downloads</a> &bull;
  <a href="https://ferrosadb.com/getting-started.html">Getting Started</a> &bull;
  <a href="https://ferrosadb.com/cql-compatibility.html">CQL Compatibility</a>
</p>

---

Ferrosa is a **drop-in Cassandra replacement** built from scratch in Rust with S3-backed storage. Same CQL. Same drivers. Zero GC pauses, ephemeral nodes, native graph queries, and 10x lower storage costs.

## Why Ferrosa?

| Problem with Cassandra | Ferrosa's Answer |
|------------------------|------------------|
| GC pauses kill your P99 | Rust — no garbage collector, no pauses, predictable latency |
| Hours to replace a node | Ephemeral nodes — boot, pull SSTables from S3, serve traffic |
| Storage costs scale linearly | S3-backed — local disk is a cache, S3 is the source of truth |
| Ops expertise required | One binary, systemd service, pair-mode HA out of the box |

## Features

### Drop-In CQL Compatibility

Use your existing Cassandra drivers, cqlsh, and application code. Ferrosa speaks CQL v5 native protocol with full DDL, DML, prepared statements, and frame compression (LZ4, Snappy).

### S3-Backed Storage

Write-behind async S3 — local ephemeral disk as cache, S3 as durable store. Nodes are stateless and replaceable. Boot a new node, it pulls data from S3 and starts serving. Storage costs drop to S3 pricing.

### Pluggable Secondary Indexes

Eight index types behind a common pluggable interface:

| Index Type | Use Case |
|-----------|----------|
| **B-tree** | Ordered range queries and sorted scans |
| **Hash** | O(1) equality point lookups |
| **Composite** | Multi-column prefix-based lookups |
| **Phonetic** | Fuzzy name matching (Soundex, Metaphone, Double Metaphone, Caverphone) |
| **Filtered** | Partial indexes over a row subset (e.g., only active users) |
| **Vector (HNSW)** | Approximate nearest neighbor — best query performance |
| **Vector (IVFFlat)** | Approximate nearest neighbor — faster builds, k-means clustering |

Indexes are **storage-attached** and built asynchronously after SSTable flush — zero impact on the write path. Per-index staleness tracking with operational metrics via `system_views.secondary_indexes`.

### Vector Search

Built-in approximate nearest neighbor search with three distance metrics:

```sql
-- Create a vector index
CREATE INDEX idx_embed ON documents (embedding) USING 'vector'
    WITH OPTIONS = {'method': 'hnsw', 'metric': 'cosine', 'dimensions': '768'};

-- Query nearest neighbors
SELECT * FROM documents ORDER BY embedding ANN OF [0.1, 0.2, ...] LIMIT 10;
```

Supports L2 (Euclidean), cosine similarity, and inner product. Up to 4,096 dimensions (f32) or 8,192 (f16).

### Phonetic Search

Find names that sound alike, regardless of spelling:

```sql
CREATE INDEX idx_name ON users (last_name) USING 'phonetic'
    WITH OPTIONS = {'algorithm': 'double_metaphone'};

SELECT * FROM users WHERE last_name SOUNDS LIKE 'Smith';
-- Finds: Smith, Smythe, Smithe, ...
```

### High Availability

Pair-mode replication with automatic failover:

- **Write forwarding** — secondary forwards writes to primary
- **DDL replication** — schema changes propagate automatically
- **Failover** — promote secondary to primary with one API call
- **Catch-up** — rejoining nodes receive schema + data automatically
- **Switchover** — swap primary/secondary roles with zero downtime

### Native Graph Queries

Query relationships in your CQL data with a Cypher subset — no separate graph database needed:

```cypher
MATCH (u:User)-[:FOLLOWS]->(f:User)
WHERE u.name = 'Alice'
RETURN f.name, f.email
```

### Lock-Free Architecture

Built on `ArcSwap` for lock-free reads, sharded memtables, and channel-based task scheduling. No mutex contention on the hot path.

### Built-In Observability

- Prometheus metrics endpoint
- Active query tracking (`system_views.active_queries`)
- Connection monitoring (`system_views.connections`)
- Index staleness metrics (`system_views.secondary_indexes`)
- TUI dashboard via `ferrosa-ctl monitor`

## Quick Start

### Install (Debian/Ubuntu)

```bash
wget https://github.com/ferrosadb/ferrosadb/releases/latest/download/ferrosa_0.1.0_amd64.deb
sudo dpkg -i ferrosa_0.1.0_amd64.deb
sudo systemctl start ferrosa
```

### Connect

```bash
cqlsh localhost 9042
```

```sql
CREATE KEYSPACE myapp WITH replication = {
    'class': 'SimpleStrategy', 'replication_factor': '1'
};

CREATE TABLE myapp.users (
    id uuid PRIMARY KEY,
    name text,
    email text
);

CREATE INDEX idx_email ON myapp.users (email) USING 'btree';

INSERT INTO myapp.users (id, name, email)
VALUES (uuid(), 'Alice', 'alice@example.com');

SELECT * FROM myapp.users WHERE id = <uuid>;
```

### Docker (Two-Node Pair Mode)

```bash
docker compose up -d
# Node 1: cqlsh localhost 9042
# Node 2: cqlsh localhost 9043
# Web console: http://localhost:9090
```

## Architecture

```
┌─────────────────────────────────────────────┐
│                 ferrosa                      │
│         CQL :9042 │ HTTP :9090              │
├─────────────┬───────────┬───────────────────┤
│ ferrosa-cql │ ferrosa-  │ ferrosa-cluster   │
│  Parser     │  graph    │  Pair mode / Raft │
│  Router     │  Cypher   │  DDL coordination │
│  Protocol   │  Planner  │  Write forwarding │
├─────────────┼───────────┴───────────────────┤
│ ferrosa-    │ ferrosa-index                  │
│  schema     │  B-tree, Hash, Composite       │
│  Registry   │  Phonetic, Filtered            │
│  Auth/RBAC  │  Vector HNSW, IVFFlat          │
├─────────────┼────────────────────────────────┤
│ ferrosa-storage                              │
│  Memtable │ Commit Log │ Compaction │ S3     │
├──────────────────────────────────────────────┤
│ ferrosa-sstable          │ ferrosa-net       │
│  BTI trie format         │  Internode RPC    │
│  Bloom filter            │  Priority lanes   │
├──────────────────────────┴───────────────────┤
│ ferrosa-common                               │
│  Token │ CellValue │ Murmur3 │ Types        │
└──────────────────────────────────────────────┘
```

## Comparison

| Feature | Ferrosa | Cassandra | ScyllaDB | DynamoDB |
|---------|---------|-----------|----------|----------|
| Language | Rust | Java | C++ | Managed |
| GC Pauses | None | Yes | None | N/A |
| Storage Backend | S3 | Local disk | Local disk | Managed |
| CQL Compatible | Yes | Yes | Yes | No |
| Vector Search | HNSW + IVFFlat | SAI (limited) | No | No |
| Secondary Indexes | 6 types | SAI / 2i | SI | GSI/LSI |
| Graph Queries | Built-in | No | No | No |
| Phonetic Search | Built-in | No | No | No |
| Node Recovery | Seconds (S3) | Hours | Hours | N/A |
| Minimum Nodes | 1 | 3 | 3 | N/A |

## Package Contents

The `.deb` package includes:

| Path | Description |
|------|-------------|
| `/usr/bin/ferrosa` | Database server |
| `/usr/bin/ferrosa-ctl` | CLI admin tool (query, describe, monitor, metrics) |
| `/lib/systemd/system/ferrosa.service` | Systemd service unit |
| `/usr/share/ferrosa/docs/` | Documentation |
| `/usr/share/ferrosa/tests/` | Smoke tests and Docker Compose config |
| `/etc/ferrosa/` | Configuration directory |
| `/var/lib/ferrosa/` | Data directory |

## License

Apache-2.0
