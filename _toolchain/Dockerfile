###############################################################################
## Base image                                                                ##
###############################################################################

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS base

ARG UID=1000
ARG GID=1000

RUN groupadd -g $GID developer && \
    useradd -u $UID -g $GID -m -s /bin/bash developer

USER developer

###############################################################################
## native builder image base                             ##
###############################################################################

FROM base AS builder

ARG SRC=src
ARG PROJECT=myConsoleApp

RUN mkdir ~/out 

# COPY does not respect the set user, thus we have to set the targets permissions manually
# COPY also does not the home directory of the set USER, thus the explicite path
#
# '~' works with RUN, because the variable is interpreted by a process running inside the target container.
#
# COPY on the other hand is an external command that just copies something from the build context into the image
COPY --chown=developer:developer $SRC/$PROJECT /home/developer/src


###############################################################################
## builder that targets a musl based distribution and build self-contained   ##
###############################################################################

FROM builder AS musl-builder

RUN dotnet publish --self-contained --runtime alpine-x64 ~/src -o ~/out


###############################################################################
## create a minimal product image                                            ##
###############################################################################


FROM alpine AS product


ARG UID=1000
ARG GID=1000

RUN adduser -D -u $UID -g $GID -s /bin/bash opuser &&\
    apk add bash icu-libs krb5-libs libgcc libintl libssl1.1 libstdc++ zlib

COPY --from=musl-builder --chown=opuser:opuser /home/developer/out/ /app/

USER opuser
