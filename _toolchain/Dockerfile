FROM mcr.microsoft.com/dotnet/sdk

RUN groupadd -g 1000 developer && \
    useradd -r -u 1000 -g developer developer
    
USER developer