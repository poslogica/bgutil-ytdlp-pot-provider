# server_deno's runtime base. SonicSort passes sonicsort-deno:local -- deno
# compiled from poslogica's deno fork on its own Debian base (same layout as
# denoland/deno:debian: /usr/bin/deno, a deno user, tini); the default is the
# public image.
ARG DENO_BASE=denoland/deno:debian-2.9.7

FROM node:26-bookworm-slim AS install
USER node
WORKDIR /app

COPY package.json package-lock.json ./

RUN --mount=type=cache,target=/home/node/.npm,uid=1000,gid=1000 \
    npm ci --omit=dev --no-audit --no-fund

FROM install AS build_node

RUN --mount=type=cache,target=/home/node/.npm,uid=1000,gid=1000 \
    npm ci --no-audit --no-fund

COPY types/ types/
COPY tsconfig.json ./
COPY src/ src/
RUN npx tsc

FROM install AS server_node

COPY --from=build_node /app/build /app/build

# npm is unused at runtime (entrypoint only invokes node) and its bundled
# dependencies otherwise ship known-vulnerable versions of tar/undici/etc.
USER root
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx
USER node

EXPOSE 4416
ENTRYPOINT ["/usr/local/bin/node", "build/main.js"]
CMD ["--host", "0.0.0.0"]

FROM ${DENO_BASE} AS server_deno

# The deno base image's Debian layer lags behind published security fixes
# (glibc/openssl/perl/pcre2/sqlite3/util-linux had fixes sitting unapplied
# in debian-2.9.5). OS_PATCH_WEEK is a weekly cache-buster passed by
# SonicSort's build scripts: without it BuildKit would cache this layer
# forever and stop picking up new fixes.
ARG OS_PATCH_WEEK=unset
USER root
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*
USER deno
WORKDIR /app
ENV DENO_DIR=/app/.cache/deno
ENV DENO_NO_PROMPT=1 DENO_NO_UPDATE_CHECK=1

COPY types/ types/
COPY tsconfig.json package.json deno.lock ./
COPY --from=install --chown=deno:deno /app/node_modules node_modules/
COPY src/ src/

RUN deno cache --frozen src/main.ts

EXPOSE 4416
ENTRYPOINT [ \
    "/usr/bin/deno", "run", "--allow-env", "--allow-net", \
    "--allow-ffi=/app/node_modules", "--allow-read=/app/node_modules", \
    "/app/src/main.ts"]
CMD ["--host", "0.0.0.0"]
