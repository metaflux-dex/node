# Verifying binaries

Two layers of trust: the visor verifies every **node** binary automatically; you
verify the **visor** binary once, when you install it.

## Node binary — automatic

The visor never runs an unverified node binary. For every update it:

1. Fetches the release manifest from the configured `manifest_url`.
2. Verifies the manifest's **threshold secp256k1 signature** against the
   `release_pubkeys` you pinned — at least `release_threshold` distinct in-policy
   keys must have signed (≥ 2 on mainnet). High-S signatures are rejected.
3. Checks the **chain binding** — the manifest's `chain_id` and `genesis_hash`
   must match your node — on every poll and after every restart.
4. Refuses any manifest at or below the last accepted sequence or version, so a
   release can never be rolled back under you.
5. Downloads the binary from the URL in the manifest and verifies its **blake3**
   content hash matches the signed value before staging it.
6. Executes the verified file by open descriptor, so nothing can swap it between
   the final hash check and exec.

The download URL is itself part of the signed manifest, so it cannot be
redirected to a different host. There is nothing for you to do here.

## Visor binary — manual, once

You download `mtf-visor` yourself, so verify it before the first run. Each
published binary has a companion `.sha256`:

```sh
ARCH=x86_64-unknown-linux-musl
BASE="https://binaries.mtf.exchange/visor/latest/${ARCH}"

curl -fsSL -o mtf-visor        "${BASE}/mtf-visor"
curl -fsSL -o mtf-visor.sha256 "${BASE}/mtf-visor.sha256"
sha256sum -c mtf-visor.sha256          # must print: mtf-visor: OK

sudo install -m 0755 mtf-visor /usr/local/bin/mtf-visor
```

Pin a specific version by replacing `latest` with the version (e.g. `0.2.0`).

> Roadmap: a detached release signature and a reproducible-build attestation for
> the visor binary itself, so its provenance is verifiable the same way the
> node's is.
