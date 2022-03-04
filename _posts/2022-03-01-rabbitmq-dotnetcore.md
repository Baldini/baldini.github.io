---
title: RabbitMQ com .NET Core
tags: rabbitmq queue filas
key: rabbitmq-dotnetcore
---

Agora que ja sabemos o que é o RabbitMQ e já até criamos uma Queue e uma Exchange, vamos agora fazer algo com .NET que irá fazer isso.

Ah não sabe do que eu to falando? Esse post é uma continuação desse [AQUI](/2022/02/28/queues-with-rabbitmq.html)

Existem diversas bibliotecas que permitem você conectar no RabbitMQ, desde um biblioteca propria, o [RabbitMQ.Client](https://www.nuget.org/packages/rabbitmq.client/), como o [EasyNetQ](https://www.nuget.org/packages/EasyNetQ) mas eu particulamente gosto muito do [MassTransit](https://masstransit-project.com/), ele tras diversas ferramentas e facilita varias coisas que vamos precisar fazer.

Bom, para começar vamos criar 3 projetos em uma solution nova, 2 Consoles Apps o nosso Consumer e nosso Producer, e uma Class Library para nossos contratos.

![image](/assets/images/2022/03/rabbit-dotnetcore-01.png)

Vamos começar trabalhando em cima da mensagem que vamos enviar, para isso precisamos criar nossos contratos, contratos são de extrema importância e para defini-los usando classes ou interfaces, formando assim um contrato fortemente tipado.

> A equipe do MassTransit aconselha a usar Interface para definir os contratos, mantendo uma separação clara dos contratos e da mensagem.

Vamos criar nosso contrato:

{% highlight C# %}
namespace SimpleLibrary
{
    public interface MyMessage
    {
        public string Content { get; }
    }
}
{% endhighlight %}

Para identificar as mensagens, o MassTransit monta o nome baseado no nome da sua classe/interface junto com seu namespace, então a nossa mensagem é a SimpleLiberary:MyMessage, e para todos os lugares que você quiser utilizar essa mensagem você precisará de um contrato com o mesmo namespace e nome da classe/interface, tanto em consumers quanto producers, isso é algo meio chato no MassTransit mas facilita para identificar as Exchanges pois ele cria as suas Exchanges automaticamente por esse nome, ficando fácil de encontrar quais queues estão recebendo qual mensagem.

Agora partindo pro projeto do Consumer,vamos instalar alguns pacotes no nosso projeto:

[MassTransit.RabbitMQ](https://www.nuget.org/packages/MassTransit.RabbitMQ/)

[MassTransit.Extensions.DependencyInjection](https://www.nuget.org/packages/MassTransit.Extensions.DependencyInjection/)

[MassTransit.AspNetCore](https://www.nuget.org/packages/MassTransit.AspNetCore/)

[Microsoft.Extensions.DependencyInjection](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/6.0.0)

[Microsoft.Extensions.Hosting](https://www.nuget.org/packages/Microsoft.Extensions.Hosting/6.0.1)

[Microsoft.Extensions.Hosting.Abstractions](https://www.nuget.org/packages/Microsoft.Extensions.Hosting.Abstractions/6.0.0)

Após isso criamos uma classe que irá consumir, ela irá herdar da classe IConsumer do MassTransit e irá consumir nossa “MyMessage”


{% highlight C# %}
public class MyConsumer : IConsumer<MyMessage>
{
    public Task Consume(ConsumeContext<MyMessage> context)
    {
        Console.WriteLine($"{context.Message.Content}");
        return Task.CompletedTask;
    }
}
{% endhighlight %}

Toda vez que uma mensagem chegar no nosso consumer irá cair no nosso método Consume, onde terá nossa logica de negócio.

Agora vamos para nosso Main fazer a configuração para nosso consumer ficar rodando.

{% highlight C# %}
class Program
{
    static async Task Main()
    {
        var host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
           {
               services.AddMassTransit(mass =>
               {
                   mass.AddBus(busCtx =>
                   {
                       return Bus.Factory.CreateUsingRabbitMq(cfg =>
                       {
                           cfg.Host(new Uri("amqp://guest:guest@localhost"));
                           cfg.ReceiveEndpoint("MyQueue", re =>
                            {
                                re.Consumer<MyConsumer>();
                            });
                       });
                   });
                   services.AddMassTransitHostedService();
               });
           }).Build();
        await host.StartAsync();
        await host.WaitForShutdownAsync();
    }
}
{% endhighlight %}

Criamos um Host para trabalharmos nossa injeção de dependecia, configuramos o MassTransit para conectar no RabbitMq e configuramos nosso ReceiveEndpoint para utilizar nosso consumer definindo o nome da fila, se iniciarmos nosso projeto já serão criadas nossas exchanges e filas la no RabbitMq.

![image](/assets/images/2022/03/rabbit-dotnetcore-02.png)

Podemos ver que ele criou 2 Exchanges, 1 para nossa mensagem MyMessage e outra para redirecionar para nossa queue, então o caminho que nossa mensagem do tipo SimpleLibrary:MyMessage irá fazer para chegar na nossa Queue MyQueue é esse:

<div align="center" markdown="1">
``` mermaid
graph LR;
    A[Exchange MyMessage]
    B[Exchange MyQueue]
    C[Queue MyQueue]
    A --> B;
    B --> C;
```
</div>

Bom agora que ja temos alguém pra consumir nossas mensagens, vamos fazer quem vai produzir nossas mensagens, o Producer, ele é bem mais simples, vamos precisar instalar somente um pacote:

[MassTransit.RabbitMQ](https://www.nuget.org/packages/MassTransit.RabbitMQ/)

E no nosso método Main adicionar nossa configuração do RabbitMq e publicar uma mensagem que o MassTransit já vai adicioná-lo no local correto

{% highlight C# %}
class Program
{
    static async Task Main(string[] args)
    {
        var bus = Bus.Factory.CreateUsingRabbitMq(cfg =>
        {
            cfg.Host(new Uri("amqp://guest:guest@localhost"));
        });
        await bus.StartAsync();
        try
        {
            while (true)
            {
                await bus.Publish<MyMessage>(new
                {
                    Content = "Hello World!"
                });
                Console.ReadKey();
            }
        }
        finally
        {
            await bus.StopAsync();
        }
    }
}
{% endhighlight %}

Com isso e rodando os 2 projetos, podemos ver as mensagens sendo enviadas de um e sendo consumidos pelo outro sem problemas.

Claro essa foi uma abordagem bem simplista, só pra entendermos um pouco sobre mensagens, producers, consumers, ver na pratica a utilização de Exchanges e Queue. Para o próximo post vamos falar sobre padronização e complicar um pouco mais nosso projeto.

Caso queira os arquivos desse post estão nesse [Repositorio]()