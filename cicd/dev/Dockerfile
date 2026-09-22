# syntax=docker/dockerfile:1
#
# WeKnora app image — restricted-network build.
#
# BUILDER_BASE_IMAGE / RUNTIME_BASE_IMAGE (built by cicd/base/Dockerfile,
# see that file's header) already carry every apt package, the Rust
# toolchain's output (the compiled BrowserSkill extension/daemon and the
# anydoc static lib), the DuckDB extensions, and `uv`. This Dockerfile never
# calls apt, git, cargo, or the DuckDB CDN — the only network calls left are
# `go mod download` and the Go compile step, both resolved through Nexus via
# GOPROXY_ARG.
#
# Build:
#   docker build -f cicd/dev/Dockerfile \
#     --build-arg BUILDER_BASE_IMAGE=nexus.internal.example.com/weknora/builder-base:latest \
#     --build-arg RUNTIME_BASE_IMAGE=nexus.internal.example.com/weknora/runtime-base:latest \
#     --build-arg GOPROXY_ARG=https://nexus.internal.example.com/repository/go-proxy \
#     --build-arg VERSION_ARG=$(git describe --tags --always) \
#     --build-arg COMMIT_ID_ARG=$(git rev-parse --short HEAD) \
#     --build-arg BUILD_TIME_ARG=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
#     -t weknora-app:dev .
#
# Run from the repo root — this Dockerfile expects the full WeKnora source
# tree as build context, same as docker/Dockerfile.app.

ARG BUILDER_BASE_IMAGE=nexus.internal.example.com/weknora/builder-base:latest
ARG RUNTIME_BASE_IMAGE=nexus.internal.example.com/weknora/runtime-base:latest

# ---------------------------------------------------------------------------
# Build stage
# ---------------------------------------------------------------------------
FROM ${BUILDER_BASE_IMAGE} AS builder
WORKDIR /app

ARG GOPRIVATE_ARG
ARG GOPROXY_ARG
ARG GOSUMDB_ARG=off
ENV GOPRIVATE=${GOPRIVATE_ARG} \
    GOPROXY=${GOPROXY_ARG} \
    GOSUMDB=${GOSUMDB_ARG}

# go.mod replace-points anydoc at ./third_party/anydoc-go, so that module's
# go.mod must exist before `go mod download`.
COPY go.mod go.sum ./
COPY third_party/anydoc-go/go.mod third_party/anydoc-go/go.mod
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .

# The anydoc crate and DuckDB extensions were already compiled/downloaded
# into the builder-base image (cicd/base/Dockerfile) — drop them into the
# paths the Go build and the runtime image expect, instead of rebuilding
# them from source here.
RUN cp -a /opt/weknora-prebuilt/anydoc-go/. third_party/anydoc-go/ && \
    mkdir -p /root/.duckdb && cp -a /opt/weknora-prebuilt/duckdb/. /root/.duckdb/

RUN --mount=type=cache,target=/go/pkg/mod bash ./scripts/copy-licenses.sh /license-bundle

# Get version and commit info for build injection
ARG VERSION_ARG
ARG COMMIT_ID_ARG
ARG BUILD_TIME_ARG
ARG GO_VERSION_ARG
ENV VERSION=${VERSION_ARG} \
    COMMIT_ID=${COMMIT_ID_ARG} \
    BUILD_TIME=${BUILD_TIME_ARG} \
    GO_VERSION=${GO_VERSION_ARG}

# WITH_ANYDOC=0 skips linking the pre-baked anydoc lib; the engine then
# shows as unavailable on the settings page (see docker/Dockerfile.app).
ARG WITH_ANYDOC=1
RUN --mount=type=cache,target=/go/pkg/mod \
    if [ "$WITH_ANYDOC" = "1" ]; then \
        make build-prod GO_BUILD_TAGS=anydoc; \
    else \
        make build-prod; \
    fi
RUN --mount=type=cache,target=/go/pkg/mod cp -r /go/pkg/mod/github.com/yanyiwu/ /app/yanyiwu/

# ---------------------------------------------------------------------------
# Final stage
# ---------------------------------------------------------------------------
FROM ${RUNTIME_BASE_IMAGE}
WORKDIR /app

# Pairing derives the gateway URL from the user's page origin by default.
ENV BROWSERSKILL_BINARY=/opt/weknora/browserskill/bsk \
    BROWSERSKILL_EXTENSION_PATH=/opt/weknora/browserskill/browser-skill-weknora-0.3.0.zip
COPY --from=builder /opt/weknora-prebuilt/browserskill /opt/weknora/browserskill

# Create data directories and set permissions (appuser already exists in
# runtime-base)
RUN mkdir -p /data/files && \
    chown -R appuser:appuser /app /data/files

# Copy migrate tool from builder stage
COPY --from=builder /go/bin/migrate /usr/local/bin/
COPY --from=builder /app/yanyiwu/ /go/pkg/mod/github.com/yanyiwu/

# Copy the binary from the builder stage
COPY --from=builder /app/config ./config
COPY --from=builder /app/scripts ./scripts
COPY --from=builder /app/migrations ./migrations
COPY --from=builder /app/dataset/samples ./dataset/samples
COPY --from=builder /root/.duckdb /home/appuser/.duckdb
COPY --from=builder /app/WeKnora .
COPY --from=builder /license-bundle/ ./

# Copy and make entrypoint script executable
COPY --from=builder /app/scripts/docker-entrypoint.sh ./scripts/docker-entrypoint.sh
RUN chmod +x ./scripts/*.sh

EXPOSE 8080

ENTRYPOINT ["./scripts/docker-entrypoint.sh"]
CMD ["./WeKnora"]
