# Verifying binaries

The network runs **two independent release-verification layers**. Both must pass
before the visor makes a binary runnable.

| Layer | Covers | Key material | Checked by |
|---|---|---|---|
| 1. Threshold manifest signature + content hash | the release manifest, and each binary's bytes | secp256k1 `release_pubkeys` in your visor config | the visor, automatically |
| 2. Detached GPG signature | each binary's bytes | the offline GPG root key you pin as `release_gpg_pubkey` | the visor, automatically |

The layers are independent on purpose. Layer 1 keys live in CI. The layer 2 root
key lives offline. An attacker who takes CI still cannot produce a binary the
visor will run.

## Layer 1 — threshold signature and content hash

For every update the visor:

1. Fetches the release manifest from the configured `manifest_url`.
2. Verifies the manifest's **threshold secp256k1 signature** against the
   `release_pubkeys` you pinned. At least `release_threshold` distinct in-policy
   keys must have signed. **High-S signatures are rejected**, so a valid
   signature cannot be mutated into a second valid one.
3. Checks the **chain binding** — the manifest's `chain_id` and `genesis_hash`
   must match your node — on every poll and after every restart.
4. Refuses any manifest at or below the last accepted **sequence** or version.
   The accepted sequence only ever advances, so a release cannot be rolled back
   under you.
5. Downloads the binary from the URL in the manifest, and verifies its **blake3**
   content hash against the signed value before staging it.
6. Executes the verified file by open descriptor, so nothing can swap the file
   between the final check and exec.

The download URL is part of the signed manifest, so it cannot be redirected to a
different host. There is nothing for you to do here.

`release_threshold` is the number of keys that must sign. **Mainnet requires at
least 2**, and the visor refuses to start a mainnet config that sets less. One
release key must never be able to push a binary network-wide. Testnet runs a
threshold of 1.

## Layer 2 — the offline GPG root key

Every released binary also carries a detached GPG signature, published beside it
as `<binary>.asc`. The visor imports your pinned key into an **isolated keyring**
that holds only that key, then runs `gpg --verify` on each staged binary. Because
the keyring holds one key and no web of trust, a good signature is necessarily
from the root.

A missing `gpg`, a missing `.asc`, or a bad signature all **reject**. None of
them silently pass. The binary is made executable only after it is durable on
disk AND both its blake3 hash and its GPG signature pass.

### Fetch and pin the key, once

The visor **refuses to start** an `[update]` section that does not set
`release_gpg_pubkey`. This applies on every network, testnet included.

```sh
sudo install -d /etc/mtf
sudo curl -fsSL -o /etc/mtf/pub_key.asc \
  "https://binaries.mtf.exchange/testnet/pub_key.asc"

# Confirm the fingerprint before you trust it.
gpg --show-keys --with-fingerprint /etc/mtf/pub_key.asc
```

The fingerprint must be exactly:

```
5AF6597573B2E475B0C646BAD8E6D0B3D187F583
```

Then point your visor config at the file. `networks/testnet.toml` already does:

```toml
[update]
release_gpg_pubkey = "/etc/mtf/pub_key.asc"
```

`gpg` must be installed on the host or in the container. The visor shells out to
it, and fails loudly at updater start when it is absent.

## The visor binary — verify it yourself, once

The two layers above protect the **node** binary. You download the **visor**
yourself, so verify it before the first run. Each published visor binary has a
companion `.sha256`:

```sh
ARCH=x86_64-unknown-linux-musl          # or aarch64-unknown-linux-musl
BASE="https://binaries.mtf.exchange/visor/latest/${ARCH}"

curl -fsSL -o mtf-visor        "${BASE}/mtf-visor"
curl -fsSL -o mtf-visor.sha256 "${BASE}/mtf-visor.sha256"
sha256sum -c mtf-visor.sha256          # must print: mtf-visor: OK

sudo install -m 0755 mtf-visor /usr/local/bin/mtf-visor
```

Pin a specific version by replacing `latest` with the version.
`deploy/Dockerfile` runs this same check, and the build fails on a mismatch.

> Roadmap: a detached GPG signature and a reproducible-build attestation for the
> visor binary itself, so its provenance is verifiable exactly like the node's.

## Never hand-place a binary

Binaries arrive only through the visor's verified manifest flow. Copying a
binary into place yourself skips **both** layers — no threshold signature, no
content hash, no GPG signature. Do not do it, not even to recover a stuck node.
