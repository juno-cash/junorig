# rx/juno Solo Mining Protocol Specification

This document describes direct mining against a Juno daemon without a pool.

For pool mining using Stratum, see [PROTOCOL_STRATUM.md](PROTOCOL_STRATUM.md).
For a user guide on setting up solo mining, see [doc/SOLO_MINING.md](doc/SOLO_MINING.md).

## Overview

| Property | Value |
|----------|-------|
| Algorithm | `rx/juno` (RandomX variant) |
| Algorithm ID | `0x7215126a` |
| Nonce size | 32 bytes |
| Block header size | 140 bytes |
| Transport | HTTP/HTTPS |
| Encoding | JSON-RPC 1.0 |

## Connection

### Supported URI Schemes

| Scheme | Description |
|--------|-------------|
| `daemon+http://host:port` | Plain HTTP connection to daemon RPC |
| `daemon+https://host:port` | TLS encrypted connection |

### Authentication

HTTP Basic Authentication using base64-encoded `username:password`.

The `-u` and `-p` options specify RPC credentials, not wallet addresses. The mining reward address is configured on the daemon itself.

## RPC Protocol

Solo mining uses **JSON-RPC 1.0** (not 2.0 like pool mining).

## Block Template Request

**Direction:** Miner → Daemon

```json
{
  "jsonrpc": "1.0",
  "id": 1,
  "method": "getblocktemplate",
  "params": [{
    "capabilities": ["coinbasetxn", "workid", "coinbase/append"]
  }]
}
```

## Block Template Response

**Direction:** Daemon → Miner

```json
{
  "result": {
    "height": 123456,
    "version": 4,
    "previousblockhash": "0000...abcd",
    "defaultroots": {
      "merkleroot": "1234...5678",
      "blockcommitmentshash": "abcd...ef01"
    },
    "curtime": 1234567890,
    "bits": "1f07ffff",
    "randomxseedhash": "aaaa...bbbb",
    "coinbasetxn": {
      "data": "..."
    },
    "transactions": [
      {"data": "...", "hash": "..."}
    ]
  }
}
```

| Field | Description |
|-------|-------------|
| `height` | Block height being mined |
| `version` | Block version (typically 4) |
| `previousblockhash` | Previous block hash (display order) |
| `defaultroots.merkleroot` | Merkle tree root (display order) |
| `defaultroots.blockcommitmentshash` | Block commitments hash (display order) |
| `curtime` | Current timestamp (Unix epoch) |
| `bits` | Compact difficulty target |
| `randomxseedhash` | RandomX seed hash for current epoch |
| `coinbasetxn.data` | Coinbase transaction hex |
| `transactions` | Array of transaction objects |

**Important:** Hash fields (`previousblockhash`, `merkleroot`, `blockcommitmentshash`) are returned in **display order** (big-endian). They must be byte-reversed to **internal order** (little-endian) when constructing the block header for hashing.

## Block Header Construction

The mining blob is constructed from the template response:

```
Offset  Size  Field                    Source
------  ----  -----                    ------
0       4     Version (LE)             version field
4       32    Previous hash            previousblockhash (reversed)
36      32    Merkle root              merkleroot (reversed)
68      32    Block commitments        blockcommitmentshash (reversed)
100     4     Timestamp (LE)           curtime field
104     4     Bits (LE)                bits field (parsed from hex)
108     32    Nonce                    Miner fills this
------
140 bytes total
```

## Block Submission

**Direction:** Miner → Daemon

```json
{
  "jsonrpc": "1.0",
  "id": 2,
  "method": "submitblock",
  "params": ["<serialized_block_hex>"]
}
```

## Serialized Block Format

The complete block for submission:

```
Offset  Size     Field
------  -----    -----
0       140      Block header (as constructed above, with solution nonce)
140     1        Solution length (varint = 32)
141     32       Solution hash (RandomX hash result)
173     1-9      Transaction count (CompactSize varint)
var     var      Coinbase transaction (from coinbasetxn.data)
var     var      Other transactions (from transactions[].data)
```

### Detailed Breakdown

1. **Block Header (140 bytes)**: The mining header with the winning nonce
2. **Solution Length**: CompactSize varint, always 32 for RandomX
3. **Solution Hash**: The 32-byte RandomX hash that met the target
4. **Transaction Count**: Number of transactions (1 + mempool transactions)
5. **Coinbase Transaction**: The coinbase from `coinbasetxn.data`
6. **Other Transactions**: Each transaction from `transactions[].data`

## Submit Response

**Direction:** Daemon → Miner

```json
{
  "result": null,
  "error": null
}
```

A `null` result indicates success. Accepted responses also include:
- `"duplicate"` - Block already received
- `"inconclusive"` - May or may not be accepted
- `"duplicate-inconclusive"` - Combination of above

Any other string value indicates rejection.

## ZMQ Notifications (Optional)

For real-time block notifications, connect to the daemon's ZMQ port:

1. Subscribe to `hashblock` topic
2. On notification, call `getblocktemplate` to fetch new work

This avoids polling latency when new blocks are found.

### Configuration

Daemon side:
```bash
junod -zmqpubhashblock=tcp://0.0.0.0:28332
```

Miner side:
```bash
./xmrig -o 127.0.0.1:8232 --daemon --daemon-zmq-port=28332 -a rx/juno
```

## Difficulty from Bits

The `bits` field uses Bitcoin's compact format:
- Top byte: exponent
- Lower 3 bytes: mantissa
- Target = mantissa × 2^(8 × (exponent - 3))

Example: `bits = 0x1f07ffff`
- Exponent: `0x1f` = 31
- Mantissa: `0x07ffff`
- Target bytes 28-30 = `0xff, 0xff, 0x07` (little-endian)

## Polling Interval

When ZMQ is not available, the miner polls for new block templates:

| Parameter | Default |
|-----------|---------|
| Poll interval | 1000 ms |
| Job timeout | 15000 ms |

Configure with `--daemon-poll-interval` and `--daemon-job-timeout`.

## Implementation Reference

| File | Description |
|------|-------------|
| `src/base/net/stratum/JunoRpcClient.cpp` | Solo mining RPC client |
| `src/base/net/stratum/JunoRpcClient.h` | Client interface |
| `src/base/net/stratum/Job.cpp` | Job/header construction |

## Example Session

```bash
# Start Juno daemon with RPC and ZMQ
junod -server -rpcuser=user -rpcpassword=pass -mineraddress=j1... \
      -zmqpubhashblock=tcp://127.0.0.1:28332

# Start miner
./xmrig -o 127.0.0.1:8232 --daemon --daemon-zmq-port=28332 \
        -u user -p pass -a rx/juno
```

```
# Miner polls for block template
→ POST / {"jsonrpc":"1.0","id":1,"method":"getblocktemplate","params":[{"capabilities":["coinbasetxn","workid","coinbase/append"]}]}

# Daemon returns template
← {"result":{"height":12345,"version":4,"previousblockhash":"...","defaultroots":{...},...}}

# Miner finds valid nonce and submits block
→ POST / {"jsonrpc":"1.0","id":2,"method":"submitblock","params":["<block_hex>"]}

# Daemon accepts block
← {"result":null,"error":null}
```
