ARG BUILD_FROM=ghcr.io/hassio-addons/base:19.0.0

########################################
# Build stage (offline-proof)
########################################
FROM node:20-alpine AS build
WORKDIR /app
ENV NODE_ENV=development

# Copy package files and prebuilt node_modules
COPY package.json package-lock.json ./
COPY backend/package.json backend/package.json
COPY frontend/package.json frontend/package.json
COPY node_modules ./node_modules   # offline-proof

# Copy source code
COPY backend backend
COPY frontend frontend

# Build all workspaces (tsc & vite from hoisted node_modules)
ENV PATH=/app/node_modules/.bin:$PATH
RUN npm run build --workspaces

# Remove devDependencies to shrink image
RUN npm prune --omit=dev --workspaces

########################################
# Home Assistant add-on image
########################################
FROM ${BUILD_FROM} AS addon
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist
WORKDIR /app

# Add Node (runtime only)
RUN apk add --no-cache nodejs npm

# Copy built app
COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/backend/package.json ./backend/package.json
COPY --from=build /app/backend/node_modules ./backend/node_modules
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY config ./config
COPY rootfs/ /

# Make sure s6 service is executable
RUN rm -rf /etc/services.d/zone-configurator && \
    chmod +x /etc/s6-overlay/s6-rc.d/zone-configurator/run

EXPOSE 42069/tcp 38080/tcp

########################################
# Standalone node image
########################################
FROM node:20-alpine AS standalone
WORKDIR /app
ENV NODE_ENV=production \
    PORT=42069 \
    FIRMWARE_LAN_PORT=38080 \
    FRONTEND_DIST=/app/frontend/dist

# Copy built app
COPY --from=build /app/package.json /app/package-lock.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/backend/package.json ./backend/package.json
COPY --from=build /app/backend/dist ./backend/dist
COPY --from=build /app/frontend/dist ./frontend/dist
COPY config ./config

EXPOSE 42069/tcp 38080/tcp
CMD ["node", "backend/dist/index.js"]

########################################
# Default build target for Home Assistant add-on
########################################
FROM addon AS final
