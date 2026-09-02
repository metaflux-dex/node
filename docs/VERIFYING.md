# Verifying binaries

The network runs **two independent release-verification layers**. Both must pass
before the visor makes a binary runnable.

| Layer | Covers | Key material | Checked by |
|---|---|---|---|
| 1. Threshold manifest signature + content hash | the release manifest, and each binary's bytes | secp256k1 `release_pubkeys` in your visor config | the visor, automatically |
| 2. Detached GPG signature | each binary's bytes | the offline GPG root key you pin as `release_gpg_pubkey` | the visor, automatically |

The layers are independent on purpose. Layer 1 keys live in CI. The Layer 2 root
key lives offline. An attacker who takes CI still cannot produce a node binary
the visor will run.

## Layer 1 — threshold signature and content hash

For every update the visor:

1. Fetches the release manifest from the configured `manifest_url`.
2. Verifies the manifest's **threshold secp256k1 signature** against the
   `release_pubkeys` you pinned. At least `release_threshold` distinct in-policy
   keys must have signed. The visor **rejects high-S signatures**. Nobody can
   mutate a valid signature into a second valid one.
3. Checks the **chain binding** — the manifest's `chain_id` and `genesis_hash`
   must match your node — on every poll and after every restart.
4. Refuses any manifest at or below the last accepted **sequence** or version.
   The accepted sequence only ever advances, so a release cannot be rolled back
   under you.
5. Downloads the binary from the URL in the manifest, and verifies its **blake3**
   content hash against the signed value before staging it.
6. Executes the verified file by open descriptor, so nothing can swap the file
   between the final check and exec.

The signed manifest carries the download URL. Nobody can redirect it to another
host. There is nothing for you to do here.

`release_threshold` is the number of keys that must sign. **Mainnet requires at
least 2**, and the visor refuses to start a mainnet config that sets fewer. One
release key must never be able to push a binary network-wide. Testnet runs a
threshold of 1.

## Layer 2 — the offline GPG root key

Every released binary also carries a detached GPG signature, published beside it
as `<binary>.asc`. The visor imports your pinned key into an **isolated
keyring**. That keyring holds only your key. The visor then runs `gpg --verify`
on each staged binary. The keyring holds one key and no web of trust, so a good
signature is necessarily from the root.

A missing `gpg`, a missing `.asc`, or a bad signature all **reject**. None of
them silently pass. The visor makes the binary executable last. First the file
must be durable on disk. Then the blake3 hash and the GPG signature must both
pass.

### Fetch and pin the key, once

> **Two roots are served. Take the one under a RELEASE directory.**
>
> The chain was created again on 2026-09-01. The unversioned paths
> `testnet/pub_key.asc` and `testnet/node/pub_key.asc` still serve the
> **previous** chain's root, fingerprint
> `5AF6597573B2E475B0C646BAD8E6D0B3D187F583`. It verifies **no release of this
> chain**, and its own fingerprint check passes — so the wrong key looks right.
> That is the trap: a passing check on a key that rejects every binary you will
> be asked to run.
>
> The fingerprint identifies the correct key. Check it every time. Do not trust
> a path.

The visor **refuses to start** an `[update]` section that does not set
`release_gpg_pubkey`. This applies on every network, testnet included. So there
is no configuration that skips this step and still gets updates.

When the key lands, fetch it and confirm the fingerprint before you trust it:

```sh
sudo install -d /etc/mtf
sudo curl -fsSL -o /etc/mtf/pub_key.asc \
  "https://binaries.mtf.exchange/testnet/node/0.9.1/pub_key.asc"

gpg --show-keys --with-fingerprint /etc/mtf/pub_key.asc
```

The fingerprint must be exactly:

```
04781F5109BA16B3CCB43D1E66E38A32E5A25D73
```

Any other fingerprint is the wrong key. Stop and ask the network operators.

`0.9.1` in that URL is a release version, and there is no version-independent
path yet. Read the current version from the manifest and substitute it:

```sh
curl -fsS https://binaries.mtf.exchange/testnet/node/manifest.json \
  | grep -o '"version": *"[^"]*"'
```

The key is the same across releases, so fetch it once. The version only selects
a directory that holds a copy.

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

> **Not live yet.** Roadmap: a detached GPG signature and a reproducible-build
> attestation for the visor binary itself, so its provenance is verifiable
> exactly like the node's.

## Never hand-place a binary

Binaries arrive only through the visor's verified manifest flow. Copying a
binary into place yourself skips **both** layers — no threshold signature, no
content hash, no GPG signature. Do not do it, not even to recover a stuck node.
