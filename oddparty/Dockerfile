FROM node:22-bookworm-slim AS build

WORKDIR /src

# better-sqlite3 builds native bindings and requires node-gyp toolchain.
RUN apt-get update \
  && apt-get install -y --no-install-recommends python3 make g++ git ca-certificates \
  && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 https://github.com/j0taaa/oddparty.git .
RUN npm install
RUN mkdir -p data
RUN npm run build

FROM node:22-bookworm-slim

WORKDIR /app
ENV NODE_ENV=production

COPY --from=build /src /app

EXPOSE 3000
CMD ["npm", "run", "start"]
