ARG BUILD_FROM=ghcr.io/hassio-addons/base:19.0.0

# -------------------------
# Build Stage
# -------------------------
FROM node:20-alpine AS build
WORKDIR /app
ENV NODE_ENV=development

# Copy root & workspace package files
COPY package.json package-lock.json ./
COPY backend/package.json backend/package.json
COPY frontend/package.json frontend/package.json

# Install root dependencies (hoisted)
RUN npm ci --include=dev

# Copy and install backend
WORKDIR /app/backend
COPY backend/package.json ./package.json
RUN npm ci --include=dev

# Copy and install frontend
WORKDIR /app/frontend
COPY frontend/package.json ./package.json
RUN npm ci --include=dev

# Copy sources
WORKDIR /app
COPY backend ./backend
COPY frontend ./frontend

# Add local binaries to PATH
ENV PATH=/app/node_modules/.bin:/app/backend/node_modules/.bin:/app/frontend/node_modules/.bin:$PATH

# Build all workspaces
RUN npm run build --workspaces

# -------------------------
# Addon Stage
# -------------------------
FROM ${BUILD_FROM} AS addon
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist
WORKDIR /app

RUN apk add --no-cache nodejs npm

COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/backend/package.json ./backend/package.json
COPY --from=build /app/backend/node_modules ./backend/node_modules
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY config ./config
COPY rootfs/ /

RUN rm -rf /etc/services.d/zone-configurator && \
    chmod +x /etc/s6-overlay/s6-rc.d/zone-configurator/run

EXPOSE 42069/tcp 38080/tcp

# -------------------------
# Standalone Stage
# -------------------------
FROM node:20-alpine AS standalone
WORKDIR /app
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/backend/package.json ./backend/package.json
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY config ./config

EXPOSE 42069/tcp 38080/tcp
CMD ["node", "backend/dist/index.js"]

# -------------------------
# Default Home Assistant Add-on build target
# -------------------------
FROM addon AS final
