# syntax=docker/dockerfile:1
#
# The web app as a plain Node container (`next build` standalone output),
# for deployments that run the control plane in docker-compose.yml rather
# than on Cloudflare and do not host the web app on Vercel.
#
# Build from the repository root so packages/shared is in the context:
#
#   docker build -f packages/web/Dockerfile \
#     --build-arg NEXT_PUBLIC_WS_URL=wss://control-plane.example.com \
#     -t open-inspect-web .
#
# NEXT_PUBLIC_* values are inlined into the client bundle at build time, so
# an image is built for one control-plane hostname. They are not secrets.
#
# Runtime env (read on the server, never inlined):
#   CONTROL_PLANE_URL    how the server reaches the control plane; may be an
#                        internal address such as http://app:8787
#   SERVICE_AUTH_SECRET  equals the control plane's SERVICE_AUTH_SECRET_WEB
#   LOG_LEVEL            optional

FROM node:24-slim AS build
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1

# The whole repository, as upstream CI builds it (`npm ci` then the two
# builds): `next build` type-checks every file in packages/web, tests
# included, and those import root devDependencies (eslint) and files from
# other packages (sandbox-runtime, scripts/). Only the standalone output
# reaches the runtime stage, so the build stage's size does not matter.
# --ignore-scripts as in packages/control-plane/Dockerfile: the root `prepare`
# script installs git hooks (husky), which a build has no use for.
COPY package.json package-lock.json ./
COPY packages/ packages/
COPY scripts/ scripts/
RUN --mount=type=cache,target=/root/.npm \
    npm ci --ignore-scripts

ARG NEXT_PUBLIC_WS_URL
ARG NEXT_PUBLIC_SANDBOX_PROVIDER=modal
ARG NEXT_PUBLIC_SCM_PROVIDER
ARG NEXT_PUBLIC_APP_NAME
ARG NEXT_PUBLIC_APP_ICON_URL
RUN test -n "$NEXT_PUBLIC_WS_URL" || { echo "NEXT_PUBLIC_WS_URL build arg is required" >&2; exit 1; }
ENV NEXT_PUBLIC_WS_URL=$NEXT_PUBLIC_WS_URL \
    NEXT_PUBLIC_SANDBOX_PROVIDER=$NEXT_PUBLIC_SANDBOX_PROVIDER \
    NEXT_PUBLIC_SCM_PROVIDER=$NEXT_PUBLIC_SCM_PROVIDER \
    NEXT_PUBLIC_APP_NAME=$NEXT_PUBLIC_APP_NAME \
    NEXT_PUBLIC_APP_ICON_URL=$NEXT_PUBLIC_APP_ICON_URL

RUN npm run build -w @open-inspect/shared \
    && npm run build -w @open-inspect/web

FROM node:24-slim
ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    HOSTNAME=0.0.0.0 \
    PORT=3000
WORKDIR /app

# outputFileTracingRoot is the monorepo root, so the standalone tree mirrors
# it and the server entry point lands at packages/web/server.js. Static
# assets and public/ are not part of the standalone output.
COPY --from=build --chown=node:node /app/packages/web/.next/standalone ./
COPY --from=build --chown=node:node /app/packages/web/.next/static ./packages/web/.next/static
COPY --from=build --chown=node:node /app/packages/web/public ./packages/web/public

USER node
EXPOSE 3000
HEALTHCHECK --interval=15s --timeout=3s --start-period=20s --retries=3 \
    CMD ["node", "-e", "fetch(`http://127.0.0.1:${process.env.PORT}/`, { redirect: 'manual' }).then((r) => process.exit(r.status < 500 ? 0 : 1), () => process.exit(1))"]
CMD ["node", "packages/web/server.js"]
