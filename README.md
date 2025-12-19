# Junorig

Juno Cash miner based on XMRig with support for the `rx/juno` algorithm.

## Quick Start


## Download
Pre-built binaries are available from the [Releases](https://github.com/juno-cash/juno-xmrig/releases) page:
- **Linux x64**: `xmrig-vX.X.X-linux-x64.zip`
- **macOS x64**: `xmrig-vX.X.X-macos-x64.zip` (requires macOS 14+ Sonoma)
- **macOS ARM64** (Apple Silicon): `xmrig-vX.X.X-macos-arm64.zip` (requires macOS 14+ Sonoma)

### Mine

```bash
./xmrig -o pool.example.com:3333 -u j1YourAddress... -a rx/juno
```

Replace `j1YourAddress...` with your Juno Cash **unified address** (starts with `j1...`).

## Options

| Option | Description |
|--------|-------------|
| `-o` | Pool address (pool.examplel.com:3333) |
| `-u` | Your Juno Cash wallet address j1address.... |
| `-a` | Algorithm (rx/juno) |
| `-t` | Number of CPU threads (default: all) |

## Example

```bash
# Use 4 threads
./xmrig -o pool.example.com:3333 -u j1YourAddress... -a rx/juno -t 4

# Use all threads
./xmrig -o pool.example.com:3333 -u j1YourAddress... -a rx/juno
```

### Build from source

## Building
See [BUILDING.md](BUILDING.md) for detailed build instructions for all platforms.

### Quick Build (Linux/macOS)
```bash
git clone https://github.com/juno-cash/juno-xmrig
cd juno-xmrig
mkdir build && cd build
cmake ..
make -j$(nproc)
```

## Documentation
- [BUILDING.md](BUILDING.md) - Detailed build instructions
- [PROTOCOL.md](PROTOCOL.md) - Junocash stratum protocol specification
- [CHANGELOG.md](CHANGELOG.md) - Version history and changes
- [SOLO_MINING.md](docs/SOLO_MINING.md) - Solo mining guide

## Mining Backends
- **CPU** (x86/x64/ARMv7/ARMv8)
- **OpenCL** for AMD GPUs
- **CUDA** for NVIDIA GPUs via external [CUDA plugin](https://github.com/xmrig/xmrig-cuda)

## Based On
- XMRig v6.24.0 (https://github.com/xmrig/xmrig)
- Licensed under GPLv3

## XMRig Original Developers
* **[xmrig](https://github.com/xmrig)**
* **[sech1](https://github.com/SChernykh)**

## License
This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.
