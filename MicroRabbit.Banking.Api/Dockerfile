# Use a imagem base do .NET 9.0
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 8080

# Use a imagem SDK para build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copiar arquivos de projeto e restaurar dependências
COPY ["MicroRabbit.Banking.Api/MicroRabbit.Banking.Api.csproj", "MicroRabbit.Banking.Api/"]
COPY ["MicroRabbit.Banking.Application/MicroRabbit.Banking.Application.csproj", "MicroRabbit.Banking.Application/"]
COPY ["MicroRabbit.Banking.Domain/MicroRabbit.Banking.Domain.csproj", "MicroRabbit.Banking.Domain/"]
COPY ["MicroRabbit.Banking.Data/MicroRabbit.Banking.Data.csproj", "MicroRabbit.Banking.Data/"]
COPY ["MicroRabbit.Domain.Core/MicroRabbit.Domain.Core.csproj", "MicroRabbit.Domain.Core/"]
COPY ["MicroRabbit.Infra.Bus/MicroRabbit.Infra.Bus.csproj", "MicroRabbit.Infra.Bus/"]
COPY ["MicroRabbit.IoC/MicroRabbit.IoC.csproj", "MicroRabbit.IoC/"]

# Restaurar dependências
RUN dotnet restore "MicroRabbit.Banking.Api/MicroRabbit.Banking.Api.csproj"

# Copiar todo o código fonte
COPY . .

# Build da aplicação
WORKDIR "/src/MicroRabbit.Banking.Api"
RUN dotnet build "MicroRabbit.Banking.Api.csproj" -c Release -o /app/build

# Publicar a aplicação
FROM build AS publish
RUN dotnet publish "MicroRabbit.Banking.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Imagem final
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Configurar variáveis de ambiente
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "MicroRabbit.Banking.Api.dll"]