# MetaFlux node

Everything an operator needs to run a **MetaFlux (MTF)** node: network
parameters, configuration templates, and deployment recipes. This repository does
not contain the node source — it is the deployment surface.

A MetaFlux deployment is two layers:

```
systemd / docker  →  mtf-visor  →  mtf-node
```

`mtf-visor` is the only process you launch. It:

- **supervises** `mtf-node` (crash-restart with exponential backoff, log capture);
- **auto-updates** it from a signature-verified release manifest; and
- **swaps the binary in lockstep with the network** at a governance-set upgrade
  height, so your node upgrades hands-free without forking.

## Hardware

| Role | vCPU | RAM | Disk |
|---|---|---|---|
| Validator | 32 | 128 GB | 1 TB NVMe SSD |
| Non-validator | 16 | 128 GB | 500 GB NVMe SSD |

- **OS:** Linux on x86-64 or arm64. The binaries are static `musl`, so any modern
  distribution works; Ubuntu 24.04 LTS is the reference.
- **RAM is not negotiable** — the matching engine, EVM, and consensus working set
  are memory-resident by design.
- **Disk:** NVMe SSD. The write-ahead log is fsync'd on every commit, so storage
  latency bounds commit throughput. Size for the WAL + snapshot history plus the
  visor's staged binaries and rotated logs.
- The consensus transport ports must be reachable by peers — see [Ports](#ports).

## Networks

| Network | chain_id | Visor config | Genesis |
|---|---|---|---|
| Testnet | 114514 | [`networks/testnet.toml`](networks/testnet.toml) | [`networks/testnet/genesis.json`](networks/testnet/genesis.json) |
| Mainnet | 8964 | [`networks/mainnet.toml`](networks/mainnet.toml) | _(published at launch)_ |

The visor config pins that network's release-signing keys, manifest URL, and
chain binding (fields you must set are marked `# EDIT`).

The **genesis** is the chain's creation artifact — chain_id, genesis time, and the
full validator set (address, pubkey, stake). The node binds its genesis block to
the genesis hash, so every node that boots from the same `genesis.json` agrees on
one chain identity. Point your node config's `genesis_file` at it, and verify the
published hash:

```sh
mtf-node genesis-hash --genesis networks/testnet/genesis.json
# must print the contents of networks/testnet/genesis.hash
```

## Install

### 1. Get the visor binary

```sh
ARCH=x86_64-unknown-linux-musl          # or aarch64-unknown-linux-musl
sudo curl -fsSL -o /usr/local/bin/mtf-visor \
  "https://binaries.mtf.exchange/visor/latest/${ARCH}/mtf-visor"
sudo chmod +x /usr/local/bin/mtf-visor
```

Verify the binary before running it — see [docs/VERIFYING.md](docs/VERIFYING.md).

### 2. Configure

```sh
sudo install -d /etc/mtf /var/lib/mtf-visor
sudo cp networks/testnet.toml /etc/mtf/visor.toml      # pick your network
sudo cp examples/node.toml    /etc/mtf/node.toml
# then edit the `# EDIT` fields in both
```

### 3. Run

- **systemd:** install [`deploy/mtf-visor.service`](deploy/mtf-visor.service), then
  `systemctl enable --now mtf-visor`.
- **docker:** use [`deploy/docker-compose.yml`](deploy/docker-compose.yml).

## How upgrades work

A binary upgrade is a network-wide, governance-gated event — never a per-operator
action:

1. Validators vote on-chain for an upgrade at freeze height `F` and the target
   build. Once ≥ 2/3 of staked validators agree, the chain records the freeze.
2. Your visor has meanwhile pre-downloaded and signature-verified the target
   binary from the release manifest.
3. At height `F` every honest node halts deterministically and exits.
4. Your visor sees the exit, confirms the staged binary matches the freeze, swaps
   it in, and restarts the node on the new version. A visor with no matching
   verified binary halts loudly rather than running the old binary past the
   freeze.

You do nothing. See [docs/VERIFYING.md](docs/VERIFYING.md) for the trust model.

## Ports

| Purpose | Default | Exposure |
|---|---|---|
| Consensus transport (gossip / peer RPC / auth) | 4001 / 4002 / 4003 | **Public** — peers must reach these |
| REST API / WebSocket | 8080 | Your choice (local or public) |
| EVM JSON-RPC | 8545 | Your choice |
| Metrics | 9100 (node) / 9110 (visor) | Local / monitoring network |

## Data & logs

- **Visor home** (`/var/lib/mtf-visor`): staged binaries, the upgrade journal,
  and rotated logs (including the node's captured stderr).
- **Node data directory**: the write-ahead log and snapshots. Back this up.
- Consensus logs surface under the node's log output; the visor's `passthrough`
  setting tees the child's stdio.

## License

[Apache-2.0](LICENSE).
