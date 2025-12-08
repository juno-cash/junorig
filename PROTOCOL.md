# Junocash Stratum Protocol Specification

## Overview

Junocash uses a modified stratum protocol based on CryptoNote stratum but with
Zcash-style 140-byte headers and 32-byte nonces.

## Job Notification

### Request (from pool)
```json
{
  "jsonrpc": "2.0",
  "method": "job",
  "params": {
    "blob": "<280 hex chars = 140 bytes>",
    "job_id": "259",
    "target": "cb10c7bab88d0600",
    "height": 34347,
    "seed_hash": "2e9a8a769d27fbee0123456789abcdef0123456789abcdef0123456789abcdef",
    "algo": "rx/0"
  }
}
```

## Blob Structure (140 bytes)

| Offset | Size | Field | Endianness |
|--------|------|-------|------------|
| 0-3 | 4 | nVersion | Little-endian |
| 4-35 | 32 | hashPrevBlock | Internal (reversed from display) |
| 36-67 | 32 | hashMerkleRoot | Internal (reversed from display) |
| 68-99 | 32 | hashBlockCommitments | Internal (reversed from display) |
| 100-103 | 4 | nTime | Little-endian |
| 104-107 | 4 | nBits (compact target) | Little-endian |
| 108-139 | 32 | nNonce | 256-bit value |

## Share Submission

### Request (to pool)
```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "submit",
  "params": {
    "id": "worker_id",
    "job_id": "259",
    "nonce": "<64 hex chars = 32 bytes>",
    "result": "<64 hex chars = 32 bytes (hash)>"
  }
}
```

### Key Differences from CryptoNote

1. **Nonce Size**: 32 bytes instead of 4 bytes
2. **Blob Size**: 140 bytes instead of 76 bytes
3. **Nonce Position**: Bytes 108-139 instead of byte 39

## Target Handling

### Pool Target Format
- 8 bytes, little-endian hex string
- Example: `"cb10c7bab88d0600"` = difficulty ~10000

### Internal Comparison
1. Expand 8-byte target to 32-byte big-endian array
2. Place target value at bytes 0-7 (MSB position)
3. Compare hash (big-endian) against target using memcmp
4. Valid share: hash <= target

### Target Conversion Code
```cpp
// Pool sends: "cb10c7bab88d0600" (8 bytes LE)
// Convert to 32-byte BE for comparison:
uint8_t target32[32] = {0};
// Reverse 8-byte LE to BE and place at start
for (int i = 0; i < 8; i++) {
    target32[i] = target8[7 - i];
}
```

## RandomX Parameters

Junocash uses standard RandomX (rx/0) with identical parameters to Monero:
- Scratchpad: 2 MB
- Programs: 8
- Iterations: 2048
- Dataset: ~2 GB
- Seed changes every 2048 blocks

## Solo Mining (getblocktemplate)

### Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "getblocktemplate",
  "params": {}
}
```

Note: Unlike CryptoNote, Junocash does not require wallet_address in params.
The node constructs the coinbase transaction.

### Response Fields
```json
{
  "result": {
    "version": 4,
    "previousblockhash": "00000...",
    "curtime": 1702012800,
    "bits": "1d00ffff",
    "height": 34500,
    "randomxseedheight": 32768,
    "randomxseedhash": "abcd...",
    "defaultroots": {
      "merkleroot": "1234...",
      "blockcommitmentshash": "5678..."
    },
    "coinbasetxn": { "data": "..." },
    "transactions": [{ "data": "..." }]
  }
}
```

## Block Submission

### submitblock Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "submitblock",
  "params": ["<full block hex>"]
}
```

### Block Structure
```
[140 bytes header with winning nonce]
[coinbase transaction]
[other transactions in order]
```

## Nonce Increment Strategy

For pool mining, the miner typically:
1. Receives a job with a 140-byte blob (nonce at bytes 108-139 is usually zeroed)
2. Increments nonce as a 256-bit integer starting from 0
3. For each nonce value, computes RandomX hash of the full 140-byte blob
4. If hash <= target, submits the share with the winning 32-byte nonce

## Hash Computation

```cpp
// Input: 140-byte header blob (nonce already inserted at bytes 108-139)
// Output: 32-byte RandomX hash

randomx_calculate_hash(vm, blob, 140, hash);

// Compare hash against target (big-endian comparison)
if (memcmp(hash, target32, 32) <= 0) {
    // Valid share found
}
```
