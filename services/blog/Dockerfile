ARG RUST_IMAGE_TAG=1-slim-trixie
ARG DEBIAN_IMAGE_TAG=trixie-slim

############################################################
# Base image with cargo-chef, to cache the build of the dependencies
FROM rust:${RUST_IMAGE_TAG} AS chef

# jemalloc needs a C toolchain to be built
RUN apt-get update && \
  apt-get install -y --no-install-recommends build-essential && \
  rm -rf /var/lib/apt/lists/*

RUN cargo install cargo-chef --locked

WORKDIR /build

############################################################
# Compute the recipe of the dependencies to build
FROM chef AS planner

COPY . .
RUN cargo chef prepare --recipe-path recipe.json

############################################################
# Builder
FROM chef AS builder

# Build only the dependencies. This layer is cached as long as the recipe, i.e Cargo.toml/Cargo.lock, does not change
COPY --from=planner /build/recipe.json recipe.json
RUN cargo chef cook --release --locked --recipe-path recipe.json

COPY . .
RUN cargo build --release --locked

############################################################
# GeoIP databases, to log where the ip doing a request is and who it belongs to.
# DB-IP's lite ones are CC-BY-4.0 and need no account, contrary to MaxMind's GeoLite2 which wants a license key.
FROM debian:${DEBIAN_IMAGE_TAG} AS geoip

# DB-IP publishes a new file every month under its own name and keeps the previous ones, so this is
# pinned and bumped by hand. An unbumped build serves stale locations, it does not fail.
ARG DBIP_MONTH=2026-09

RUN apt-get update && \
  apt-get install -y --no-install-recommends ca-certificates curl && \
  rm -rf /var/lib/apt/lists/*

# Downloaded then gunzipped in two steps, as sh has no pipefail: `curl | gunzip` would swallow a 404
RUN mkdir /geoip && \
  curl -fsS -o /geoip/city.mmdb.gz "https://download.db-ip.com/free/dbip-city-lite-${DBIP_MONTH}.mmdb.gz" && \
  curl -fsS -o /geoip/asn.mmdb.gz "https://download.db-ip.com/free/dbip-asn-lite-${DBIP_MONTH}.mmdb.gz" && \
  gunzip /geoip/city.mmdb.gz /geoip/asn.mmdb.gz

############################################################
# Final image
FROM debian:${DEBIAN_IMAGE_TAG}

RUN apt-get update && \
  apt-get -y upgrade && \
  rm -rf /var/lib/apt/lists/* && \
  useradd -ms /bin/bash app

COPY --from=builder /build/target/release/blog /usr/local/bin/blog

# One directory per hostname to serve, i.e: public/wstunnel is served for wstunnel.erebe.eu
# --chown is required, as COPY keeps the permissions of the files, which may not be readable by the app user
COPY --chown=app:app public/ /app/public/

# Left root-owned and read-only: the app mmaps them, which is undefined behaviour if anything rewrites them
COPY --from=geoip /geoip/ /app/geoip/

USER app
WORKDIR /app
ENV ROOT_DIR=/app/public
ENV HTTP_LISTEN="[::]:8080"
ENV GEOIP_CITY_DB=/app/geoip/city.mmdb
ENV GEOIP_ASN_DB=/app/geoip/asn.mmdb
EXPOSE 8080

ENTRYPOINT ["/usr/local/bin/blog"]
