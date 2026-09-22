# syntax=docker/dockerfile:1
#
# WeKnora restricted-network build bases.
#
# docker/Dockerfile.app pulls three things over the open internet that are
# NOT go modules or npm packages, so pointing GOPROXY/npm at Nexus alone
# does not make it buildable on a restricted network:
#   - BrowserSkill: `git clone https://github.com/Tencent/BrowserSkill` +
#     `cargo build` (crates.io)
#   - anydoc: `cargo build` against the crate fetched from rsproxy.cn /
#     static.crates.io
#   - DuckDB extensions (spatial, excel): pulled from DuckDB's own
#     extension CDN via `INSTALL <ext>;`
# plus the Rust toolchain (rustup) and `uv` (astral.sh) installers, and the
# apt packages for both the build and runtime stages.
#
# This file builds ALL of that ONCE, from a machine that has internet (or a
# mirror for each of those: GitHub, crates.io, the DuckDB CDN, an apt
# mirror), and bakes the results into two images. cicd/dev/Dockerfile then
# builds FROM those images and only ever talks to Nexus, for `go mod
# download` and the Go compile step.
#
# Rebuild these two targets only when one of these pins changes:
#   - scripts/browserskill-release.json (BrowserSkill extension/daemon version)
#   - third_party/anydoc-go/Cargo.toml   (pinned anydoc crate version)
#   - go.mod                             (github.com/duckdb/duckdb-go/v2 version)
#   - the apt package lists below
#
# Build from the repo root (these stages need the real source):
#   docker build -f cicd/base/Dockerfile --target builder-base \
#     --build-arg GOPROXY_ARG=https://nexus.internal.example.com/repository/go-proxy \
#     -t nexus.internal.example.com/weknora/builder-base:latest .
#   docker build -f cicd/base/Dockerfile --target runtime-base \
#     -t nexus.internal.example.com/weknora/runtime-base:latest .
#   docker push nexus.internal.example.com/weknora/builder-base:latest
#   docker push nexus.internal.example.com/weknora/runtime-base:latest
#
# Single-platform only as written (matches the host architecture). Building
# for multiple architectures means running this file once per target
# platform, same as upstream does for the browserskill stage via
# $TARGETPLATFORM.

# ---------------------------------------------------------------------------
# vendor-browserskill — needs GitHub + crates.io, not Nexus.
# ---------------------------------------------------------------------------
FROM node:24-bookworm-slim AS vendor-browserskill
WORKDIR /build
RUN apt-get update && \
    apt-get install -y --no-install-recommends git python3 ca-certificates curl build-essential cmake pkg-config && \
    rm -rf /var/lib/apt/lists/*
ENV RUSTUP_HOME=/usr/local/rustup CARGO_HOME=/usr/local/cargo
ENV PATH=/usr/local/cargo/bin:$PATH
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain stable
COPY scripts/build_browserskill.sh scripts/browserskill-release.json ./scripts/
COPY patches/browserskill ./patches/browserskill
RUN bash scripts/build_browserskill.sh /opt/weknora/browserskill

# ---------------------------------------------------------------------------
# vendor-anydoc — needs crates.io/rsproxy.cn, not Nexus.
# ---------------------------------------------------------------------------
FROM golang:1.26-bookworm AS vendor-anydoc
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends build-essential curl && rm -rf /var/lib/apt/lists/*
ENV RUSTUP_HOME=/usr/local/rustup CARGO_HOME=/usr/local/cargo
ENV PATH=/usr/local/cargo/bin:$PATH
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain stable
COPY third_party/anydoc-go third_party/anydoc-go
COPY scripts/build-anydoc-lib.sh scripts/build-anydoc-lib.sh
RUN bash scripts/build-anydoc-lib.sh

# ---------------------------------------------------------------------------
# vendor-duckdb — extensions pulled from DuckDB's CDN, not Nexus. The go
# module fetch here can legitimately go through Nexus (GOPROXY_ARG).
# ---------------------------------------------------------------------------
FROM golang:1.26-bookworm AS vendor-duckdb
WORKDIR /app
ARG GOPROXY_ARG
ARG GOSUMDB_ARG=off
ENV GOPROXY=${GOPROXY_ARG} GOSUMDB=${GOSUMDB_ARG}
COPY go.mod go.sum ./
COPY third_party/anydoc-go/go.mod third_party/anydoc-go/go.mod
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY cmd/download cmd/download
RUN go run cmd/download/duckdb/duckdb.go

# ---------------------------------------------------------------------------
# builder-base — what cicd/dev/Dockerfile's build stage starts FROM.
# Everything above is already sitting under /opt/weknora-prebuilt; the app
# build only has to `go mod download` and compile.
# ---------------------------------------------------------------------------
FROM golang:1.26-bookworm AS builder-base
RUN apt-get update && \
    apt-get install -y --no-install-recommends git build-essential libsqlite3-dev curl && \
    rm -rf /var/lib/apt/lists/*
ARG GOPROXY_ARG
ARG GOSUMDB_ARG=off
ENV GOPROXY=${GOPROXY_ARG} GOSUMDB=${GOSUMDB_ARG}
RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
COPY --from=vendor-browserskill /opt/weknora/browserskill /opt/weknora-prebuilt/browserskill
COPY --from=vendor-anydoc /app/third_party/anydoc-go /opt/weknora-prebuilt/anydoc-go
COPY --from=vendor-duckdb /root/.duckdb /opt/weknora-prebuilt/duckdb

# ---------------------------------------------------------------------------
# runtime-base — what cicd/dev/Dockerfile's final stage starts FROM. All apt
# packages and `uv` are already installed.
# ---------------------------------------------------------------------------
FROM debian:12.12-slim AS runtime-base
RUN useradd -m -s /bin/bash appuser
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates build-essential postgresql-client default-mysql-client \
        tzdata sed curl bash vim wget libsqlite3-0 \
        python3 python3-pip python3-dev libffi-dev libssl-dev \
        nodejs npm gosu ffmpeg && \
    python3 -m pip install --break-system-packages --upgrade pip setuptools wheel && \
    mkdir -p /home/appuser/.local/bin && \
    curl -LsSf https://astral.sh/uv/install.sh | CARGO_HOME=/home/appuser/.cargo UV_INSTALL_DIR=/home/appuser/.local/bin sh && \
    chown -R appuser:appuser /home/appuser && \
    ln -sf /home/appuser/.local/bin/uvx /usr/local/bin/uvx && \
    chmod +x /usr/local/bin/uvx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
