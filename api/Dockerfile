# Builds nocturne-api from the latest upstream source.
#
# The upstream repo does not ship a Dockerfile for the API — it uses
# "dotnet publish --container" in CI (PublishContainer MSBuild target).
# This Dockerfile replicates that process in a standard multi-stage build.
#
# Node.js is installed in the SDK stage because the .csproj runs NSwag and
# Zod/remote-function codegen steps (via MSBuild targets) that invoke node
# during the publish. These generate frontend-side artefacts that are not
# included in the final image, but the build fails without the runtime.
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

RUN apt-get update \
    && apt-get install -y --no-install-recommends git nodejs npm \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth=1 --branch main \
        https://github.com/nightscout/nocturne.git /src

WORKDIR /src

RUN dotnet publish src/API/Nocturne.API/Nocturne.API.csproj \
        -c Release \
        -o /publish \
        --no-self-contained

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "Nocturne.API.dll"]
