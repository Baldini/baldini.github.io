---
title: Não use exceptions
tags: Exception
key: dont-use-exceptions
---

Acho que a maioria da pessoas que trabalham com .NET a algum tempo ja deram o famoso "throw new Exception("deu erro aqui")", quando comecei na area era bem comum você ver um código cheio de exceptions para tratar varios tipos de "erros", quer dizer qualquer coisa que não era o caminho feliz mas isso atrapalha em muito o desempenho da aplicação.

## Vamos para o código

Um dos primeiro códigos que fazemos é o famoso "verificar se é par", bom vamos fazer o tratamento disto com exception em um ConsoleApp.

{% highlight C# %}
public void IsEvenException()
{
    try
    {
    if (number % 2 != 0)
        throw new Exception("It's not even");
    }
    catch (Exception)
    {
    }
}
{% endhighlight %}

Podemos simplesmente retornar um bool tambem

{% highlight C# %}
public bool IsEvenBool()
{
    if (number % 2 == 0)
        return true;

    return false;
}
{% endhighlight %}

### Benchmark

E podemos rodar um [Benchmark.net](https://github.com/dotnet/BenchmarkDotNet) para testarmos e vermos como irá se comportar.

{% highlight C# %}
class Program
{
    static void Main(string[] args)
    {
        var summary = BenchmarkRunner.Run<IsEven>();
    }
}

[MemoryDiagnoser]
public class IsEven
{

    [Params(2, 5)]
    public int number;

    [Benchmark]
    public void IsEvenException()
    {
        try
        {
            if (number % 2 != 0)
                throw new Exception("Is not even");
        }
        catch (Exception)
        {
        }
    }
    [Benchmark]
    public bool IsEvenBool()
    {
        if (number % 2 == 0)
            return true;

        return false;
    }
}
{% endhighlight %}

![image](/assets/images/2022/02/dont-use-exception-01.jpg)

Podemos ver que para o número 2, o par, não teve la grande mudança no tempo de execução, já no número impar quando a exception é lançada o tempo explodiu.

Bom, não faz sentido usarmos exceptions para tratarmos o fluxo da nossa aplicação, afinal são exceções, se você está seguindo seu fluxo de trabalho não faz sentido usar exceptions, que para a galera mais antiga vai lembrar, são GOTO menos feios.

## ASP.NET

Se mesmo assim eu não te convenci, vamos para uma aplicação ASP.NET core, com o mesmo código:

{% highlight C# %}
[ApiController]
[Route("[controller]")]
public class IsEvenController : ControllerBase
{
   [HttpGet]
    [Route("Bool")]
    public ActionResult IsEvenBool(int number)
    {
        if (IsEven.IsEvenBool(number))
            return Ok();

        return BadRequest("It's not even");
    }

    [HttpGet]
    [Route("Exception")]
    public ActionResult IsEventException(int number)
    {
        try
        {
            IsEven.IsEvenException(number);
            return Ok();
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }
}

public class IsEven
{
    public static void IsEvenException(int number)
    {
        if (number % 2 != 0)
            throw new Exception("It's not even");
    }   

    public static bool IsEvenBool(int number)
    {
        if (number % 2 == 0)
            return true;

        return false;
    }
}
{% endhighlight %}

E efetuando um CURL podemos verificar os tempos de respostas:

![image](/assets/images/2022/02/dont-use-exception-02.jpg)

Temos mais que o dobro do tempo de resposta, sendo um código extremamente simples.

Então não use exceptions para administrar seus fluxos, você pode usar algum Notification Pattern, Tuples, [Fluent Results](https://github.com/altmann/FluentResults) ou quem sabe até alguma estratégia funcional.
