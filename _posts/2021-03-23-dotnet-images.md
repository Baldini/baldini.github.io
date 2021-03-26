---
title: Diminuindo suas imagens Docker .NET Core
tags: Docker .NET dotnet Core
key: dotnet-images
---

Já passou o tempo em que aplicações .NET eram grandes e precisavam de 1Gb de dependência, hoje com poucos megas você consegue rodar uma aplicação .NET core com todas as dependências, e no docker ficando abaixo dos 100Mb, com pequenas modificações e consumindo a mesma quantidade de memoria e CPU. Para isso vamos destrinchar um pouco as imagens docker disponibilizadas pela microsoft, publish e fazer pequenos testes.

## Setup

Para exemplificar criei uma aplicação em ASP.NET Core 5.0, uma WEB API, e não modifiquei em nada, além disso criei um teste com [Artillery](artillery.io) fazendo algumas chamadas para vermos a memória utilizada, subi a imagem no docker local e executei os testes.

Esse teste foi executado na minha própria máquina, que não é o melhor ambiente para um benchmark completo, então é muito mais para curiosidade e pequenas diferenças devem ser desconsideradas.

### Estrutura

- Verificação do tamanho da imagem
- Verificação da memória utilizada sem nenhuma chamada
- Verificação da memória máxima com o script de teste

### Arquivo de teste

A ideia foi fazer 10 chamadas por 10 segundos, para Warmup, e logo em seguida aumentar até 30 durante 40 segundos, gerando aproximadamente 900 chamadas, segue o arquivo do Artillery utilizado:

{% highlight yml %}
config:
  target: 'http://localhost:3222'
  phases:
    - duration: 10
      arrivalRate: 10
      name: "Warmup"
    - duration: 40
      arrivalRate: 10
      rampTo: 30
scenarios:
  - flow:
    - get:
        url: "/WeatherForecast"
{% endhighlight %}

## Multi-stage build

Nos dockerfiles de exemplo usamos [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/), onde temos uma imagem base que será utilizada, por fim fazemos o build dentro do próprio docker e repassamos os arquivos publicados para a imagem base, fazendo com que fique mais simples de entender o que está acontecendo, como a aplicação está sendo buildada ao invés de fazer o build local e copiar para a imagem docker.

## Dockerfile padrão

{% highlight dockerfile %}
FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0-buster-slim AS build
WORKDIR /src
COPY ["DockerImages/DockerImages.csproj", "DockerImages/"]
RUN dotnet restore "DockerImages/DockerImages.csproj"
COPY . .
WORKDIR "/src/DockerImages"
RUN dotnet build "DockerImages.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DockerImages.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DockerImages.dll"]
{% endhighlight %}

Nesse dockerfile estamos usando a imagem padrão do asp na versão 5.0 com buster-slim, ela é baseada no Debian e tem o ASP.NET Core e os runtimes do .NET e é otimizada para rodar aplicações ASP.NET.

Esta é a imagem padrão quando adicionamos suporte a docker pelo Visual Studio.

### Resultados

- Size: 205mb
- Memory Idle: 35mb
- Memory Max: 60mb

## Runtime Deps alpine

{% highlight dockerfile %}
FROM mcr.microsoft.com/dotnet/runtime-deps:5.0-alpine AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0-buster-slim AS build
WORKDIR /src
COPY ["DockerImages/DockerImages.csproj", "DockerImages/"]
RUN dotnet restore "DockerImages/DockerImages.csproj"
COPY . .
WORKDIR "/src/DockerImages"
RUN dotnet build "DockerImages.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DockerImages.csproj" -o /app/publish -r linux-musl-x64 --self-contained

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["./DockerImages"]
{% endhighlight %}

Aqui temos algumas modificações, mudamos a imagem para a runtime-deps:alpine, que é baseada no Alpine e contém as dependências necessárias para rodar o .NET, ela é ideal para rodar aplicações self-contained, então alteramos o comando de publicação para gerar uma aplicação self-contained e para o runtime do linux.

### Resultados

- Size: 104mb
- Memory Idle: 32mb
- Memory Max: 54mb

Com essas modificações conseguimos diminuir o tamanho total da imagem para 104mb, sem modificações no uso de memória.

## Trimmed

{% highlight dockerfile %}
FROM mcr.microsoft.com/dotnet/runtime-deps:5.0-alpine AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0-alpine AS build
WORKDIR /src
COPY ["DockerImages/DockerImages.csproj", "DockerImages/"]
RUN dotnet restore "DockerImages/DockerImages.csproj"
COPY . .
WORKDIR "/src/DockerImages"
RUN dotnet build "DockerImages.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DockerImages.csproj" -r linux-musl-x64 -o /app/publish /p:PublishTrimmed=true --no-restore --self-contained

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["./DockerImages"]
{% endhighlight %}

Podemos adicionar no dotnet publish para remover as bibliotecas não usadas e deminuindo ainda mais o tamanho da nossa imagem.

### Resultados

- Size: 61.6mb
- Memory Idle: 36mb
- Memory Max: 59mb

Agora temos uma imagem ainda menor, sem comprometer a memória consumida.

## Single file

{% highlight dockerfile %}
FROM mcr.microsoft.com/dotnet/runtime-deps:5.0-alpine AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0-alpine AS build
WORKDIR /src
COPY ["DockerImages/DockerImages.csproj", "DockerImages/"]
RUN dotnet restore "DockerImages/DockerImages.csproj"
COPY . .
WORKDIR "/src/DockerImages"
RUN dotnet build "DockerImages.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DockerImages.csproj" -r linux-musl-x64 -o /app/publish /p:PublishTrimmed=true /p:PublishSingleFile=true --no-restore --self-contained

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["./DockerImages"]
{% endhighlight %}

Podemos diminuir ainda mais a imagem colocando a opção no publish de SingleFile, que faz com que todos os arquivos e dependências fiquem dentro de um único arquivo.

### Resultados

- Size: 50.2mb
- Memory Idle: 36mb
- Memory Max: 59mb

Agora temos uma imagem ainda menor, sem comprometer a memória consumida.

## Resultados finais

| Image  | Size  | Memory Idle  | Memory Max |
|---|---|---|---|
|Base|205Mb|35Mb|60Mb|
|Runtime Deps|104Mb|32mb|54mb|
|Trimmed|61.6Mb|36Mb|59mb|
|Single file|50.2Mb|36Mb|59mb|

Você pode explorar mais opções de [Publish](https://docs.microsoft.com/pt-br/dotnet/core/tools/dotnet-publish) ou versões diferentes das imagens [Docker](https://github.com/dotnet/dotnet-docker).

Com isso podemos ter uma imagem com nossa aplicação, com 60mb de tamanho, consumindo 35mb de ram, e com mais segurança com imagens alpine e com um dos melhores desempenhos. :)