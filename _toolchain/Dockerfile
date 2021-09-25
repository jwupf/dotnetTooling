FROM mcr.microsoft.com/dotnet/sdk

ARG UID=1000
ARG GID=1000

RUN groupadd -g $GID developer && \
    useradd -u $UID -g $GID -m -s /bin/bash developer

USER developer