# ─── Build stage ─────────────────────────────────────────────────────────────
FROM rust:1.78-slim-bookworm AS builder

ARG BUILD_DATE
ARG VCS_REF

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Cache dependencies by copying manifests first
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release 2>/dev/null || true
RUN rm -rf src

# Build the real application
COPY . .
RUN cargo build --release --bin blinks-backend

# ─── Runtime stage ────────────────────────────────────────────────────────────
FROM debian:bookworm-slim AS runtime

ARG BUILD_DATE
ARG VCS_REF

LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.title="blinks-backend" \
      org.opencontainers.image.description="BLINKS payment backend"

# Runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    libssl3 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN useradd -r -s /bin/false -u 1001 blinks

WORKDIR /app

# Copy binary and config
COPY --from=builder /app/target/release/blinks-backend ./blinks-backend
COPY --chown=blinks:blinks config/ ./config/
COPY --chown=blinks:blinks migrations/ ./migrations/

RUN chmod +x ./blinks-backend

USER blinks

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

ENTRYPOINT ["./blinks-backend"]
