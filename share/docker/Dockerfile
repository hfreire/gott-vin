FROM ghost:4.48.8-alpine AS ghost

# Install utils
RUN set -x \
  && apk --no-cache add \
    su-exec \
    curl \
    netcat-openbsd \
    patch

# Patch ghost code
COPY --chown=node:node share/ghost/ghost.patch $GHOST_INSTALL/ghost.patch
RUN set -x \
  && cd $GHOST_INSTALL \
    && su-exec node patch -p1 < ghost.patch \
    && rm ghost.patch \
  && cd $GHOST_INSTALL/current \
    && su-exec node yarn add modern-logger@1.5.53 ghost-storage-adapter-s3@2.8.0 \
    && su-exec node ln -s $GHOST_INSTALL/current/node_modules/ghost-storage-adapter-s3 $GHOST_INSTALL/current/core/server/adapters/storage/s3 \
    && su-exec node yarn cache clean \
    && npm cache clean --force \
    && rm -rv /tmp/yarn* /tmp/v8*

# Replace default ghost assets and configuration
COPY --chown=node:node share/ghost/redirects.json $GHOST_INSTALL/content.orig/data/redirects.json
COPY --chown=node:node share/ghost/routes.yaml $GHOST_INSTALL/content.orig/settings/routes.yaml
COPY --chown=node:node share/ghost/config.production.json $GHOST_INSTALL/config.production.json

FROM ghost AS prebuild

RUN apk --no-cache add git

FROM prebuild AS build
ARG NAME
ARG NPM_TOKEN
USER node

RUN mkdir /home/node/$NAME
WORKDIR /home/node/$NAME

# Install npm packages
COPY --chown=node:node package.json package-lock.json .snyk ./
RUN NODE_ENV= npm ci

# Build app
COPY --chown=node:node src src/
COPY --chown=node:node bower.json gulpfile.js ./
RUN set -x \
  && npm run build --if-present \
  && npm prune --production \
  && npm cache clean --force \
  && rm -rf bower_components && rm -rf node_modules && rm -rf /tmp/phatomjs

FROM ghost
LABEL maintainer="hugo@exec.sh"
ARG NAME
ARG VERSION
ARG VERSION_COMMIT
ARG VERSION_BUILD_DATE

RUN su-exec node mkdir /home/node/$NAME
WORKDIR /home/node/$NAME

# Copy app build
COPY --from=build --chown=node:node /home/node/$NAME /home/node/$NAME
COPY --chown=node:node share/docker/start.sh start.sh
COPY --chown=node:node share/docker/test.sh test.sh

# Link app with ghost installation
RUN set -x \
  && su-exec node ln -s /home/node/$NAME/src $GHOST_INSTALL/current/content/themes/$NAME \
  && su-exec node ln -s $GHOST_INSTALL/current/content/themes/$NAME $GHOST_INSTALL/content.orig/themes/$NAME

# Set app runtime environment variables
ENV NAME $NAME
ENV VERSION $VERSION
ENV VERSION_COMMIT $VERSION_COMMIT
ENV VERSION_BUILD_DATE $VERSION_BUILD_DATE

EXPOSE 3000

ENTRYPOINT [ "./start.sh" ]

HEALTHCHECK --start-period=10s --interval=5m --timeout=3s \
  CMD nc -z localhost 3000 || exit 1
