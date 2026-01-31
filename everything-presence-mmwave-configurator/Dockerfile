# -----------------------------
# ARG must come before any FROM if used in FROM
# -----------------------------
ARG BUILD_FROM=ghcr.io/hassio-addons/base:19.0.0

# -----------------------------
# Build stage
# -----------------------------
FROM node:20-alpine AS build
WORKDIR /app
ENV NODE_ENV=development

# Copy package files
COPY package.json package-lock.json ./
COPY backend/package.json backend/
COPY frontend/package.json frontend/

# Install all deps including devDeps (requires network once)
RUN npm ci --include=dev

# Copy source code
COPY backend backend
COPY frontend frontend

# Build backend and frontend
RUN npm run build --workspaces

# -----------------------------
# Add-on stage (offline-proof)
# -----------------------------
FROM ${BUILD_FROM} AS addon
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
WORKDIR /app

ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

# Install only Node (production deps already built)
RUN apk add --no-cache nodejs npm

# Copy package.json and production node_modules
COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules

# Copy built backend & frontend
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist

# Copy add-on config and rootfs
COPY config ./config
COPY rootfs/ /

RUN rm -rf /etc/services.d/zone-configurator \
    && chmod +x /etc/s6-overlay/s6-rc.d/zone-configurator/run

EXPOSE 42069/tcp 38080/tcp

# Default command
CMD ["node", "backend/dist/index.js"]

# -----------------------------
# Optional standalone image
# -----------------------------
FROM node:20-alpine AS standalone
WORKDIR /app

ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY config ./config

EXPOSE 42069/tcp 38080/tcp
CMD ["node", "backend/dist/index.js"]

# Default build target for Home Assistant add-on
FROM addon AS final
