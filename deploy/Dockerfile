# Minimal image: pull the static mtf-visor and run it as PID 1. The visor then
# downloads + supervises mtf-node itself.
#
# Build:
#   docker build -f deploy/Dockerfile \
#     --build-arg ARCH=x86_64-unknown-linux-musl \
#     --build-arg VISOR_VERSION=latest -t mtf-visor .
FROM debian:bookworm-slim

ARG ARCH=x86_64-unknown-linux-musl
ARG VISOR_VERSION=latest

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

# NOTE: this fetches over HTTPS but does not yet check the published checksum —
# verify it (see docs/VERIFYING.md) and add a checked download before production.
RUN curl -fsSL -o /usr/local/bin/mtf-visor \
      "https://binaries.mtf.exchange/visor/${VISOR_VERSION}/${ARCH}/mtf-visor" \
    && chmod +x /usr/local/bin/mtf-visor

ENTRYPOINT ["/usr/local/bin/mtf-visor", "run", "--config", "/etc/mtf/visor.toml"]
