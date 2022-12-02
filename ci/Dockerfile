FROM alpine:3.17

LABEL maintainer="jakub@status.im"
LABEL source="https://github.com/vacp2p/rfc"
LABEL description="rfc.vac.dev build image"

RUN apk add --no-cache \
    openssh-client-common \
    openssh-client-default \
    git hugo py3-pip

RUN pip install ghp-import

# Jenkins user needs a specific UID/GID to work.
RUN addgroup -g 1001 jenkins \
 && adduser -D -u 1001 -G jenkins jenkins
