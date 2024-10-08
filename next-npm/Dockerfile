# syntax=docker/dockerfile:1

# Comments are provided throughout this file to help you get started.
# If you need more help, visit the Dockerfile reference guide at
# https://docs.docker.com/engine/reference/builder/

ARG PORT=3000
ARG START_COMMAND="npm start"
ARG OUTPUT_DIR=".next"
ARG BUILD_COMMAND="npm run build"

################################################################################
# Use node image for base image for all stages.
FROM chainguard/node:latest as base
# Set working directory for all build stages.
WORKDIR /app

################################################################################
# Create a stage for installing production dependecies.
FROM base as deps

# Download dependencies as a separate step to take advantage of Docker's caching.
# Leverage a cache mount to /root/.npm to speed up subsequent builds.
# Leverage bind mounts to package.json and package-lock.json to avoid having to copy them
# into this layer.
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

################################################################################
# Create a stage for building the application.
FROM deps as build
ARG BUILD_COMMAND
ENV BUILD_COMMAND_ENV=$BUILD_COMMAND

# Download additional development dependencies before building, as some projects require
# "devDependencies" to be installed to build. If you don't need this, remove this step.
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci

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
