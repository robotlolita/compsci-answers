# O que `this.` significa em JavaScript?

> Resposta originalmente para a pergunta do Quora ["What does `this.` in JavaScript mean?"](https://www.quora.com/What-does-this-in-JavaScript-mean/answer/Quildreen-Motta)

**Sumário**: `this` é um argumento comum passado na hora que uma função (do tipo método) é invocada. JavaScript reserva a sintaxxe `objeto.método(foo)` para significar `var f = objeto.método; f.call(objeto, foo)`, aonde `Function.prototype.call` é como você passa esse argumento `this` para uma função, já que JavaScript tomou a decisão de deixar esse parâmetro implícito nas chamadas de funções e definições.

Por conta disso é impossível saber que valor `this` vai ter antes da função ser chamada. Isso acontece em toda linguagem orientada a objetos. O resto desse artigo explica o porquê.

- - - 

## `this` em linguagens orientadas a objeto em geral

Vamos começar do início.

Em uma linguagem orientada a objetos você tem objetos, que são valores contendo operações. Essas operações são geralmente chamadas de "métodos". E esses métodos definem o que cada objeto pode fazer. Usamos objetos apenas chamando métodos neles.

> Nota: isso é uma simplificação. Essa definição funciona para esse artigo e um entendimento superficial de OOP, mas deixa de ser verdade quando você começa a olhar para detalhes específicos de diferentes modelos ou linguagens.

Agora nós temos um problema: esses métodos algumas vezes precisam saber sobre outros métodos no mesmo objeto. Como eles referenciam um ao outro?

Bom, a forma mais simples é apenas chamar o outro método diretamente, certo?

```js
object Foo {
  métodoA(x) {
    métodoB(x + 1);
  }

  métodoB(x) {
    print(x);
  }
}

Foo.métodoA(1); // exibe "2"
```

Esse objeto possui dois métodos: `métodoA` e `métodoB`. `métodoA` depende da existência de `métodoB`, o que parece okay, certo...?

Mas isso acaba sendo um problema quando você adiciona o conceito de herança. Se `métodoA` se referir a `métodoB` diretamente, não é possível herdar desse objeto e mudar apenas o `métodoB`. Nós perdemos uma parte integral de programas orientados a objetos, e isso não é bacana.

Vamos tentar algo diferente. Ao invés de se referir à função (seu endereço) diretamente, nós vamos usar um dicionário que diz aonde a função está:

```js
object Foo {
  dicionário = Foo;

  métodoA(x) {
    return dicionário.métodoB(x + 1);
  }

  métodoB(x) {
    print(x);
  }
}

object Bar extends Foo {
  dicionário = Bar;

  métodoB(x) {
    print(x);
    print(x);
  }
}

Bar.métodoB(1); // exibe "1", depois "1"
Bar.métodoA(1); // exibe "2"
```

Agora cada objeto define uma variável local chamada "dicionário" e... ah, ainda não funciona. Para o `métodoA`, a variável `dicionário` ainda é `Foo`, porque é isso que está no escopo da função. Isso não funciona com herança porque estamos tentando pegar algo de um escopo e usar em um escopo diferente.

Nós *poderíamos* fazer isso funcionar simplesmente copiando toda a memória de Foo e duplicando isso em Bar, mas mudando as variáveis locais de todas as funções copiadas de Foo para funcionarem no escopo de Bar. Mas isso não é uma boa solução, nós teríamos um consumo maior de memória, e um aumento do tamanho dos programas.


Uma solução melhor provavelmente existe.

Vamos voltar ao problema: funções estão se referindo a valores que são decididos no momento em que a função é *definida*. Isso está causando dores de cabeça porque, se adicionamos herança, nós ainda não sabemos em que locais a função será *usada*.

Por exemplo, se definirmos templates de objetos:

```js
class Foo {
  métodoA(x) {
    aqui.métodoB(x + 1);
  }

  métodoB(x) {
    print(x);
  }
}

class Bar extends Foo {
  métodoB(x) {
    print(x);
    print(x);
  }
}
```

Então podemos decidir que quando esses templates são resolvidos (ou instanciados) nós vamos copiar toda a memória e mudar os tokens especiais `aqui` para o endereço do objeto de fato:

```js
bar = new Bar(); // instancia o template duplicando a memória
bar.métodoA(1); // exibe "2" depois "2"
bar.métodoB(1); // edibe "1" depois "1"
```

Apesar de funcional, isso ainda tem um problema de consumo de memória muito grande, e como essa duplicação de memória é feita na hora de criar um objeto, criar objetos se torna algo de processamento intensivo. Não dá para ter muitos objetos em memória dessa forma.

Além disso, se torna impossível chamar um método que foi sobre-escrito. Se `Bar` herda de `Foo` mas altera o método `métodoB`, não é possível mais chamar o `métodoB` de `Bar`.

Okay, talvez tenhamos nos confundido com o que significa *usar* um desses métodos. Nós não reutilizamos esses métodos quando herdamos ou instanciamos algo. Nós os reutilizamos quando *chamamos* esse método. E que forma melhor de informar ao método aonde estão as funções que podem ser acessadas do que nos *parâmetros* daquele método? Sendo assim, vamos adicionar um parâmetro `dicionário` em cada método que nos diz aonde os outros métodos estão!

(e vamos voltar para objetos porque eu não gosto de classes :'>)

```js
object Foo {
  métodoA(dicionário, x) {
    dicionário.métodoB(dicionário, x + 1);
  }

  métodoB(dicionário, x) {
    print(x);
  }
}

object Bar extends Foo {
  métodoB(dicionário, x) {
    print(x);
    print(x);
  }
}

Bar.métodoA(Bar, 1); // exibe "2" depois "2"
Bar.métodoB(Bar, 1); // exibe "1" depois "1"
```

Existem duas formas para implementar `object A extends B`. A primeira é simplesmente duplicar o dicionário (que define nomes para as funções) de B e adicionar o dicionário de A. Isso não é muito eficiente em termos de memória e esforço de CPU para criar um objeto. A outra forma, que a maioria das linguagens (incluindo JavaScript) usa, é apenas manter uma referência interna para `B`. Quando um método é chamado em `A` e ele não existe no dicionário, o método é pesquisado no dicionário de `B`, que está nessa referência interna. Aonde os métodos estão na memória não importa porque nós vamos passar o dicionário para cada método chamado.

> **tangente**: esse parâmetro `dicionário` é geralmente chamado de "receiver" (receptor), e comumente nomeado `this`, `self` ou `me`.

Antes de continuar e olhar para as coisas específicas de JavaScript... ter que escrever `Bar` duas vezes para chamar um método é um pouco chato. Podemos descartar um desses.

```js
object Foo {
  métodoA(x) {
    this.métodoB(x + 1);
  }
  // equivalente a:
  //
  // métodoA(this, x) {
  //   this.métodoB(x + 1);
  // }

  métodoB(x) {
    print(x);
  }
}

object Bar extends Foo {
  métodoB(x) {
    print(x);
    print(x);
  }
}

Bar.métodoA(1); 
// => equivalente a `Bar.métodoA(Bar, 1)`
```

Agora o parâmetro dicionário é implícito, e pode ser resolvido com a sintaxe especial `Bar.método()` como sendo sempre o objeto ao lado esquerdo do `.`.

> **Nota**: tornar esse parâmetro implícito é o que causa essas confusões em pessoas tentando entender orientação a objetos para começar. Linguagens orientadas a objetos não deveriam omitir esse parâmetro. Lua, Python, F#, e Siren fazem o parâmetro explícito, que é mais fácil de entender.

## Dispatch

Quando chamamos um método em uma linguagem orientada a objetos (incluindo JavaScript) existem duas operações sendo executadas: **Seleção** e **Invocação**. Quando falamos das duas juntas nós usamos o termo **Dispatch**. A maioria das linguagens não tem uma fase separada (observável pelo usuário) de **Seleção** e **Invocação**, o que reduz a confusão com argumentos receptores, mas também proíbe que você use métodos como funções de primeira classe—que é um conceito poderoso.

Durante a fase de **Seleção** o runtime vai olhar quais objetos estão envolvidos na chamada de método para decidir qual operação deve ser executada. Em geral (e certamente o caso para JavaScript), a seleção considera apenas o objeto à esquerda do `.`, também chamado de "single dispatch". Algumas seleções podem considerar os argumentos comuns (à direita do `.`) depois também, conhecido como "double dispatch". Ou ainda, a seleção pode considerar que todos os argumentos têm o mesmo "peso" nessa decisão ("multiple dispatch").

> **Nota**: nem todas as linguagens usam `.` para a sintaxe de chamada de métodos. Algumas usam espaço, algumas usam `:`, algumas usam `->`, etc.

Como mencionado, JavaScript, assim como várias outras linguagens orientadas a objeto, usa single dispatch. Isso significa que em uma expressão como `objeto.método(a, b, c)` a seleção só usa `objeto` para decidir qual operação vai ser executada para `método`.

Depois da fase de seleção vem a fase de **Invocação**. Agora que temos uma operação (um endereço de memória com instruções para executar) tudo o que falta é passar os argumentos para essa operação e executá-la. Como isso é feito varia de implementação para implementação (argumentos podem ser adicionados à pilha antes de pular para o endereço da função, podem ser definidos em registros, etc. O caso específico não é relevante aqui).

Juntando tudo isso, nossa sintaxe mágica `objeto.método(a, b, c)` se parece com isso dentro da VM de JavaScript:

```js
// Isso importa se alguma dessas expressões não for uma simples variável
const receptor = objeto;
const arg1 = a;
const arg2 = b;
const arg3 = c;

// Pegamos uma referência para a função a partir do dicionário
const operação = receptor['método'];

// E chamamos a função passando todos os argumentos.
operação.call(receptor, arg1, arg2, arg3);
```

Note como a operação é chamada com `operação.call(receptor, ...)` e não apenas `operação(receptor, ...)`. Isso porque em JavaScript o parâmetro receptor é completamente implícito, e precisamos de algum esforço para passá-lo para a função. Esse não é o caso em Python ou Lua, que tornam esse parâmetro explícito.

O que acontece quando temos apenas `objeto.método`? JavaScript permite isso, e aplica a mesma regra. Mas agora nós temos que ter o cuidado de chamar essa função corretamente passando o argumento receptor que ela espera porque a linguagem não vai fazer isso automáticamente mais.

Uma forma de contornar esse problema é criar uma função com um parâmetro receptor pré-definido. Isto é, a função sempre vai passar o mesmo valor para o parâmetro receptor. Isso pode ser feito em JavaScript com o método `.bind` de funções. Nota que não é mais possível usar essa função como um *método* após fazer isso, porque perdemos a possibilidade de passar o parâmetro receptor!

```js
const operação = objeto.método.bind(objeto);
operação(a, b, c); // sempre equivalente a "operação.clal(objeto, a, b, c)"
```

Em Python, quando você tem `objeto.método`, a linguagem vai criar uma função com o parâmetro receptor pré-definido automaticamente, e para evitar isso você precisa usar `classe.método`. Como JavaScript não possui classes isso não faria sentido.

Em Java 8, uma forma de transformar métodos em funções de primeira classe foi adicionada. Nessa versão a sintaxe `objeto::método` faz o mesmo que `objeto.método.bind(objeto)` em JavaScript. [Existe uma proposta de melhoria para adicionar essa sintaxe em JavaScript](https://github.com/tc39/proposal-bind-operator).

Outras linguagens variam geralmente não permitem esse tipo de código (seleção e invocação sempre são feitas em conjunto), como acontece em Ruby.

## Peculiaridades de JavaScript

### Locais aonde `this` é permitido

Em JavaScript, `this` pode ser utilizado fora de uma função. Quando usado nessa forma, `this` se refere ao [objeto global](https://www.ecma-international.org/ecma-262/8.0/#sec-global-object). Note que em Node.js seu código nunca é executado fora de uma função, por conta do sistema de módulos—inclusive, `return` funciona em qualquer lugar do seu módulo Node.

### Funções "arrow"

ECMAScript 2015 adiciona arrow functions. Essas funções são declaradas como `(argumentos, vão, aqui) => expressãoOuBloco`. Arrows *não são* métodos, e portanto elas nunca recebem um argumento receptor. Você também não pode usar `.call` ou `.bind` nelas por conta disso.

As pessoas geralmente fazem essa distinção dizendo que "arrows possuem `this` léxico". Mas essa descrição é confusa porque "this" sempre foi um nome léxico em JavaScript—assim como qualquer outro parâmetro na sua função. O significado de `this` é sempre o mesmo *dentro do escopo de uma função*. O que acontece é que, como cada função comum introduz sua própria variável `this`, as variáveis de funções que a contém não são visíveis. Arrows não têm esse problema porque simplesmente não adicionam uma nova variável com esse nome.

```js
function foo(a, b) {
  return function bar(b, c) {
    return (c, d) => this + a + b + c + d;
  }
}.call(1, 2, 3).call(3, 4, 5).call(5, 6, 7)

// É resolvido como:
// 3 + 2 + 4 + 6 + 7

// Porque é entendido como
(this, a, b) => {
  return (this, b, c) => {
    return (c, d) => this + a + b + c + d
  }
}(1, 2, 3)(3, 4, 5)(6, 7);

// Note que `this` e `b` da primeira função, e `c` da segunda
// não são visíveis porque as funções de dentro introduzem
// novas variáveis com o mesmo nome
//
// Se aplicarmos as substituições:

// Substituindo `this`, `a`, e `b` na primeira função por 1, 2, 3:
(1, 2, 3) => {
  return (this, b, c) => {
    return (c, d) => this + 1 + b + c + d
  }
}(3, 4, 5)(6, 7)

// Substiuindo `this`, `b`, e `c` na segunda função por 3, 4, 5:
(1, 2, 3) => {
  return (3, 4, 5) => {
    return (c, d) => 3 + 1 + 4 + c + d
  }
}(6, 7)

// Finalmente substituindo `c` e `d` por 6 e 7 na terceira função:
(1, 2, 3) => {
  return (3, 4, 5) => {
    return (6, 7)  => 3 + 1 + 4 + 6 + 7
  }
}
```

### Strict mode

Em versões antigas de JavaScript (e novas sem `strict mode`), invocar uma função sem um argumento receptor no lado esquerdo passa o objeto global como receptor. Em alguns casos isso faz sentiddo, já que variáveis podem ser definidas no objeto global, e `foo(a)` pode ser visto implicitamente como `global.foo(a)`. Mas isso acontecia mesmo para variáveis locais, e causava vários tipos de problemas. Com `strict mode` as invocações de função que não usam a sintaxe especial de chamada de método (`objeto.método(...)`) passam `undefined` como o parâmetro receptor.