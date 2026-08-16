# --- Stage 1: Base ---
FROM node:alpine AS base

RUN addgroup -S api-app-group && adduser -S -G api-app-group api-app-user
# # Debian-style user creation
# RUN groupadd -r api-app-group && useradd -r -g api-app-group api-app-user
WORKDIR /app
# Pre-set ownership of the workdir
RUN chown api-app-user:api-app-group /app
COPY --chown=api-app-user:api-app-group package*.json ./

# --- Stage 2: Development ---
FROM base AS development
RUN npm install --legacy-peer-deps
COPY --chown=api-app-user:api-app-group . .
# Create logs dir for dev environment
RUN mkdir -p /app/logs && chown api-app-user:api-app-group /app/logs
USER api-app-user
EXPOSE 9879
CMD ["npm", "run", "dev"]

# --- Stage 3: Build (Intermediate) ---
FROM development AS builder
# RUN npm run docker:pre-run
USER root
RUN npm run build
RUN npm prune --omit=dev --legacy-peer-deps

# --- Stage 4: Production ---
FROM base AS production
ENV NODE_ENV=production

# Copy artifacts from builder
COPY --from=builder --chown=api-app-user:api-app-group /app/node_modules ./node_modules
COPY --from=builder --chown=api-app-user:api-app-group /app/dist ./dist

# CRITICAL: Re-create and permission the logs directory in the final image
RUN mkdir -p /app/logs && chown api-app-user:api-app-group /app/logs

USER api-app-user
EXPOSE 9879

# Using 'node' directly is more memory-efficient than 'npm start'
CMD ["npm", "run","start"]