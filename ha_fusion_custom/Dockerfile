ARG BUILD_FROM=ghcr.io/hassio-addons/base/amd64:14.3.3
FROM ${BUILD_FROM}

RUN apk add --no-cache nodejs npm

WORKDIR /app

# Copy package.json / lock files first
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the app
COPY . .

# Build & prune
RUN npm run build && npm prune --omit=dev

# Use correct start command
CMD ["npm", "run", "serve"]
