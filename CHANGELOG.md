# Changelog

All notable changes to this project will be documented in this file.

For the original XMRig changelog, see [doc/CHANGELOG_XMRIG.md](doc/CHANGELOG_XMRIG.md).

## [6.24.0-juno] - 2024-12-07

### Added
- **Junocash Protocol Support**: Full implementation of 280-byte stratum protocol
- **32-byte Nonce**: Support for full 256-bit nonces (vs 4-byte standard)
- **rx/juno Algorithm**: Registered as alias for rx/0 with Junocash coin
- **JunocashTemplate**: Parser for Junocash getblocktemplate RPC responses
- **Solo Mining**: Full getblocktemplate/submitblock support for Junocash nodes
- **Pool Mining**: Compatible with pool.junohash.com and similar pools

### Modified Files
- `CMakeLists.txt` - Added SUPPORT_JUNOCASH preprocessor define
- `src/base/crypto/Algorithm.h` - Added RX_JUNO algorithm ID
- `src/base/crypto/Algorithm.cpp` - Registered rx/juno name and aliases
- `src/base/crypto/Coin.h` - Added JUNO coin enum
- `src/base/crypto/Coin.cpp` - Added Junocash coin properties
- `src/base/net/stratum/Job.h` - Added junocash header/target fields
- `src/base/net/stratum/Job.cpp` - Added junocash field copying
- `src/base/net/stratum/Client.cpp` - 280-byte job detection, 32-byte nonce submission
- `src/base/net/stratum/DaemonClient.h` - JunocashTemplate include
- `src/base/net/stratum/DaemonClient.cpp` - Junocash RPC branching
- `src/backend/cpu/CpuWorker.cpp` - Junocash-specific mining loop
- `src/net/JobResult.h` - Added 32-byte nonce field

### New Files
- `src/junocash/JunocashTemplate.h` - Template struct definition
- `src/junocash/JunocashTemplate.cpp` - RPC parsing and header construction

### Technical Details
- Header size: 140 bytes (108 base + 32 nonce)
- Nonce offset: bytes 108-139
- Target comparison: Big-endian memcmp
- Algorithm: RandomX (rx/0 compatible)
- Seed hash: 32 bytes from randomxseedhash field

### Base Version
- XMRig 6.24.0 (https://github.com/xmrig/xmrig/releases/tag/v6.24.0)
