# Minimal image: pull the static visor and run it as PID 1. The visor then
# downloads and supervises the node itself.
#
# Build:
#   docker build -f deploy/Dockerfile \
#     --build-arg ARCH=x86_64-unknown-linux-musl \
#     --build-arg VISOR_VERSION=latest \
#     --build-arg RELEASE_GPG_KEY_URL=<url of the release GPG root key> \
#     -t mtf-visor .
FROM debian:bookworm-slim

ARG ARCH=x86_64-unknown-linux-musl
ARG VISOR_VERSION=latest
# The release GPG root key for this chain. It is served per RELEASE, and there is
# no version-independent path — override this when the pinned version moves. Do
# NOT point it at the unversioned path: that serves the previous chain's root.
ARG RELEASE_GPG_KEY_URL=https://binaries.mtf.exchange/testnet/node/0.9.1/pub_key.asc

# `gnupg` is REQUIRED, not optional. The visor shells out to `gpg` to check each
# staged binary's detached signature. Without it the updater fails at start.
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
    && rm -rf /var/lib/apt/lists/*

# Fetch the visor and its published checksum, then verify before install. The
# build FAILS on a mismatch. Do not remove this check: the visor's whole job is
# verification, so an unverified visor makes every layer below it worthless.
RUN set -eu; \
    cd /tmp; \
    BASE="https://binaries.mtf.exchange/visor/${VISOR_VERSION}/${ARCH}"; \
    curl -fsSL -o mtf-visor        "${BASE}/mtf-visor"; \
    curl -fsSL -o mtf-visor.sha256 "${BASE}/mtf-visor.sha256"; \
    sha256sum -c mtf-visor.sha256; \
    install -m 0755 mtf-visor /usr/local/bin/mtf-visor; \
    rm -f /tmp/mtf-visor /tmp/mtf-visor.sha256

# Fetch the offline GPG root key the visor pins, then gate on its fingerprint.
#
# The fingerprint gate is the real check. An image built on the unversioned
# path's key verifies no release of this chain, and it reports success while
# doing so, so the gate below refuses that key by fingerprint.
RUN set -eu; \
    if [ -z "${RELEASE_GPG_KEY_URL}" ]; then \
      echo "RELEASE_GPG_KEY_URL is empty. Set it to the pub_key.asc under the" >&2; \
      echo "release directory you are installing. Expected fingerprint:" >&2; \
      echo "  04781F5109BA16B3CCB43D1E66E38A32E5A25D73" >&2; \
      echo "Refusing to build. See docs/VERIFYING.md." >&2; \
      exit 1; \
    fi; \
    mkdir -p /etc/mtf; \
    curl -fsSL -o /etc/mtf/pub_key.asc "${RELEASE_GPG_KEY_URL}"; \
    gpg --show-keys --with-colons /etc/mtf/pub_key.asc \
      | grep -q "^fpr:::::::::04781F5109BA16B3CCB43D1E66E38A32E5A25D73:" \
      || { echo "release GPG key fingerprint does not match; refusing to build" >&2; exit 1; }

ENTRYPOINT ["/usr/local/bin/mtf-visor", "run", "--config", "/etc/mtf/visor.toml"]
