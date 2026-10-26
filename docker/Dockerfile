# ================================
# Dependencies Stage
# ================================
FROM node:24-alpine AS deps

RUN corepack enable

WORKDIR /app

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY scripts ./scripts
RUN pnpm install --frozen-lockfile

# Install runtime dependencies in a fresh layer. Pruning dependencies copied
# from another stage can rename directories across overlay filesystem layers.
FROM node:24-alpine AS production-deps

RUN corepack enable
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY scripts ./scripts
RUN CI=true HUSKY=0 pnpm install --prod --frozen-lockfile --ignore-scripts && \
    node scripts/postinstall.js

# ================================
# Build Stage
# ================================
FROM node:24-alpine AS builder

RUN corepack enable

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN pnpm build


# ================================
# Production Stage
# ================================
FROM node:24-alpine AS production

RUN apk add --no-cache dumb-init

RUN addgroup -g 1001 -S nodejs && \
    adduser -S reactrouter -u 1001

WORKDIR /app

COPY --from=production-deps --chown=reactrouter:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=reactrouter:nodejs /app/build ./build
COPY --from=builder --chown=reactrouter:nodejs /app/package.json ./package.json

USER reactrouter

EXPOSE 3000

ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "node_modules/@react-router/serve/bin.js", "./build/server/index.js"]
