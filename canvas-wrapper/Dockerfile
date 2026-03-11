# syntax=docker/dockerfile:1.7

FROM alpine/git:2.47.2 AS source
ARG CANVAS_WRAPPER_REF=main
WORKDIR /src
RUN git clone --depth 1 --branch "${CANVAS_WRAPPER_REF}" https://github.com/j0taaa/canvas-wrapper.git .

FROM oven/bun:1.2.22 AS deps
WORKDIR /app
COPY --from=source /src/package.json /src/bun.lock ./
RUN bun install --frozen-lockfile

FROM node:22-bookworm-slim AS builder
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=deps /app/node_modules ./node_modules
COPY --from=source /src ./
RUN ./node_modules/.bin/next build

FROM node:22-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME=0.0.0.0

RUN groupadd --system nodejs && useradd --system --gid nodejs nextjs

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000

CMD ["node", "server.js"]
