---
title: Serialização com protobuf
tags: .NET protobuf serialização
key: netcoreprotobuf01
---

Este é um post escrito em 2019, o original foi escrito no [Medium](https://medium.com/@guibaldini/protobuf-dotnet-43f993ca9bdb) e passei para cá

------------------------------------------

[Protocol Buffers](https://developers.google.com/protocol-buffers/) ou protobuf para os íntimos é um método de serialização de dados estruturados, criado pela Google para comunicação entre serviços internos, com ele você cria um arquivo de configuração, arquivo .proto, com sua estrutura de dados e importa na sua linguagem favorita e gerar suas classes de _Dados._

<script src="https://gist.github.com/Baldini/e28471a85240e1d7eabba22703cf1fbf.js"></script>


Nele definimos o tipo dos dados, as estruturas, obrigatoriedade, valores default, etc. Existe uma [documentação](https://developers.google.com/protocol-buffers/docs/proto) completa mas um ponto que devemos prestar atenção é que existe um numero único para cada campo, ele deve ser único dentro da sua estrutura e serve para identificação do campo dentro da formatação binária.

Então ele é igual Json e XML?
-----------------------------

Sim e não, ele é sim um tipo de serialização mas ele é mais completo, mais simples e mais leve.

Podemos importar o arquivo .Proto com um [utilitário](https://www.nuget.org/packages/protobuf-net.Protogen), feito com .NET para gerar nossa classe de _Data._

```
dotnet tool install --global protobuf-net.Protogen --version 2.3.17
```

Após instalado podemos executar o comando abaixo para gerar nosso código com qualquer arquivo .proto.

```
protogen <File> --csharp\_out=<Output>
```

Ele irá gerar o código abaixo
<script src="https://gist.github.com/Baldini/8a9f8a3246a68f1c6ec0034ccb26ca58.js"></script>

Dica: Existe um [site](https://protogen.marcgravell.com) que faz exatamente esse processo, caso não queira instalar a global tool

Pra quem já utilizou _web services SOAP_, isso se parece muito com o arquivo que é gerado com _WSDL_, mas calma não morra do coração, não é isso, ele é só um arquivo de modelo, com isto você não consegue efetuar chamadas diretamente para o _webservice_, e quando você olha com mais atenção vê que não tão feio quanto o gerado pelo _WSDL._

Com isso podemos utilizar o pacote [Google.Protobuf](https://www.nuget.org/packages/Google.Protobuf/) e serializar de forma simples:
<script src="https://gist.github.com/Baldini/f2bd46ecc33846d3f9da0fa19e5c456f.js"></script>

Protobuf-net
------------

O [protobuf-net](https://www.nuget.org/packages/protobuf-net) é uma library criada para facilitar a utilização de protobuf em Dot Net, ele suporta tanto classes com Data Annotations, bem parecido com o antigo JsonProperty, quanto o arquivo .proto, eu gosto mais dele pois as classes ficam mais legíveis do que com o pacote do google, para o exemplo irei usar Data Annotation pra facilitar a minha vida ¯\\\_(ツ)\_/¯.

Vou iniciar criando meus modelos, resolvi criar 2 tipos de modelos, um com somente uma propriedade do tipo Int, e outro com alguns campos a mais e o model mais simples, adiciono a Data Annotation _\[ProtoContract\]_ nas classes e os _\[ProtoMember()\]_ nas propriedades.

<script src="https://gist.github.com/Baldini/e74cd441f203fb97719d9d8ebe4110df.js"></script>


Como dar pra perceber, diferente de Json ou XML, não existe um nome para os campos, como por exemplo _\[JsonProperty(“MyString”)\]_, ao invés disso cada campo tem um inteiro como identificador, que não pode se repetir na sua classe, sem esse Annotation o campo não será serializado.

Dica: ele não aceita números negativos e números menores ocupam menos espaço, não comece em 100000000 XD.

Tá, é só isso?
--------------

Não, mas vamos fazer o código de serialização em protobuf junto com um código de serialização em Json para vermos se realmente tem alguma diferença no desempenho, vou usar o [Benchmark Dot Net](https://github.com/dotnet/BenchmarkDotNet) pra me auxiliar e o [Bogus](https://github.com/bchavez/Bogus) para criar alguns dados fake.

<script src="https://gist.github.com/Baldini/928ea0bd66839fb4d0b4454f03b23150.js"></script>

Montei 4 tipos de testes, um para cada tipo de serialização, totalizando 8 sendo eles:

1.  Serializar _SimpleClass;_
2.  Serializar _ComplexClass;_
3.  Serializar uma lista com 10 _ComplexClass;_
4.  Serializar uma lista com 100 _ComplexClass._

Bom então vamos modificar nosso método _Main_ e iniciar o teste

<script src="https://gist.github.com/Baldini/13c27f31d621f9cf693848e010ef1495.js"></script>

Fiz o _Build_ da aplicação em _release_ e executei-a por linha de comando, o resultado é este:

 ![image](/assets/images/2019/05/2019-05-30-protobuf_01.png)

Resultado do Benchmark efetuado

Esse é um Benchmark bem simples mas podemos perceber diminuição alocação de memoria e tempo gasto na operação ,principalmente conforme ela vai aumentando a quantidade, são bem menores com o Protobuf-net, fiz alguns outros testes e usando a biblioteca do [Google](https://www.nuget.org/packages/Google.Protobuf/) você consegue ganhar mais um pouco de desempenho.

Existem alguns benchmarks mais completos na internet caso queira saber mais, por exemplo [este](https://auth0.com/blog/beating-json-performance-with-protobuf/) e [este](https://codeburst.io/json-vs-protocol-buffers-vs-flatbuffers-a4247f8bda6f) dentre outros

 ![image](/assets/images/2019/05/2019-05-30-protobuf_02.jpg)

Então vou usar protobuf pra tudo?
---------------------------------

Calma jovem padawan, não é para tudo que devemos usar, Json ainda é muito bom para os seguinte casos na minha opnião:

1.  Você precisa que seus dados sejam legíveis para humanos;
2.  Você vai integra com algum cliente (afinal seu cliente pode não fazer ideia do que é protobuf);
3.  Seus dados são dinâmicos (não existe um schema);
4.  Sua aplicação que irá consumir os dados é em Javascript ou um Browser.

Mas para comunicação interna entre seus serviços, cache, filas e etc você pode usar protobuf sem problemas :D

 ![image](/assets/images/2019/05/2019-05-30-protobuf_03.png)


<!--more-->
