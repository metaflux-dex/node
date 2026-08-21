# Minimal image: pull the static visor and run it as PID 1. The visor then
# downloads and supervises the node itself.
#
# Build:
#   docker build -f deploy/Dockerfile \
#     --build-arg ARCH=x86_64-unknown-linux-musl \
#     --build-arg VISOR_VERSION=latest -t mtf-visor .
FROM debian:bookworm-slim

ARG ARCH=x86_64-unknown-linux-musl
ARG VISOR_VERSION=latest

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

# Fetch the offline GPG root key the visor pins. Verify its fingerprint against
# docs/VERIFYING.md before you trust an image built this way.
RUN mkdir -p /etc/mtf \
 && curl -fsSL -o /etc/mtf/pub_key.asc "https://binaries.mtf.exchange/testnet/pub_key.asc" \
 && gpg --show-keys --with-colons /etc/mtf/pub_key.asc \
    | grep -q "^fpr:::::::::5AF6597573B2E475B0C646BAD8E6D0B3D187F583:" \
 || (echo "release GPG key fingerprint does not match; refusing to build" >&2; exit 1)

ENTRYPOINT ["/usr/local/bin/mtf-visor", "run", "--config", "/etc/mtf/visor.toml"]
