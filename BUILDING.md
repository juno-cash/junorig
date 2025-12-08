# Building Juno Protocol XMRig

## Prerequisites

### Ubuntu/Debian
```bash
sudo apt-get install git build-essential cmake libuv1-dev libssl-dev libhwloc-dev
```

### Fedora/RHEL
```bash
sudo dnf install git cmake gcc gcc-c++ libuv-devel openssl-devel hwloc-devel make
```

### macOS
```bash
brew install cmake libuv openssl hwloc
```

### Windows
- Visual Studio 2019+ or MinGW-w64
- CMake 3.10+
- Pre-built dependencies from https://github.com/xmrig/xmrig-deps

## Build Steps

### Linux/macOS

```bash
git clone https://github.com/danielj86/juno-protocol-xmrig.git
cd juno-protocol-xmrig
mkdir build && cd build
cmake ..
make -j$(nproc)
```

### macOS with Homebrew OpenSSL

```bash
cmake .. -DOPENSSL_ROOT_DIR=$(brew --prefix openssl)
make -j$(sysctl -n hw.ncpu)
```

### Windows (Visual Studio)

```batch
git clone https://github.com/danielj86/juno-protocol-xmrig.git
cd juno-protocol-xmrig
mkdir build
cd build
cmake .. -G "Visual Studio 16 2019" -A x64 -DXMRIG_DEPS=c:\xmrig-deps\msvc2019\x64
cmake --build . --config Release
```

### Windows (MSYS2/MinGW)

```bash
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-cmake mingw-w64-x86_64-libuv mingw-w64-x86_64-openssl mingw-w64-x86_64-hwloc
mkdir build && cd build
cmake .. -G "Unix Makefiles"
make -j$(nproc)
```

## CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `-DSUPPORT_JUNOCASH` | ON | Enable Junocash protocol support |
| `-DWITH_RANDOMX` | ON | Enable RandomX (required for Junocash) |
| `-DWITH_HTTP` | ON | Enable HTTP/daemon support |
| `-DWITH_TLS` | ON | Enable SSL/TLS for secure connections |
| `-DWITH_HWLOC` | ON | Enable hwloc for NUMA optimization |
| `-DWITH_OPENCL` | ON | Enable OpenCL GPU mining |
| `-DWITH_CUDA` | ON | Enable CUDA GPU mining |

## Verify Build

After successful build, verify Junocash support:

```bash
./xmrig --version
# Should show: XMRig 6.24.0 (Juno Protocol)

./xmrig --help | grep -i juno
# Should show junocash coin option
```

## Troubleshooting

### OpenSSL not found
```bash
cmake .. -DOPENSSL_ROOT_DIR=/usr/local/opt/openssl
```

### hwloc not found
```bash
cmake .. -DWITH_HWLOC=OFF
# Or install: sudo apt-get install libhwloc-dev
```

### libuv not found
```bash
cmake .. -DUV_LIBRARY=/usr/local/lib/libuv.a -DUV_INCLUDE_DIR=/usr/local/include
```

### Build fails with missing headers
Ensure all submodules are initialized:
```bash
git submodule update --init --recursive
```
