# rx/juno Stratum Protocol Specification

This document describes the Stratum protocol variant used by Junorig for pool mining Juno Cash (rx/juno algorithm).

For solo mining directly against a Juno daemon, see [PROTOCOL_SOLO.md](PROTOCOL_SOLO.md).

## Overview

Junorig implements a hybrid Stratum protocol combining:
- **XMRig's Stratum** (JSON-RPC 2.0) for authentication and share submission
- **Zcash-style mining notifications** for job distribution

| Property | Value |
|----------|-------|
| Algorithm | `rx/juno` (RandomX variant) |
| Algorithm ID | `0x7215126a` |
| Nonce size | 32 bytes |
| Block header size | 140 bytes |
| Transport | TCP (plain or TLS) |
| Encoding | JSON-RPC 2.0 |

## Connection

### Supported URI Schemes

| Scheme | Description |
|--------|-------------|
| `stratum://host:port` | Plain TCP connection |
| `stratum+tcp://host:port` | Plain TCP connection (explicit) |
| `stratum+ssl://host:port` | TLS encrypted connection |
| `stratum+tls://host:port` | TLS encrypted connection |

### Timeouts

| Parameter | Default |
|-----------|---------|
| Connect timeout | 20 seconds |
| Response timeout | 20 seconds |
| Retry pause | 5 seconds |
| Retry count | 5 attempts |

## Authentication

### Login Request

The client initiates a session by sending a `login` request.

**Direction:** Client → Pool

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "login",
  "params": {
    "login": "j1...",
    "pass": "x",
    "agent": "junorig/6.x.x",
    "rigid": "worker-name"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `login` | string | Yes | Juno unified address (starts with `j1`) |
| `pass` | string | Yes | Password (typically `x`, often ignored) |
| `agent` | string | Yes | Miner user-agent string |
| `rigid` | string | No | Rig/worker identifier |

### Login Response

**Direction:** Pool → Client

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "id": "client-session-id",
    "job": {
      "job_id": "abc123",
      "blob": "...",
      "target": "...",
      "height": 123456,
      "seed_hash": "...",
      "algo": "rx/juno"
    },
    "extensions": ["algo", "nicehash", "keepalive", "tls"]
  },
  "error": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Client session ID (used in subsequent requests) |
| `job` | object | Initial mining job |
| `extensions` | array | Supported protocol extensions |

### Protocol Extensions

| Extension | Description |
|-----------|-------------|
| `algo` | Pool can specify algorithm per-job |
| `nicehash` | NiceHash difficulty adjustment compatibility |
| `keepalive` | Connection keep-alive support |
| `tls` | TLS/SSL connection support |
| `connect` | Pool can request client reconnection |

## Job Distribution

### Standard Job Notification

Used for initial job in login response and some pool implementations.

**Direction:** Pool → Client

```json
{
  "jsonrpc": "2.0",
  "method": "job",
  "params": {
    "job_id": "abc123",
    "blob": "...",
    "target": "...",
    "height": 123456,
    "seed_hash": "...",
    "algo": "rx/juno"
  }
}
```

### Zcash-Style Job Notification (Primary)

The primary job notification method for rx/juno mining.

**Direction:** Pool → Client

```json
{
  "id": null,
  "method": "mining.notify",
  "params": [
    "job_id",
    "version",
    "prevhash",
    "merkleroot",
    "blockcommitments",
    "time",
    "bits",
    false,
    "seed_hash"
  ]
}
```

| Index | Field | Size | Description |
|-------|-------|------|-------------|
| 0 | `job_id` | string | Unique job identifier |
| 1 | `version` | 8 hex | Block version (4 bytes) |
| 2 | `prevhash` | 64 hex | Previous block hash (32 bytes) |
| 3 | `merkleroot` | 64 hex | Merkle root (32 bytes) |
| 4 | `blockcommitments` | 64 hex | Block commitments (32 bytes) |
| 5 | `time` | 8 hex | Block timestamp (4 bytes) |
| 6 | `bits` | 8 hex | Difficulty bits (4 bytes) |
| 7 | `clean_jobs` | boolean | If true, discard all previous jobs |
| 8 | `seed_hash` | 64 hex | RandomX seed hash (32 bytes) |

## Block Header Structure

The mining blob is constructed from the `mining.notify` parameters:

```
Offset  Size  Field
------  ----  -----
0       4     Version (little-endian)
4       32    Previous block hash
36      32    Merkle root
68      32    Block commitments
100     4     Timestamp (little-endian)
104     4     Difficulty bits (little-endian)
108     32    Nonce (miner fills this)
------  ----
Total:  140 bytes
```

### Nonce Details

| Property | Value |
|----------|-------|
| Size | 32 bytes (256 bits) |
| Offset | Byte 108 |
| Format | Little-endian |
| Hex length | 64 characters |

This is significantly larger than standard RandomX (4-byte nonce), allowing for much larger search space without extra-nonce.

## Share Submission

### Submit Request

**Direction:** Client → Pool

```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "method": "submit",
  "params": {
    "id": "client-session-id",
    "job_id": "abc123",
    "nonce": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "result": "..."
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Client session ID from login response |
| `job_id` | string | Job ID from the job notification |
| `nonce` | string | 64 hex characters (32 bytes) |
| `result` | string | Resulting block hash (64 hex characters) |

### Submit Response

**Direction:** Pool → Client

```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "result": {
    "status": "OK"
  },
  "error": null
}
```

On rejection:

```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "result": null,
  "error": {
    "code": -1,
    "message": "Low difficulty share"
  }
}
```

## Keep-Alive

### Keep-Alive Request

**Direction:** Client → Pool

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "keepalived",
  "params": {
    "id": "client-session-id"
  }
}
```

### Keep-Alive Response

**Direction:** Pool → Client

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "result": {
    "status": "KEEPALIVED"
  },
  "error": null
}
```

## Pool Commands

### Client Reconnect

Pool can request the client to reconnect to a different server.

**Direction:** Pool → Client

```json
{
  "id": null,
  "method": "client.reconnect",
  "params": ["newhost.example.com", 3333]
}
```

## Error Handling

### Critical Errors (Trigger Reconnection)

These error messages in pool responses trigger automatic reconnection:

- `"Unauthenticated"`
- `"your IP is banned"`
- `"IP Address currently banned"`
- `"Invalid job id"`

### Common Error Codes

| Code | Description |
|------|-------------|
| -1 | Generic error |
| 21 | Job not found |
| 23 | Low difficulty share |
| 24 | Duplicate share |
| 25 | Invalid share |

## Example Session

```
# Client connects via TCP

# 1. Client sends login
→ {"id":1,"jsonrpc":"2.0","method":"login","params":{"login":"j1...","pass":"x","agent":"junorig/6.22.0"}}

# 2. Pool responds with session ID and initial job
← {"id":1,"jsonrpc":"2.0","result":{"id":"abc123","job":{...},"extensions":["algo","keepalive"]}}

# 3. Pool sends new job notification
← {"id":null,"method":"mining.notify","params":["job1","04000000","...","...","...","...","...",false,"..."]}

# 4. Client finds valid share and submits
→ {"id":2,"jsonrpc":"2.0","method":"submit","params":{"id":"abc123","job_id":"job1","nonce":"...","result":"..."}}

# 5. Pool accepts share
← {"id":2,"jsonrpc":"2.0","result":{"status":"OK"}}

# 6. Client sends keep-alive periodically
→ {"id":3,"jsonrpc":"2.0","method":"keepalived","params":{"id":"abc123"}}
← {"id":3,"jsonrpc":"2.0","result":{"status":"KEEPALIVED"}}
```

## Implementation References

| File | Description |
|------|-------------|
| `src/base/net/stratum/Client.cpp` | Core Stratum client implementation |
| `src/base/net/stratum/Job.cpp` | Job parsing including `setZcashJob()` |
| `src/base/net/stratum/Job.h` | Job structure and nonce handling |
| `src/base/net/stratum/BaseClient.cpp` | Base client with response handling |
| `src/base/crypto/Algorithm.h` | Algorithm definitions (RX_JUNO) |

## Differences from Standard Stratum

| Feature | Standard XMRig Stratum | rx/juno Variant |
|---------|------------------------|-----------------|
| Nonce size | 4 bytes | 32 bytes |
| Job notification | `job` method | `mining.notify` (Zcash-style) |
| Block format | CryptoNote | Zcash header (140 bytes) |
| Address format | Various | Juno unified (`j1...`) |
| Nonce offset | Byte 39 | Byte 108 |
