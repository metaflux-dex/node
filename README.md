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

## What a node gives you

A node is not a follower that needs the hosted API to be useful. Set
`[observability] api_listen` and it answers the reads below from its own
committed state, and it accepts orders. The four reads under [What a node does
not have](#what-a-node-does-not-have) are the exception. No hosted API is in the
path.

**`POST /exchange` — the write side.** Submit EIP-712 signed actions straight to
your own node's mempool: orders, cancels, transfers, staking, governance votes.

**`POST /info` — 74 request types, from committed state.** Markets and metadata,
the order book, per-account state, open orders, positions, balances, funding
rates, fees, staking, vaults, governance, and bridge state.

```sh
curl -s http://127.0.0.1:8080/info \
  -H 'content-type: application/json' -d '{"type":"markets"}'
```

Every response is `{"type": …, "data": …}`. Any number that carries precision is
a string. No client loses a digit to a float.

**`GET /ws` — folded from the node's own blocks.** Market data:
`l2_book`, `bbo`, `trades`, `all_mids`, `active_asset_ctx`, `markets`.
Per-account, each taking a `user` address: `fills`, `order_updates`,
`user_events`, `notifications`, `ledger_updates`, `open_orders`,
`user_fundings`, `account_state`, `web_data`, `active_asset_data`,
`spot_margin_state`, `user_twap_slice_fills`, `user_twap_history`.

There is no `candles` channel. A node does not aggregate OHLCV — a subscribe is
refused as an unknown channel. The hosted API, or your own indexer, builds bars
from the `trades` stream.

The socket carries two explorer tapes as well. The list above omits them. They
duplicate what an archive already serves. They may be removed, for the reason
that removed candles: a consensus node emits its blocks. It does not maintain a
viewer's feed.

```json
{"method":"subscribe","subscription":{"type":"trades","coin":"BTC"}}
```

The socket also carries a `post` lane, so one connection can stream and make
request/response calls at the same time:

```json
{"method":"post","id":1,"request":{"type":"info","payload":{"type":"markets"}}}
```

**EVM JSON-RPC.** Set `evm_rpc_listen` and the node answers standard Ethereum
JSON-RPC — `eth_call`, `eth_getLogs`, `eth_sendRawTransaction`,
`eth_getTransactionReceipt`, `eth_subscribe` and the rest of the usual set — for
the EVM that executes inside each consensus round. It needs `api_listen` set as
well, because it reuses that read handle.

**Prometheus metrics** on `metrics_listen`. `mtf_committed_round` is the one to
watch: sample it twice and see it climb.

### What a node does not have

A node holds committed state and a bounded recent window, not an archive. Four
account-history reads answer with an empty array rather than a wrong one:
`user_funding`, `user_ledger_updates`, `user_twap_slice_fills` and
`delegator_history`. The node streams these events. It does not retain them for
REST.
`historical_orders` returns executed fills only.

Deep history, cross-account aggregation and leaderboards are not part of this
repository. If you need them, either consume the streams under
[Reading L1 data](#reading-l1-data) and build the index you want, or use the
hosted API.

## Hardware

| Role | vCPU | RAM | Disk |
|---|---|---|---|
| Validator | 32 | 128 GB | 1 TB NVMe SSD |
| Non-validator | 16 | 128 GB | 500 GB NVMe SSD |

- **OS:** Linux on x86-64 or arm64. The binaries are static `musl`, so any modern
  Linux distribution works, provided `gpg` is installed. Ubuntu 24.04 LTS is the
  reference.
- **RAM is not negotiable.** The matching engine, EVM and consensus working set
  are memory-resident by design.
- **Disk:** NVMe SSD. The node fsyncs the write-ahead log on every commit, so
  storage latency bounds commit throughput. Size for the WAL and snapshot
  history, plus the visor's staged binaries and rotated logs.
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

`chain_id` alone does not identify a chain here. The testnet kept `chain_id`
114514 across a new genesis, so two different chains share that number. Only the
genesis hash tells them apart.

The **genesis** is the chain's creation artifact: chain_id, genesis time, the
epoch length (`epoch_rounds`), the active-set cap (`genesis_max_active`), and the
full validator set (address, pubkey, stake). The node binds its genesis block to
the genesis hash. Every node that boots from the same `genesis.json` agrees on
one chain identity. Point your node config's `genesis_file` at it, and verify the
published hash:

```sh
sha256sum networks/testnet/genesis.json
# the published file has sha256
# ce60d163795bbd39882d783c1484d2e3f7674363c82b38c68b8b0ce3b12c35c6

mtf-node genesis-hash --genesis networks/testnet/genesis.json
# must print the contents of networks/testnet/genesis.hash
```

Check the `sha256sum` first. It is an anchor outside the pair that the second
command compares, so it still fails when both members of that pair are wrong
together.

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

> **Take the key from a RELEASE path, and check the fingerprint.** The
> unversioned paths `testnet/pub_key.asc` and `testnet/node/pub_key.asc` serve a
> DIFFERENT root. That key verifies no release of this chain, and its own
> fingerprint check passes, so the wrong key looks right. The fingerprint below
> is what identifies the correct key. Trust it, not the path.

```sh
sudo install -d /etc/mtf
sudo curl -fsSL -o /etc/mtf/pub_key.asc \
  "https://binaries.mtf.exchange/testnet/node/0.9.1/pub_key.asc"
gpg --show-keys --with-fingerprint /etc/mtf/pub_key.asc
# must show 04781F5109BA16B3CCB43D1E66E38A32E5A25D73
```

`0.9.1` is a release version. There is no version-independent path yet, so read
the current version from the manifest and substitute it:

```sh
curl -fsS https://binaries.mtf.exchange/testnet/node/manifest.json \
  | grep -o '"version": *"[^"]*"'
```

The key is the same across releases. If a future release serves a key with a
different fingerprint, STOP and ask the network operators.

The visor refuses to start without this key pinned. See
[docs/VERIFYING.md](docs/VERIFYING.md).

### 3. Configure

```sh
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mtf || true
sudo install -d /etc/mtf
sudo install -d -o mtf -g mtf /var/lib/mtf-visor /var/lib/mtf-node
sudo cp networks/testnet.toml         /etc/mtf/visor.toml
sudo cp networks/testnet/genesis.json /etc/mtf/genesis.json
sudo cp examples/node.toml            /etc/mtf/node.toml
# then edit the `# EDIT` fields in both configs
```

The `genesis.json` you just copied is the PREVIOUS chain's file — see
[Networks](#networks). Replace it with the published file before you start the
node. `examples/node.toml` carries the previous chain's validator set for the
same reason, and the node fails closed until both match.

### 3b. Check the seed list before you trust it

`examples/node.toml` ships the seed list that the network advertised on the day
it was written. Seeds move. Ask a running node for the current seed list rather
than assuming the file is still right:

```sh
curl -s -X POST https://api.testnet.mtf.exchange/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"gossip_root_ips"}'
```

Each row carries an `id`, the three consensus endpoints, and `pubkey_hex`. A
`peers` entry takes the same shape, so a row is copied across as it stands:

```json
{ "data": { "peers": [
  { "id": 1, "gossip": "host:4001", "peer_rpc": "host:4002", "auth": "host:4003",
    "pubkey_hex": "<66 hex chars>" }
] } }
```

`pubkey_hex` is MANDATORY. The node refuses to boot without it.

`genesis.json` carries the same key under a different name: the field is
`pubkey`, and the validator is keyed by `index`, not `id`. The two must agree. If
they disagree, trust `genesis.json` — that is the file the node cross-checks at
boot.

**An empty list is an answer, not a failure.** It says the network advertises no
public addresses — there is no fallback to the addresses its nodes dial each
other on, which you could not reach anyway. Ask the network operators for
addresses.

### 4. Run

- **systemd:** install [`deploy/mtf-visor.service`](deploy/mtf-visor.service),
  then `systemctl enable --now mtf-visor`.
- **docker:** run install step 3 above, so `/etc/mtf` holds
  `visor.toml`, `node.toml` and `genesis.json`, then
  `docker compose -f deploy/docker-compose.yml up -d`. The compose file mounts
  that directory; it does not carry its own copies. The image pins the GPG root
  key itself, so the build fails until that key is published — see step 2.

## Read before you start

- **This is a testnet, and its data can be reset.** The chain was created again
  on 2026-09-01 with the same `chain_id` and a new genesis. It replaced a chain
  created on 2026-07-26. That can happen again. Do not treat testnet state as
  durable.
- **Joining is not fully self-service.** Correct config is not enough. The
  network operators must add your node to the running validators' configs before
  it receives any blocks — a read-only observer included. See
  [docs/JOINING.md](docs/JOINING.md).
- **Validator registration is not live yet.** See
  [docs/VALIDATOR.md](docs/VALIDATOR.md).

## Trust model, in one paragraph

The network runs two independent release-verification layers, and the visor
checks both before it makes any binary runnable. Layer 1 is a threshold secp256k1
signature over the release manifest. It also covers a blake3 content hash of the
binary. The manifest binds the `chain_id` and `genesis_hash`. Its sequence only
advances, so nobody can roll a release back under you. Layer 2 is a detached GPG
signature, checked against an offline root key that you fetch once and pin. The
Layer 1 keys live in CI. The Layer 2 root key lives offline. An attacker who
takes CI still cannot make your node run a hostile node binary. You verify the
visor itself, once, with its published checksum. Full detail in
[docs/VERIFYING.md](docs/VERIFYING.md).

## How upgrades work

A binary upgrade is a network-wide, governance-gated event, never a per-operator
action:

1. Validators vote on-chain for an upgrade at a freeze height and a target build.
   Once validators holding two-thirds of stake agree, the chain records the
   freeze.
2. Your visor has meanwhile downloaded and verified the target binary.
3. At the freeze height every honest node halts deterministically and exits 77.
4. Your visor sees that exit, confirms the staged binary matches the freeze,
   swaps it in, and restarts the node on the new version.

A visor with no verified matching binary halts instead of running the old binary
past the freeze. **The node exits 77; the visor reports 78 to systemd.** The
systemd unit prevents a restart on 78 only. Docker cannot tell 78 from a crash
and restarts the container. Use the systemd unit for a validator.

In the normal case you do nothing.

## Ports

**Every address below is a listen address for YOUR node.** They are not
project-hosted endpoints, and this table is not a list of servers to connect to.
Choose ports that suit your host.

| Purpose | Template default | Exposure |
|---|---|---|
| Consensus transport (gossip / peer RPC / auth) | 4001 / 4002 / 4003 | **Public** — peers must reach these |
| REST API / WebSocket | 8080 | Your choice (local or public) — the visor reads it too |
| EVM JSON-RPC | 8545 | Your choice |
| Metrics | 9100 (node) / 9110 (visor) | Local / monitoring network |

**The EVM JSON-RPC starts only when the REST API is also configured.** It reuses
the same read handle, so with `api_listen` unset the node logs a warning and
skips `evm_rpc_listen`.

**Automatic upgrades need `api_listen` too.** The visor reads the node's upgrade
state over that address. `info_url` in your visor config must equal `api_listen`.
If they disagree, the visor cannot see a freeze, and the swap described in [How
upgrades work](#how-upgrades-work) never happens.

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

**A validator refuses most of these, and that refusal is deliberate.** They
de-anonymise order flow: they name the address behind each fill. They also add
write load to a machine whose job is consensus. Run them on a dedicated
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
