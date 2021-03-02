---
title: Usando docker-compose pra facilitar seu desenvolvimento local
tags: docker docker-compose visualstudio
key: docker-compose-vs
---

Acredito que todo mundo já encontrou um projeto ou entrou em uma empresa, em que existiam tantas dependências para rodar o sistema, que você perde um dia inteiro só de instalação, fora a configuração, banco relacional, banco não relacional, cache, serviços, etc.

E, às vezes, você precisa fazer somente uma modificação, em parte do sistema, mas precisa de um documento gigante no confluence pra ver como fazer toda a configuração. Porém podemos simplificar isso em nossos projetos, usando docker.

Ai você pode pensar: "Ah, Guilherme, mas aqui na empresa nunca vão subir algo assim pra produção". Mas eu nunca disse para usar em produção e sim pra facilitar a sua vida localmente.

## Docker

Mas primeiro, o que é [Docker](https://www.docker.com/)?

Docker é um sistema gerenciador de containers, que é parecido com virtualização, mas somente no conceito.

### Containers

Diferente de uma máquina virtual, containers são, basicamente, processos. Enquanto máquinas virtuais tem todo um sistema operacional por trás, containers não necessitam disso e contém o mínimo necessário para rodar a aplicação, que normalmente é única, enquanto em máquinas virtuais existem várias aplicações rodando. Além disso, containers utilizam recursos do Kernel local, diferente de máquinas virtuais.

Dica: Caso vá rodar docker no windows, dê preferência por rodar ele em ambiente [WSL2](https://docs.microsoft.com/pt-br/windows/wsl/install-win10) a performance é bem melhor.

### Subindo nosso primeiro container

Bom, vamos subir um container básico, depois de já instalado o docker em sua máquina, podemos executar o comando de "Hello World" para vermos nosso primeiro container rodando:

<script id="asciicast-kag8uK5nV6dcE3x97vpHTDrll" src="https://asciinema.org/a/kag8uK5nV6dcE3x97vpHTDrll.js" async></script>

Este comando sobe um container simples, de "Hello-world", que escreve algumas coisas no console e fecha o processo, podemos ver que eu não tinha a imagem na minha máquina, então o docker fez o download no [Docker Hub](https://hub.docker.com/).

Podemos subir uma imagem um pouco mais complexa, como a getting-started.
PS. A partir de agora já terei feito o download das imagens para facilitar nos exemplos.

<script id="asciicast-rcT4oK6a5n1La2x0T85ClX0xC" src="https://asciinema.org/a/rcT4oK6a5n1La2x0T85ClX0xC.js" async></script>

Estou subindo a imagem e mapeando com o comando "-p 80:80" a porta 80 do container para a porta 80 do meu computador host, explicarei mais sobre portas a seguir, então se eu acessar <http://localhost> no meu navegador, posso ver uma pagina web do docker:

![image](/assets/images/2020/02/docker-compose-vs/docker-compose-vs-01.jpg)

## Docker compose

Beleza, então conseguimos subir aplicações sem precisar instala-las e de forma rápida, mas se eu tiver, por exemplo, um Redis, um MySQL e um RabbitMq, vou precisar executar toda vez um comando "docker run" pra cada um dos containers? E pra repassar isso para o time todo?

Ai que entra o docker compose, que, nada mais é que uma ferramenta para definir um ou vários containers e subi-los de maneira simples.

### Images

A imagem é um template de comandos, para nosso container subir corretamente, exatamente um passo-a-passo, que você não vai precisar fazer manualmente.

Vamos montar nosso docker compose de exemplo com um Redis, RabbitMq e um MySQL, primeiro vamos atrás das imagens docker dessas aplicações. Assim como usamos a imagem [Hello World](https://hub.docker.com/_/hello-world) e [Getting Started](https://hub.docker.com/_/getting-started) devemos que encontrar as demais, para isso, vamos até o docker hub e fazemos uma pesquisa rápida pelas mesmas aplicações grandes e conhecidas, lá já tem suas imagens oficiais no docker hub e toda a sua documentação:

[Redis](https://hub.docker.com/_/redis)
[MySQL](https://hub.docker.com/_/mysql)
[RabbitMq](https://hub.docker.com/_/rabbitmq)

### Tags

Tags são as versões que você quer utilizar, sendo que existem diversos tipos, como versões específicas de um banco, versões do rabbit com e sem página de manutenção, ou até mesmo com versões diferentes do Kernel, com Alpine sendo menor ou com alguma outra pra ter mais ferramentas.

No meu caso usarei a Latest do MySQL e Redis e a 3.8-management do RabbitMQ, todas pegas da documentação.

### Ports

Lembra que containers são contidos? Eles não são acessados normalmente e é preciso expor portas para isso, assim como expomos a porta 80 para o Getting-Started, seguindo a documentação do docker hub, podemos ver que teremos que usar as seguintes portas para as nossas aplicações:

- Redis
  - 6379
- MySQL
  - 3306
- RabbitMq
  - 5672 - Porta padrão do Rabbit
  - 15672 - Porta do Management

Para ficar mais simples, mapearei as mesmas portas locais, então a porta 6379 do container será mapeada para a porta 6379 da minha máquina e assim por diante.

### Enviroments Variables

Variáveis de ambiente, cada container segue um padrão e em nosso caso usaremos somente no MySQL para setar nossa senha, para o usuário root, no caso do rabbit vamos deixar ele setar as credenciais padrões, que são Guest@Guest e o redis sem credenciais.

- MySql
  - MYSQL_ROOT_PASSWORD=MySecretPassword

### Outras configurações

Existem diversas configurações que podem ser encontradas nas configurações que não usaremos aqui no exemplo, algumas valem a pena ser citadas:

#### Depends_on

Mostra que um container A tem dependência de container B e espera o container A subir para começar a subir o B, lembrando que, o container subir não quer dizer que um banco dentro dele está pronto para receber conexão.

#### Volumes

Mapeia uma pasta da máquina Host para o container, então caso você queira ter algum arquivo compartilhado da máquina host com o container é essa configuração que você usará, lembrando que quando você remove o container, todos os dados dentro dele somem, mas os arquivos compartilhados em volumes não.

#### Network

É possivel criar redes diferentes para comunicação entre containers, apesar de containers que estão no mesmo compose poderem se comunicar, pode ser necessário criar networks para comunicação entre containers em composes diferentes.

### Finalmente montando nosso compose

Agora com todos os dados, podemos montar nosso compose, ele segue um padrão Yaml e ficará assim:

{% highlight Yaml %}
version: '3.4'

services:
  mysql:
    image: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=MySecretPassword
  
  redis:
    image: redis
    ports:
      - "6379:6379"
  
  rabbitmq:
    image: rabbitmq:3.8-management
    ports:
      - "5672:5672"
      - "15672:15672"
{% endhighlight %}

Executando o comando abaixo conseguimos subir os containers e conecta-los de qualquer aplicação na sua máquina.

<script id="asciicast-IZPWFwyVSUxhsPcyub7BodoW5" src="https://asciinema.org/a/IZPWFwyVSUxhsPcyub7BodoW5.js" async></script>

Apertando Ctrl+C os containers param e voltam como novo.

PS. Cuidado para não perder seus dados desse jeito, já que não está sendo mapeado nenhum volume nesse compose, seus dados serão apagados quando os containers forem deletados

## Visual Studio

Agora como podemos usar isso com nossas aplicações .NET? Isso é simples.

### Sem docker

Se a sua aplicação não está rodando no docker é só subir seu docker-compose pela linha de comando e conectar como se as aplicações estivessem local, algo assim para nosso exemplo:

{% highlight json %}
{
    "Redis" : "localhost:6380",
    "RabbitMq": "amqp://guest:guest@localhost",
    "MySql":"Server=localhost;Database=myDataBase;Uid=root;Pwd=MySecretPassword;"
}
{% endhighlight %}

### Com docker

Caso queira rodar sua aplicação no docker, junto com o visual studio, é só habilitar o "Container Orchestrator Support", clicando com o botão direito no seu projeto, o Visual Studio criará seu docker-compose, que você poderá modificar a vontade.

![image](/assets/images/2020/02/docker-compose-vs/docker-compose-vs-02.jpg)

As configurações mudam um pouquinho, agora você não vai acessar mais localhost e sim pelo nome do container, pois todos estão na mesma rede e compose, e sua aplicação não estará mais rodando na sua máquina, e sim em um container, ficando algo assim:

{% highlight json %}
{
    "Redis" : "redis:6380",
    "RabbitMq": "amqp://guest:guest@rabbitmq",
    "MySql":"Server=mysql;Database=myDataBase;Uid=root;Pwd=MySecretPassword;"
}
{% endhighlight %}

E é só definir o projeto de startup como o docker-compose, tudo irá se conectar e conversar.

Sem precisar instalar um monte de ferramentas, sem um monte de gente desenvolvendo no mesmo banco de Dev, e sem precisar gastar 2~3 dias com o Desenvolvedor novo para configurar tudo na máquina, instala o docker e dá um docker-compose up que ta resolvido.
