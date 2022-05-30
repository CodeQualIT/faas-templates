FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/classic-watchdog:0.2.0 as watchdog
FROM --platform=${TARGETPLATFORM:-linux/amd64} python:3-alpine

ARG TARGETPLATFORM
ARG BUILDPLATFORM

# Allows you to add additional packages via build-arg
ARG ADDITIONAL_PACKAGE

# Copy over watchdog
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

# Install additional packages (and git for private git dependencies in requirements.txt)
RUN apk --no-cache add ca-certificates git ${ADDITIONAL_PACKAGE}

# Add non root user
RUN addgroup -S app && adduser app -S -G app

RUN chown -R app /home/app && \
    mkdir -p /home/app/python && chown -R app /home/app

# Setup python env vars
USER app
ENV PATH=$PATH:/home/app/.local/bin:/home/app/python/bin/
ENV PYTHONPATH=$PYTHONPATH:/home/app/python

# Copy over template files and install template requirements
WORKDIR /home/app/
COPY index.py           .
COPY requirements.txt   .
RUN --mount=type=secret,id=GIT_TOKEN GIT_TOKEN=$(cat /run/secrets/GIT_TOKEN) pip install -r requirements.txt --target=/home/app/python

# Mark the function dir as a module
RUN mkdir -p function
RUN touch ./function/__init__.py

# Copy over the function specific requirements file and install these requirements
WORKDIR /home/app/function/
COPY function/requirements.txt	.
RUN --mount=type=secret,id=GIT_TOKEN GIT_TOKEN=$(cat /run/secrets/GIT_TOKEN) pip install -r requirements.txt --target=/home/app/python

# Remove git as it is no longer needed
apk del git

# Copy over the specific function code
WORKDIR /home/app/
USER root
COPY function           function

# Allow any user-id for OpenShift users.
RUN chown -R app:app ./ && \
    chmod -R 777 /home/app/python

# Prepare and run the watchdog
USER app
ENV fprocess="python3 index.py"
EXPOSE 8080
HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1
CMD ["fwatchdog"]
