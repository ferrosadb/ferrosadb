# Ferrosa

CQL-compatible distributed database with S3-backed storage, built in Rust.

**[ferrosadb.com](https://ferrosadb.com)**

## Installation (Debian/Ubuntu)

```bash
# Download the latest release
wget https://github.com/ferrosadb/ferrosadb/releases/latest/download/ferrosa_0.1.0_amd64.deb

# Install
sudo dpkg -i ferrosa_0.1.0_amd64.deb

# Start
sudo systemctl start ferrosa

# Connect
cqlsh localhost 9042
```

## Features

- CQL v5 native protocol — drop-in client compatibility
- S3-backed storage — local ephemeral disk as cache, S3 as durable store
- Pluggable secondary indexes — B-tree, hash, composite, phonetic, filtered, vector (HNSW + IVFFlat)
- Vector search — approximate nearest neighbor with L2, cosine, inner product
- Pair-mode high availability — automatic failover, promotion, switchover
- Built-in graph queries — Cypher subset over CQL data
- Async index builds — zero write-path impact with staleness tracking

## License

Apache-2.0
