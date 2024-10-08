# syntax=docker/dockerfile:1

# Comments are provided throughout this file to help you get started.
# If you need more help, visit the Dockerfile reference guide at
# https://docs.docker.com/engine/reference/builder/

ARG PORT=3000
ARG START_COMMAND="pnpm start"
ARG BUILD_COMMAND="pnpm run build"
ARG OUTPUT_DIR=".next"

################################################################################
# Use node image for base image for all stages.
FROM chainguard/node:latest-dev as base

# Install pnpm
USER root
RUN apk add pnpm

# Set user to non-root node
USER node
# Set working directory for all build stages.
WORKDIR /app

################################################################################
# Create a stage for installing production dependecies.
FROM base as deps

# Download dependencies as a separate step to take advantage of Docker's caching.
# Leverage a cache mount to /root/.local/share/pnpm/store to speed up subsequent builds.
# Leverage bind mounts to package.json and pnpm-lock.yaml to avoid having to copy them
# into this layer.
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=pnpm-lock.yaml,target=pnpm-lock.yaml \
    --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --prod --frozen-lockfile

################################################################################
# Create a stage for building the application.
FROM deps as build
ARG BUILD_COMMAND
ENV BUILD_COMMAND_ENV=$BUILD_COMMAND

# Download additional development dependencies before building, as some projects require
# "devDependencies" to be installed to build. If you don't need this, remove this step.
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=pnpm-lock.yaml,target=pnpm-lock.yaml \
    --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

# Copy the rest of the source files into the image.
COPY . .
# Run the build script.
RUN $BUILD_COMMAND_ENV

################################################################################
# Create a new stage to run the application with minimal runtime dependencies
# where the necessary files are copied from the build stage.
FROM base as final
ARG PORT
ARG START_COMMAND
ARG OUTPUT_DIR

ENV START_COMMAND_ENV=$START_COMMAND
# Use production node environment by default.
ENV NODE_ENV production

# Copy package.json so that package manager commands can be used.
COPY package.json .

# Copy the production dependencies from the deps stage and also
# the built application from the build stage into the image.
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/${OUTPUT_DIR} ./${OUTPUT_DIR}

# Expose the port that the application listens on.
EXPOSE ${PORT}

# Run the application.
ENTRYPOINT $START_COMMAND_ENV
