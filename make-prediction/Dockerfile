FROM oven/bun:1 AS build

WORKDIR /src

RUN apt-get update \
  && apt-get install -y --no-install-recommends git ca-certificates \
  && rm -rf /var/lib/apt/lists/*
RUN git clone --depth 1 https://github.com/j0taaa/make-prediction.git .
RUN bun install --frozen-lockfile
RUN bun run build

FROM oven/bun:1

WORKDIR /app
ENV NODE_ENV=production

COPY --from=build /src /app

EXPOSE 3000
CMD ["bun", "run", "start", "--hostname", "0.0.0.0", "--port", "3000"]
