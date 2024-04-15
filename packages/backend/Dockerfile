# This dockerfile builds an image for the backend package.
# It should be executed with the root of the repo as docker context.
#
# Before building this image, be sure to have run the following commands in the repo root:
#
# yarn install
# yarn tsc
# yarn build:backend
#
# Once the commands have been run, you can build the image using `yarn build-image`

FROM node:18-bookworm-slim
USER root

RUN sh -c 'mkdir -p /.keystore'
COPY client-identity.p12 /.keystore
# Install isolate-vm dependencies, these are needed by the @backstage/plugin-scaffolder-backend.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip python3-venv g++ build-essential curl ca-certificates && \
    yarn config set python /usr/bin/python3
# Install Docker
ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
RUN pip3 install mkdocs-techdocs-core
RUN curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
# Install sqlite3 dependencies. You can skip this if you don't use sqlite3 in the image,
# in which case you should also move better-sqlite3 to "devDependencies" in package.json.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends git
# From here on we use the least-privileged `node` user to run the backend.
# USER node

# This should create the app dir as `node`.
# If it is instead created as `root` then the `tar` command below will fail: `can't create directory 'packages/': Permission denied`.
# If this occurs, then ensure BuildKit is enabled (`DOCKER_BUILDKIT=1`) so the app dir is correctly created as `node`.
WORKDIR /app

# This switches many Node.js dependencies to production mode.
# ENV NODE_ENV production

# Copy repo skeleton first, to avoid unnecessary docker cache invalidation.
# The skeleton contains the package.json of each package in the monorepo,
# and along with yarn.lock and the root package.json, that's enough to run yarn install.
# COPY --chown=node:node yarn.lock package.json packages/backend/ ./
#RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

# RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
#     yarn install --production --network-timeout 300000

# Then copy the rest of the backend bundle, along with any other files we might want.
# COPY --chown=node:node packages/backend app-config*.yaml ./
COPY package.json yarn.lock ./
RUN yarn add @backstage/cli -W
RUN yarn add @types/better-sqlite3 -W
COPY . . 
# RUN yarn install --network-timeout 300000
RUN find /app/node_modules/ ! -user root | xargs chown root:root || true
EXPOSE 3000/tcp
EXPOSE 7007/tcp
#RUN tar xzf bundle.tar.gz && rm bundle.tar.gz
# Set up Docker daemon
CMD ["dockerd", "--host=unix:///var/run/docker.sock", "--host=tcp://0.0.0.0:2375"]

# RUN yarn config set network-timeout 600000 -g && yarn add @backstage/cli -W 
# RUN yarn build:backend
# CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
ENTRYPOINT yarn dev
