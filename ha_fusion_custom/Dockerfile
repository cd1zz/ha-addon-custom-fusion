ARG BUILD_FROM=ghcr.io/hassio-addons/base/amd64:14.3.3
FROM ${BUILD_FROM}

# Install node
RUN \
  apk add --no-cache \
    nodejs \
    npm

WORKDIR /app

# Copy files to /app
COPY . .

# Install, build and prune
RUN npm install --verbose && \
  npm run build && \
  npm prune --omit=dev

# Set environment
ENV PORT=8080 \
  NODE_ENV=production \
  ADDON=true

EXPOSE 8080

CMD ["node", "server.js"]