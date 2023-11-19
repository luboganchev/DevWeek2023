#See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/runtime:6.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DevWeek2023/DevWeek2023.csproj", "DevWeek2023/"]
RUN dotnet restore "DevWeek2023/DevWeek2023.csproj"
COPY . .
WORKDIR "/src/DevWeek2023"
RUN dotnet build "DevWeek2023.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DevWeek2023.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DevWeek2023.dll"]