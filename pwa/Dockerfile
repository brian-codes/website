#syntax=docker/dockerfile:1.4
# Base stage for dev and build
FROM node:18-alpine as builder_base

# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat

WORKDIR /srv/app

RUN corepack enable && \
	corepack prepare --activate pnpm@latest && \
	pnpm config -g set store-dir /.pnpm-store

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during dev and build.
# ENV NEXT_TELEMETRY_DISABLED 1


# Deps stage, preserve dependencies in cache as long as the lockfile isn't changed
FROM builder_base AS deps

COPY pnpm-lock.yaml ./
RUN pnpm fetch

COPY --link . .
RUN pnpm install --offline --frozen-lock

# Development image
FROM deps as dev

EXPOSE 3000
ENV PORT 3000

CMD ["sh", "-c", "pnpm install --offline --frozen-lock; pnpm dev"]


FROM builder_base AS builder
COPY --link . .
COPY --from=deps --link /srv/app/node_modules ./node_modules

ARG NEXT_ROOT_URL
ENV NEXT_ROOT_URL=$NEXT_ROOT_URL
# RUN echo $NEXT_ROOT_URL 

RUN --mount=type=secret,id=GITHUB_KEY \
	export GITHUB_KEY=$(cat /run/secrets/GITHUB_KEY) ; \
	if [ -z "$GITHUB_KEY" ]; then \
		 echo "Please set the GITHUB_KEY secret" && exit 1 ; \
	fi  \
	&& pnpm run prebuild \
	&& pnpm run build \
	&& pnpm run postbuild

# Production image, copy all the files and run next
FROM node:18-alpine AS prod
WORKDIR /srv/app

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder --link /srv/app/public ./public

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --link --chown=nextjs:nodejs /srv/app/.next/standalone ./
COPY --from=builder --link --chown=nextjs:nodejs /srv/app/public/* ./public/
COPY --from=builder --link --chown=nextjs:nodejs /srv/app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000

CMD ["node", "server.js"]
