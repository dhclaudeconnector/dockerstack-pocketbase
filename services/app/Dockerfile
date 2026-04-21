# ================================================================
#  services/app/Dockerfile
#  PocketBase + Litestream (SQLite → S3 replication)
#
#  Build args:
#    PB_VERSION  — phiên bản PocketBase (default: 0.28.2)
#
#  Startup flow:
#    entrypoint.sh → restore/skip S3 → litestream replicate
#      → pocketbase serve --http=0.0.0.0:${PORT} --dir=/pb/pb_data
#
#  Layer cache strategy:
#    Layer 1 (stable): apk dependencies — chỉ rebuild khi thêm package mới
#    Layer 2 (per-version): PocketBase binary download — rebuild khi PB_VERSION đổi
#    Layer 3 (per-commit): config + entrypoint files
#
#  Với cách tách này, khi chỉ đổi PB_VERSION, Docker không cần
#  chạy lại `apk add` (layer 1 được cache), tiết kiệm ~15–20s build.
# ================================================================

# ── Stage 1: lấy litestream binary ─────────────────────────────
FROM litestream/litestream:0.3.13 AS litestream-bin

# ── Stage 2: PocketBase trên Alpine ────────────────────────────
FROM alpine:3.19

# ── Layer 1: system deps — cache riêng, không phụ thuộc PB_VERSION ──
# Đặt TRƯỚC ARG PB_VERSION để layer này không bị invalidate khi
# PB_VERSION thay đổi.
RUN apk add --no-cache ca-certificates wget unzip

# ── Layer 2: PocketBase download — chỉ rebuild khi PB_VERSION đổi ──
ARG PB_VERSION=0.28.2
RUN wget -q \
       "https://github.com/pocketbase/pocketbase/releases/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_amd64.zip" \
       -O /tmp/pocketbase.zip \
    && unzip /tmp/pocketbase.zip -d /pb \
    && rm /tmp/pocketbase.zip \
    && chmod +x /pb/pocketbase

# ── Layer 3: runtime files — rebuild khi config/entrypoint thay đổi ──
# Copy litestream binary từ stage 1
COPY --from=litestream-bin /usr/local/bin/litestream /usr/local/bin/litestream

# Copy litestream config và entrypoint
COPY litestream.yml /etc/litestream.yml
COPY entrypoint.sh  /entrypoint.sh
RUN chmod +x /entrypoint.sh

WORKDIR /pb

# PocketBase data
VOLUME ["/pb/pb_data"]

EXPOSE 8090

HEALTHCHECK --interval=5s --timeout=5s --start-period=10s --retries=6 \
  CMD wget -qO- http://localhost:8090/api/health || exit 1

ENTRYPOINT ["/entrypoint.sh"]
# CMD được forward qua $@ vào entrypoint → litestream replicate -exec "CMD"
CMD ["./pocketbase", "serve", "--http=0.0.0.0:8090", "--dir=/pb/pb_data"]