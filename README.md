# MetaFlux node

Everything an operator needs to run a **MetaFlux (MTF)** node: network
parameters, configuration templates, and deployment recipes. This repository does
not contain the node source. It is the deployment surface.

A MetaFlux deployment is two layers:

```
systemd / docker  →  visor  →  node
```

The visor is the only process you launch. It:

- **supervises** the node (crash-restart with backoff, log capture);
- **auto-updates** it from a release manifest, verified by two independent
  signature layers; and
- **swaps the binary in lockstep with the network** at a governance-set freeze
  height, so your node upgrades hands-free without forking.

## Hardware

| Role | vCPU | RAM | Disk |
|---|---|---|---|
| Validator | 32 | 128 GB | 1 TB NVMe SSD |
| Non-validator | 16 | 128 GB | 500 GB NVMe SSD |

- **OS:** Linux on x86-64 or arm64. The binaries are static `musl`, so any modern
  distribution works. Ubuntu 24.04 LTS is the reference.
- **RAM is not negotiable.** The matching engine, EVM and consensus working set
  are memory-resident by design.
- **Disk:** NVMe SSD. The write-ahead log is fsync'd on every commit, so storage
  latency bounds commit throughput. Size for the WAL and snapshot history, plus
  the visor's staged binaries and rotated logs.
- **`gpg` must be installed.** The visor shells out to it to check release
  signatures, and fails at start without it.
- The consensus transport ports must be reachable by peers. See [Ports](#ports).

## Networks

| Network | chain_id | Visor config | Genesis |
|---|---|---|---|
| Testnet | 114514 | [`networks/testnet.toml`](networks/testnet.toml) | [`networks/testnet/genesis.json`](networks/testnet/genesis.json) |
| Mainnet | — | — | **Not launched. The mainnet config and genesis will be published in this repository at launch.** |

The visor config pins that network's release keys, manifest URL, GPG root key
path, and chain binding. Fields you must set are marked `# EDIT`.

The **genesis** is the chain's creation artifact: chain_id, genesis time, and the
full validator set (address, pubkey, stake). The node binds its genesis block to
the genesis hash, so every node that boots from the same `genesis.json` agrees on
one chain identity. Point your node config's `genesis_file` at it, and verify the
published hash:

```sh
mtf-node genesis-hash --genesis networks/testnet/genesis.json
# must print the contents of networks/testnet/genesis.hash
```

The hash covers the file's BYTES. Do not reformat or re-key `genesis.json`.

## Install

Full walkthrough: [docs/JOINING.md](docs/JOINING.md).

### 1. Get the visor binary, and verify it

```sh
ARCH=x86_64-unknown-linux-musl          # or aarch64-unknown-linux-musl
BASE="https://binaries.mtf.exchange/visor/latest/${ARCH}"

curl -fsSL -o mtf-visor        "${BASE}/mtf-visor"
curl -fsSL -o mtf-visor.sha256 "${BASE}/mtf-visor.sha256"
sha256sum -c mtf-visor.sha256          # must print: mtf-visor: OK

sudo install -m 0755 mtf-visor /usr/local/bin/mtf-visor
```

### 2. Pin the GPG root key

```sh
sudo install -d /etc/mtf
sudo curl -fsSL -o /etc/mtf/pub_key.asc \
  "https://binaries.mtf.exchange/testnet/pub_key.asc"
gpg --show-keys --with-fingerprint /etc/mtf/pub_key.asc
# must show 5AF6597573B2E475B0C646BAD8E6D0B3D187F583
```

The visor refuses to start without this key pinned. See
[docs/VERIFYING.md](docs/VERIFYING.md).

### 3. Configure

```sh
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mtf || true
sudo install -d -o mtf -g mtf /var/lib/mtf-visor /var/lib/mtf-node
sudo cp networks/testnet.toml         /etc/mtf/visor.toml
sudo cp networks/testnet/genesis.json /etc/mtf/genesis.json
sudo cp examples/node.toml            /etc/mtf/node.toml
# then edit the `# EDIT` fields in both configs
```

### 4. Run

- **systemd:** install [`deploy/mtf-visor.service`](deploy/mtf-visor.service),
  then `systemctl enable --now mtf-visor`.
- **docker:** run the same three install steps above so `/etc/mtf` holds
  `visor.toml`, `node.toml` and `genesis.json`, then
  `docker compose -f deploy/docker-compose.yml up -d`. The compose file mounts
  that directory; it does not carry its own copies.

## Read before you start

- **This is a testnet, and its data can be reset.** The chain was re-genesised on
  2026-07-26 with the same `chain_id` and a new genesis. That can happen again.
  Do not treat testnet state as durable.
- **Joining is not fully self-service.** Correct config is not enough. The
  network operators must add your node to the running validators' configs before
  it receives any blocks — a read-only observer included. See
  [docs/JOINING.md](docs/JOINING.md).
- **Validator registration is not live yet.** See
  [docs/VALIDATOR.md](docs/VALIDATOR.md).

## Trust model, in one paragraph

The network runs two independent release-verification layers, and the visor
checks both before it makes any binary runnable. Layer one is a threshold
secp256k1 signature over the release manifest plus a blake3 content hash of the
binary; the manifest also binds the `chain_id` and `genesis_hash`, and its
sequence only ever advances, so a release cannot be rolled back under you. Layer
two is a detached GPG signature checked against an offline root key that you
fetch once and pin. The layer-one keys live in CI and the layer-two root lives
offline, so taking CI is still not enough to make your node run a hostile binary.
You verify the visor itself, once, with its published checksum. Full detail in
[docs/VERIFYING.md](docs/VERIFYING.md).

## How upgrades work

A binary upgrade is a network-wide, governance-gated event, never a per-operator
action:

1. Validators vote on-chain for an upgrade at a freeze height and a target build.
   Once at least two-thirds of staked validators agree, the chain records the
   freeze.
2. Your visor has meanwhile downloaded and verified the target binary.
3. At the freeze height every honest node halts deterministically and exits 77.
4. Your visor sees that exit, confirms the staged binary matches the freeze,
   swaps it in, and restarts the node on the new version.

A visor with no verified matching binary halts loudly instead of running the old
binary past the freeze. **The node exits 77; the visor reports 78 to the
supervisor.** The systemd unit prevents a restart on 78 only.

You do nothing.

## Ports

**Every address below is a listen address for YOUR node.** They are not
project-hosted endpoints, and this table is not a list of servers to connect to.
Choose ports that suit your host.

| Purpose | Template default | Exposure |
|---|---|---|
| Consensus transport (gossip / peer RPC / auth) | 4001 / 4002 / 4003 | **Public** — peers must reach these |
| REST API / WebSocket | 8080 | Your choice (local or public) |
| EVM JSON-RPC | 8545 | Your choice |
| Metrics | 9100 (node) / 9110 (visor) | Local / monitoring network |

**The EVM JSON-RPC starts only when the REST API is also configured.** It reuses
the same read handle, so with `api_listen` unset the node logs a warning and
skips `evm_rpc_listen`.

## What the node serves

Set `[observability] api_listen` and the node serves its own API. Everything
below comes from the node's committed state, so it answers from your own machine
with no dependency on anyone else's endpoint.

**`POST /info` — 74 request types.** Markets and metadata, per-account state,
open orders, fills, funding, ledger history, staking, vaults, governance,
bridge state, and the order book. One shape throughout:

```sh
curl -s http://127.0.0.1:8080/info \
  -H 'content-type: application/json' \
  -d '{"type":"markets"}'
```

Every response is `{"type": …, "data": …}`. Numbers that carry precision are
strings, so no client loses a digit to a float.

**`GET /ws` — 22 subscription channels.** Market data: `l2_book`, `bbo`,
`trades`, `candles`, `all_mids`, `active_asset_ctx`, `markets`. Per-account,
each requiring a `user` address: `fills`, `order_updates`, `user_events`,
`notifications`, `ledger_updates`, `open_orders`, `user_fundings`,
`account_state`, `web_data`, `active_asset_data`, `spot_margin_state`,
`user_twap_slice_fills`, `user_twap_history`. Chain: `explorer_block`,
`explorer_txs`.

```json
{"method":"subscribe","subscription":{"type":"trades","coin":"BTC"}}
```

The socket also carries a `post` lane, so one connection can both stream and
make request/response calls:

```json
{"method":"post","id":1,"request":{"type":"info","payload":{"type":"markets"}}}
```

**EVM JSON-RPC.** Set `evm_rpc_listen` and the node serves standard Ethereum
JSON-RPC. It needs `api_listen` set as well, because it reuses that read handle;
with `api_listen` unset the node logs a warning and skips it.

**Prometheus metrics** on `metrics_listen`. `mtf_committed_round` is the one to
watch: sample it twice and see it climb.

## Reading L1 data

The node can write each block's activity to hourly-rotated NDJSON under its data
directory. Every stream is off by default and none of them touch the app hash.

| `[persistence]` key | Written to |
|---|---|
| `write_trades` | `node_trades/hourly/{date}/{hour}` |
| `write_fills` | `node_fills/hourly/{date}/{hour}` |
| `write_order_statuses` | `node_order_statuses/hourly/{date}/{hour}` |
| `write_funding` | `node_funding/hourly/{date}/{hour}` |
| `write_ledger` | `node_ledger/hourly/{date}/{hour}` |
| `write_equity_snapshots` | `node_equity_snapshots/hourly/{date}/{hour}` |
| `write_gov` | `node_gov/hourly/{date}/{hour}` |
| `write_asset_ctxs` | `node_asset_ctxs/hourly/{date}/{hour}` |
| `write_replica_cmds` | `replica_cmds/{date}/{hour}` |
| `record_l2` | `l2_book_diffs.jsonl` |
| `record_l4` | `l4_book_diffs.jsonl` |

**Most of these are refused on a validator, and that refusal is deliberate.**
They de-anonymise order flow — they name the addresses behind each fill — and
they add write load to a machine whose job is consensus. Run them on a dedicated
non-validating node. `allow_stream_on_validator` overrides the refusal if you
accept both costs.

Two are exempt because they carry only public data: `write_gov` (governance
votes, already public) and `write_asset_ctxs` (mark, oracle, funding and open
interest per market).

`record_l4` is the one to be careful with. It records de-anonymised book diffs,
which is every resting order with its owner. `record_l2` is the aggregated
sibling: price levels only, no owner and no order id.

## Data and logs

- **Visor home** (`/var/lib/mtf-visor`): staged binaries, the upgrade journal,
  and rotated logs, including the node's captured stderr.
- **Node data directory**: the write-ahead log and snapshots.
- **Never hand-copy `snapshot` and `wal` from a peer.** A node bootstrapped that
  way looks healthy and forks its app hash silently at every round. The certified
  bootstrap is the only supported catch-up. See
  [docs/JOINING.md](docs/JOINING.md).

## Documentation

| Page | What it covers |
|---|---|
| [docs/JOINING.md](docs/JOINING.md) | How a full node joins today, and the coordination it needs |
| [docs/VALIDATOR.md](docs/VALIDATOR.md) | Validator admission rules, and what is not live yet |
| [docs/VERIFYING.md](docs/VERIFYING.md) | Both release-verification layers |

## License

[Apache-2.0](LICENSE).
