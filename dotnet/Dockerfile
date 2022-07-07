#FROM bitnami/dotnet-sdk:3.1 AS build-env
FROM mcr.microsoft.com/dotnet/core/sdk:2.2.401-alpine AS build-env
#FROM mcr.microsoft.com/dotnet/sdk:3.1.420-alpine3.15 AS build-env
ENV HOME=/app
RUN mkdir -p $HOME
WORKDIR $HOME
ADD . $HOME
# Restore as distinct layers
#RUN dotnet restore
# Build and publish a release
RUN dotnet publish Spider.csproj -c Release -o SpiderApp

# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:2.2.6 AS runtime
#FROM mcr.microsoft.com/dotnet/runtime:3.1-alpine3.15
MAINTAINER DT
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
WORKDIR /app
COPY --from=build-env /app/SpiderApp/ .
RUN chmod -R 777 .
ENTRYPOINT ["dotnet","Spider.dll","-s"]
