# Becoming a validator

> **This section documents the target behaviour. It is not live yet.**
>
> The `RegisterValidator` action is implemented and tested, but it is **not
> reachable on any public wire**. It has no typed `/exchange` mapping, no
> governance-CLI lane, and no multisig lane. You cannot register yourself today,
> by any route.
>
> **Today, becoming a validator requires coordination with the network
> operators.** Open an issue on this repository to start that conversation. The
> rules below are the protocol's admission rules, and they are what registration
> will use when the lane opens.

Read [JOINING.md](JOINING.md) first. A validator is a full node with extra
requirements, so every joining rule there applies here too — including the fact
that the operators must add your node to the running validators' configs before
it receives any blocks.

## Registration

Registration is the `RegisterValidator` action. It carries four fields:

| Field | Meaning |
|---|---|
| `signer` | the address that owns the validator |
| `self_stake` | the stake the signer bonds to itself |
| `commission_bps` | commission in basis points; **must be 10000 or less** |
| `consensus_pubkey` | a **33-byte compressed secp256k1** public key |

The consensus public key is the same key your node authenticates with on the
peer-auth port. It is 66 hex characters.

## The self-delegation requirement

`self_stake` must clear the self-delegation requirement. It is currently
**50,000 MTF**.

**This is a governance parameter, not a constant.** Governance can change it, and
the value above is the current default. Read the live value before you rely on
it; do not hard-code 50,000 into any tooling.

## Admission is by stake rank, not first-come

Registering does **not** guarantee activation.

The active set is built by ranking every eligible validator:

1. Sort by **total stake, descending**.
2. Break ties by **address, ascending**.
3. Truncate at `max_active` — **100** on this network.

Jailed, deregistered and zero-stake validators are excluded before ranking.

If 100 validators out-stake you, you sit outside the active set and wait. You are
registered, but you do not vote. Gaining a seat means out-staking the validator
ranked 100th, not waiting your turn.

## The epoch-boundary delay

The active set does not change the moment you register.

The set for the NEXT epoch is pinned from committed staking state at the first
block of each epoch. So a registration takes effect at the epoch boundary
**after** the one it lands in. In the worst case you wait almost two full epochs.

An epoch is **100,000 rounds**.

## Cadence is measured, and it moves

An epoch is 100,000 rounds, not a fixed number of hours. To turn rounds into
time you must measure the chain.

**Measured on 2026-08-21: 9.6 rounds per second.** At that rate:

| | |
|---|---|
| One epoch | about **2.9 hours** |
| Worst-case registration to voting | about **6 hours** |

**This figure moves.** Three readings across three weeks gave 6.3, 8.3 and 9.6
rounds per second. A number printed here is a reading, not a property of the
network. Any fixed block time you see quoted anywhere teaches a value that will
be wrong.

### Measure it yourself

Sample the committed round twice, with a known gap, and divide:

```sh
curl -s -X POST http://127.0.0.1:8080/info \
  -H 'content-type: application/json' -d '{"type":"block_info"}'
```

Read the `round` field. Wait a known number of seconds. Read it again.

```
rounds per second = (round2 - round1) / elapsed_seconds
epoch duration    = 100000 / rounds_per_second
```

Do this before you plan anything around an epoch boundary. Do not derive a rate
from `target_block_interval_ms` — that is a target, not the observed rate.

## Hardware

A validator needs more than an observer. See the hardware table in the
[README](../README.md). RAM is not negotiable.

## Operating duties

- **Keep your key safe.** `validator_key_hex` signs consensus votes. Losing it
  loses the validator; leaking it lets someone else vote as you.
- **Never restart outside a coordinated window.** A lone restart on a live chain
  cannot catch up to the pruned tip. Restarts go through the operators'
  coordinated window.
- **Vote on upgrades.** A freeze needs two-thirds of stake to agree. A validator
  that does not vote slows the whole network down.
- **Never hand-copy `snapshot` and `wal`** from another node, and never
  hand-place a binary. Both fail silently. See [JOINING.md](JOINING.md).
