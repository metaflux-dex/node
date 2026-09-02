# Joining the testnet

This page describes how a full node joins the MetaFlux testnet today.

## Read this first — joining is not fully self-service

**A new node cannot join on its own.** Correct configuration is necessary, but it
is not sufficient.

Each running validator builds its peer-authentication catalog from its OWN static
config file at boot. Your node is not in those files. So the existing validators
reject your node's connections, and they never dial your node. Your node stays at
round 0 and receives no blocks.

**This applies to a read-only observer too.** An observer still authenticates on
the peer-auth port, so it needs the same entry as any other node.

To join, the network operators must add your node id and public key to the
running validators' configs. That change takes effect in a coordinated upgrade
window, because it restarts the validators.

**Contact path:** open an issue on this repository with your node id, your public
key, and the host:port set you will listen on. The network operators will tell
you which upgrade window will carry your entry.

Do not expect steps 1 to 4 below to sync a node by themselves. They prepare a
node that is correct and ready. The coordination step above makes it work.

## This is a testnet, and its data can be reset

The chain was created again on **2026-09-01** with the same `chain_id` (114514)
and a new genesis. It replaced a chain created on 2026-07-26. **That can happen
again.** A new genesis invalidates every node's data directory.

The `chain_id` did not change, so `chain_id` alone does not tell you which chain
you are on. Only the genesis hash does.

Do not treat testnet state as durable. Do not build anything that assumes an
address, a balance, or a block height survives.

## 1. Install the visor and pin the GPG root key

Follow [VERIFYING.md](VERIFYING.md). It covers both verification layers and the
one-time key fetch. Do not skip the fingerprint check.

**Two roots are served, and only one is this chain's.** The unversioned path
still serves the previous chain's root, and its own fingerprint check passes.
Take the key from a release directory and check the fingerprint. That page says
which.

## 2. Take the network files

```sh
# The unit runs as the `mtf` service user, so create it before the directories.
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mtf || true
sudo install -d /etc/mtf
sudo install -d -o mtf -g mtf /var/lib/mtf-visor /var/lib/mtf-node
sudo cp networks/testnet.toml         /etc/mtf/visor.toml
sudo cp networks/testnet/genesis.json /etc/mtf/genesis.json
sudo cp examples/node.toml            /etc/mtf/node.toml
```

The file must have sha256
`ce60d163795bbd39882d783c1484d2e3f7674363c82b38c68b8b0ce3b12c35c6`. Check the
bytes with `sha256sum` FIRST. That anchor sits outside the pair the next check
compares, so it still fails when both members of that pair are wrong together.

Verify the genesis you just copied. The command prints the chain's genesis hash:

```sh
mtf-node genesis-hash --genesis /etc/mtf/genesis.json
```

It must print exactly the contents of `networks/testnet/genesis.hash`:

```
8f6fce34e462c5532f41280d3b262fe76013ce1a1d92dd2470b55bb28ec2102f
```

The hash covers the file's BYTES. Do not reformat or re-key `genesis.json`. Any
byte change moves the hash. The visor then halts your node, because it reports
the wrong chain identity.

## 3. Edit your node config

Open `/etc/mtf/node.toml` and set every `# EDIT` field. The three that stop a
boot when wrong:

1. `node.id` — your node id, assigned by the operators.
2. `node.validator_key_hex` — REQUIRED on testnet. The node fails to boot
   without it, **including a read-only observer**, because the same key
   authenticates your node on the peer-auth port.
3. `network.listen` — the addresses peers dial you on. The gossip, peer_rpc and
   auth ports must be reachable from the internet.

The seed list is already filled in, with the mandatory `pubkey_hex` on it.
Seeds move, so check it first — see "Check the seed list before you trust it" in
the README. The network advertises ONE seed. If that host is offline there is no
seed at all.

## 4. Start it

```sh
sudo cp deploy/mtf-visor.service /etc/systemd/system/mtf-visor.service
sudo systemctl daemon-reload
sudo systemctl enable --now mtf-visor
```

Watch the logs. Until your entry lands in the validators' configs, expect
rejected connections and no block progress. That is the coordination gap above,
not a misconfiguration on your side.

## Catching up: the certified bootstrap

A node that falls too far behind cannot replay its way back. It recovers like
this:

1. The node exits **80** to ask for a certified bootstrap.
2. The visor restarts it after a backoff.
3. The node adopts a state bundle certified by **two-thirds of stake**,
   recomputes the app hash, and **refuses a mismatch**.

Measured 2026-08-21: about 5 minutes. That is a reading, not a guarantee. You
do nothing. Never add 80 to `RestartPreventExitStatus` — that blocks the node's
own recovery path.

### Never hand-copy `snapshot` and `wal` from a peer

**This is forbidden, and it fails silently.** A node bootstrapped that way
rejoins and commits in lockstep with everyone else, so it looks healthy. Its app
hash then forks at every round, and nothing tells you.

The certified bootstrap is the only supported catch-up. It is the only path that
recomputes the app hash and rejects a mismatch.

### Never hand-place a binary

Binaries arrive only through the visor's verified manifest flow. A hand-placed
binary skips both verification layers. See [VERIFYING.md](VERIFYING.md).

## Upgrades

You do nothing. A binary upgrade is a network-wide, governance-gated event:

1. Validators vote on-chain for an upgrade at a freeze height and a target build.
   Once validators holding two-thirds of stake agree, the chain records the
   freeze.
2. Your visor has already downloaded and verified the target binary.
3. At the freeze height every honest node halts and exits 77.
4. Your visor swaps the verified binary in and restarts the node.

A visor with no verified matching binary halts instead of running the old binary
past the freeze. It exits 78, and that needs you to act.

## Measuring the chain, not assuming it

You **measure** block cadence. It is never fixed, and it moves. Never size
anything on a configured interval. `target_block_interval_ms` is a target, not
the observed rate.

The procedure has one home: [VALIDATOR.md](VALIDATOR.md), "Measure it
yourself".
