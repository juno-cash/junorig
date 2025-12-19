# Juno Cash Solo Mining

## Single Miner (Direct to Node)

```
[Juno Node:8232] <--RPC/ZMQ--> [xmrig]
```

`~/.junocash/junocashd.conf`:
```ini
server=1
rpcuser=YOUR_RPC_USER
rpcpassword=YOUR_RPC_PASSWORD
mineraddress=YOUR_JUNO_WALLET_ADDRESS
zmqpubhashblock=tcp://127.0.0.1:28332
```

Run:
```bash
./xmrig -o 127.0.0.1:8232 --daemon --daemon-zmq-port=28332 -a rx/juno -u YOUR_RPC_USER -p YOUR_RPC_PASSWORD
```

## Multi-Miner Setup

For multiple miners, use [juno-xmrig-proxy](https://github.com/user/juno-xmrig-proxy):

```
[Juno Node:8232] <--RPC/ZMQ--> [xmrig-proxy:3334] <--stratum--> [miners]
```

Proxy:
```bash
./xmrig-proxy -o 127.0.0.1:8232 --daemon --daemon-zmq-port=28332 -a rx/juno -u YOUR_RPC_USER -p YOUR_RPC_PASSWORD -b 0.0.0.0:3334
```

Miners:
```bash
./xmrig -o PROXY_IP:3334 -u worker1 -a rx/juno
```

## Notes

- `-u`/`-p` are RPC credentials when connecting to daemon
- `mineraddress` on the node determines where block rewards go
- ZMQ enables instant block notifications (recommended)
