---
title: Injeção de dependência
tags: .NET dependency-injection ASPNET
key: aspnet-core-dependency-injection
---

A ideia da injeção de dependência é manter o desacoplamento entre módulos do sistema, com a injeção as classes de dependências não são instanciadas diretamente na classe utilizada, mas sim em uma estrutura responsável por isso, um container, isso nos ajuda a controlar essas instancias de forma mais simples e performática, já que o .NET faz isto para nós e facilitando os testes unitários por trabalhar com interfaces, assim facilitando os mocks.

Ou seja, onde antes fazíamos isso:

{% highlight C# %}
public class Foo
{
    private Bar bar { get; set; }
    public Foo()
    {
        this.bar = new Bar();
    }
}
{% endhighlight %}

Agora faremos

{% highlight C# %}
public class Foo
{
    private Bar bar { get; set; }
    public Foo(Bar Bar)
    {
        this.bar = Bar;
    }
}
{% endhighlight %}

Bom, vamos explicar como funciona por exemplo

## Exemplos

Para exemplo vamos criar nossa classe a ser injetada:

{% highlight C# %}
public class MyRepository
{
    public Guid MyGuid { get; set; }
    public int MyInt { get; set; }
    public MyRepository()
    {
        MyGuid = Guid.NewGuid();
    }

    public void PlusOne()
    {
        MyInt += 1;
    }
}
{% endhighlight %}

iremos criar uma classe que irá usar nosso ~repository~

{% highlight C# %}
public class MyService
{
    private readonly MyRepository repository;

    public MyService(MyRepository repository)
    {
        this.repository = repository;
    }

    public void PlusOne()
    {
        repository.PlusOne();
    }
}
{% endhighlight %}

E nosso controller

{% highlight C# %}
[ApiController]
[Route("[controller]")]
public class MyController : ControllerBase
{
    private readonly MyRepository repository;
    private readonly MyService myService;

    public MyController(MyRepository repository, MyService myService)
    {
        this.repository = repository;
        this.myService = myService;
    }

    [HttpGet]
    public int Get()
    {
        repository.PlusOne();
        myService.PlusOne();
        return repository.MyInt;
    }
}
{% endhighlight %}

### Scoped

Classes scoped são instanciadas a cada novo escopo solicitado, no caso do ASP.NET Core, cada chamada HTTP recebida.

São ótimas opções para classes que mantem estado e como mantem uma instancia por chamada é mais difícil ter problemas de memória por causa delas, claro que se o código dentro dela tiver problemas não tem muito pra onde fugir.

#### Code

Vamos injetar nosso "Repository" e "Service" no nosso startup

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<MyRepository>();
    services.AddScoped<MyService>();
    services.AddControllers();
}
{% endhighlight %}

Rodando o projeto e chamando nosso endpoint podemos perceber que a instancia da classe MyRepository é compartilhada entre o MyController e o MyService, já que ambas estão sendo utilizadas dentro no mesmo escopo, uma chamada HTTP.

![image](/assets/images/2020/02/2021-02-18-dependency-injection-01.jpg)

![image](/assets/images/2020/02/2021-02-18-dependency-injection-02.jpg)

Ambos os GUID são iguais e, como chamamos o método PluOne() 2 vezes, o nosso MyInt é 2.

### Transient

Classes Transient são instanciadas todas as vezes que são solicitadas.

Bom para classes leves e sem estado, mas podem causar aumento do uso de recursos e problema de memória.

#### Code

Mudamos a injeção do nosso "Repository" para Transient

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddTransient<MyRepository>();
    services.AddScoped<MyService>();
    services.AddControllers();
}
{% endhighlight %}

![image](/assets/images/2020/02/2021-02-18-dependency-injection-03.jpg)

![image](/assets/images/2020/02/2021-02-18-dependency-injection-04.jpg)

Podemos observar que os GUID são diferentes para cada classe e o valor do MyInt não é mantido, pois dentro do transient cada vez que é solicitada, a classe de repository é instanciada novamente.

### Singleton

Classes Singleton são instanciadas na primeira vez que são solicitadas, toda nova vez que for solicitada vai ser enviado a mesma instancia.

#### Code

Mudamos a injeção do nosso "Repository" para singleton
{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<MyRepository>();
    services.AddScoped<MyService>();
    services.AddControllers();
}
{% endhighlight %}

![image](/assets/images/2020/02/2021-02-18-dependency-injection-05.jpg)

![image](/assets/images/2020/02/2021-02-18-dependency-injection-06.jpg)

Aqui o comportamento parece igual ao Scoped, mesmo GUID e o MyInt subindo para 2, mas se efetuarmos uma nova chamada teremos algumas mudanças... ou não.

![image](/assets/images/2020/02/2021-02-18-dependency-injection-07.jpg)

![image](/assets/images/2020/02/2021-02-18-dependency-injection-08.jpg)

Mesmo GUID da chamada anterior e o MyInt não resetou, já que classes Injetadas como Singleton só tem 1 instancia durante todo o ciclo de vida da aplicação, o que pode ser muito bom para classes que podem ser reutilizáveis em diversos pontos ou muito ruim causando memory leaks ou quebra de contextos entre chamadas.

## Bonus

### Entity Framework Context

Quando você está trabalhando com Banco de dados e injeção de dependências sempre fica a dúvida sobre qual usar, normalmente dentro da documentação da biblioteca que você está usando já vem indicando como é a melhor forma de cuidar da sua instancia de conexão mas caso esteja usando Entity Framework Core, já temos uma facilitada:

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<MyContext>(options => options.UseSqlServer(connectionString));
    services.AddControllers();
}
{% endhighlight %}

Com isso já temos a injeção do nosso Context e podemos receber ele em nossas classes mas podemos melhorar um pouco mais:

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContextPool<MyContext>(options => options.UseSqlServer(connectionString));
    services.AddControllers();
}
{% endhighlight %}

Com o AddDbContextPool, habilita um pool de instancias reutilizáveis do seu contexto, melhorando o desempenho e consumo de memória.
