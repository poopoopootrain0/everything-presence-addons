ARG BUILD_FROM=ghcr.io/hassio-addons/base:19.0.0

FROM node:20-alpine AS build
WORKDIR /app
ENV NODE_ENV=development

# Copy workspace package files
COPY package.json package-lock.json ./
COPY backend/package.json backend/package.json
COPY frontend/package.json frontend/package.json

# Install all dependencies (including dev) offline
RUN npm ci --include=dev

# Add node_modules/.bin to PATH for local binaries (tsc, vite)
ENV PATH=/app/node_modules/.bin:$PATH

# Copy sources
COPY backend backend
COPY frontend frontend

# Build backend and frontend
RUN npm run build --workspaces

# Prune devDependencies for production
RUN npm prune --omit=dev --workspaces

########################
# Build addon image
########################
FROM ${BUILD_FROM} AS addon
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
WORKDIR /app
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

# Install node runtime only
RUN apk add --no-cache nodejs npm

# Copy built artifacts and runtime dependencies
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json /app/package-lock.json ./
COPY config ./config
COPY rootfs/ /

RUN rm -rf /etc/services.d/zone-configurator && \
    chmod +x /etc/s6-overlay/s6-rc.d/zone-configurator/run

EXPOSE 42069/tcp 38080/tcp

########################
# Standalone image
########################
FROM node:20-alpine AS standalone
WORKDIR /app
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json /app/package-lock.json ./
COPY config ./config

EXPOSE 42069/tcp 38080/tcp
CMD ["node", "backend/dist/index.js"]

########################
# Default build target
########################
FROM addon AS final
