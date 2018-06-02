# Como a v8 otimiza os métodos `.map`, `.filter`, e similares?

> Resposta original para a pergunta do Quora ["Are JavaScript functions like map(), reduce(), and filter() already optimized for traversing array?"](https://www.quora.com/Are-JavaScript-functions-like-map-reduce-and-filter-already-optimized-for-traversing-array/answer/Quildreen-Motta).


Desde 2017 a v8 otimiza *packed arrays*[1] em JS quando você usa os métodos de Array nativos (map, filter, etc). A v8 pode fazer o inlining[2] desses métodos na maioria dos casos, o que dá ao código performance similar à de estruturas de iteração sintáticas. Eu imagino que outros navegadores investiram em esforços similares de otimização, já que esses métodos são muito utilizados em JavaScript hoje em dia.

- - -

As otimizações dependem de cada VM. Na v8, esses métodos são otimizados da mesma forma que qualquer outro código. A diferença é que o algoritmo utilizado por `map()`, `reduce()`, e outras funções nativas em Arrays é muito diferente do `for loop` que a maioria das pessoas utiliza para iteração de arrays. As pessoas ignoram que arrays em JavaScript podem ter elementos faltantes. Isto é, JavaScript possui *sparse arrays*, então o campo `.length` não te diz quantos elementos realmente existem em uma array.

## Por que `Array.prototype.filter()` não é eficiente?

Para exemplificar, vamos dar uma olhada na implementação de `Array.prototype.filter` da v8, que é escrita em JavaScript[3]: https://github.com/v8/v8/blob/73c9be9b31d2e7fa86f4effd36fcc2047a6bf9b3/src/js/array.js#L1179-L1213

Ao ler o código, você vai notar que existe um comentário dizendo que a v8 não pode fazer esse código realmente eficiente, por conta da especificação de `Array.prototype.filter`. Essa função aceita *sparse arrays*, então a implementação precisa verificar se existe realmente um elemento em cada índice (linha 1158). Essa verificação em particular, assim como verificar se uma propriedade existe em um objeto, é lenta. Além disso, funções passadas como argumento para `filter()` não têm a garantia de serem puras, e recebem a array original como argumento. Nada te previne de escrever:

```js
xs.filter(function(element, index, array) {
  delete array[~~(Math.random() * array.length)];
  return true;
})
```

A possibilidade desses *side-effects* torna tais funções bem mais difíceis de otimizar. Por conta disso, compiladores geralmente são mais conservadores nas otimizações que fazem, para que seu programa possa rodar de forma eficiente, mas sem comprometer a corretude dele.

## Funções de ordem superior como `filter()` podem ser eficientes?

Você também vai notar que o código na implementação de `Array.prototype.filter()` **não é** o mesmo que as pessoas geralmente escrevem em um `for loop` que vai fazer uma iteração de array. Como o código faz coisas completamente distintas, comparar a performance deles faz pouco sentido.

Entretanto, podemos compará-los reimplementando `map()` e escrevendo uma função equivalente que usa um `for loop`:

```js
// The map
function map(xs, f) {
    var ys = [];
  for (var i = 0; i < xs.length; ++i) {
    ys[i] = f(xs[i]);
  }
  return ys;
}
function inc(a) {
  return a + 1;
}
map(xs, inc)
 
// The for loop
function forloop(xs) {
  var ys = [];
  for (var i = 0; i < xs.length; ++i) {
    ys[i] = xs[i] + 1;
  }
  return ys;
}
forloop(xs)
```

Agora que as duas funções têm o mesmo comportamento, podemos verificar se a v8 vai tratá-las da mesma forma. Isso pode ser feito analisando a representação intermediária (*intermediate representation*, ou IR) que a v8 usa para compilar JavaScript. [Vyacheslav Egorov](http://mrale.ph/), uma das pessoas que trabalhava no compilador da v8, escreveu uma ferramenta chamada [IRHydra](http://mrale.ph/irhydra/2/) que podemos utilizar para analisar essa IR.

Para o exemplo acima, a v8 vai gerar milhares de linhas de código assembly (código gerado para [map](https://gist.github.com/robotlolita/61574cd59bea3ee9c9b2); código gerado para [for](https://gist.github.com/robotlolita/c3349204fb5bd1ff0533)). Quando usamos a IRHydra para analisar essas linhas, podemos ver que o código de preparação (que verifica e trata os parâmetros) para as duas funções é bem similar. A maior diferença é que `map()` recebe 2 parâmetros, enquanto `forloop()` recebe apenas um. Por conta disso `map()` precisa verificar alguns valores a mais, mas como essa é uma verificação feita antes da iteração ela não tem um impacto muito grande na performance.

![Representação no IRHydra - `map` à esquerda, `forLoop` à direita](../assets/irhydra-map-for.png)

A parte do código que vem antes da iteração precisa alocar uma array nova, e verificar os objetos para garantir que as premissas utilizadas pelo compilador para otimizar o código foram satisfeitas. Dessa forma o compilador pode desotimizar o código e recompilá-lo se você passar uma String ou Objeto ao invés de uma Array--a instrução `CheckMaps` faz essas verificações de tipo. Isso é necessário porque se a v8 executasse um código otimizado para arrays em cima de uma String, a memória seria corrompida, e o programa não funcionaria como o esperado.

A iteração em si é a parte interessante desse IR. Se olharmos para a iteração na função `map()` e na função `forloop()`, podemos perceber que elas têm praticamente as mesmas instruções! A v8 viu que era possível fazer um *inlining* da função passada para `map()`, tornando-a praticamente idêntica a `forloop()`. Note que não existem chamadas de função em `map()` e aonde se esperaria uma existe uma instrução `Add`.

![Representação no IRHydra - `map` à esquerda, `forLoop` à direita](../assets/irhydra-map-for-loops.png)


## Como posso prever a performance sem fazer tudo isso?

"Mas isso dá muito trabalho! Tem alguma forma de prever a performance do meu código sem passar por tudo isso?"

Não. Não tem.

A única forma de saber a performance do seu código em VMs de JavaScript modernas é através da análise e *profiling* cuidadosos. E se você realmente precisar, olhando como cada VM funciona internamente.

A seção anterior foi inteiramente sobre a v8, e enquanto VMs modernas alcançam performance similar em vários aspectos, elas fazem isso através de análises e otimizações diferentes. O que funciona para a v8 não necessariamente funcionará para a SpiderMonkey, a JSC, a Chakra, a Nashorn, ou qualquer outra VM.

*Profiling* da sua aplicação **real** em cada plataforma que você suporta é a melhor forma de ter um *feedback* útil da performance, e decidir como os problemas de performance serão resolvidos, se existirem.


## Micro-otimizações não importam!

Como [Mattias Peter Johansson](https://www.quora.com/profile/Mattias-Petter-Johansson) disse em sua resposta para essa pergunta, micro-otimizações não importam muito para código de verdade. Algoritmos melhores importam. A melhoria de performance vista aqui foi conseguida com *um algoritmo melhor*, inclusive. Mas esse algoritmo melhor é *menos genérico*, e pode ter casos em que ele faria a coisa errada (e.g.: se alguém utilizar uma *sparse array* ou *array-like*).

Enquanto um humano pode analisar cada contexto e decidir qual o melhor algoritmo para cada situação, baseando-se em quais tipos de dados serão realmente usados pela aplicação, o compilador geralmente não pode fazer isso, ele usa as formas mais genéricas, que vão funcionar de forma boa o suficiente para a maioria dos casos.

Além disso, não está claro que `Array.prototype.map()` seria a causa do problema de performance na sua aplicação. Você só pode saber o que será relevante para otimizar depois de uma análise de performance. E, talvez, se `Array.prototype.map()` está causando problemas em uma parte do seu código muito utilizada, é porque arrays não são a melhor estrutura de dados para seu problema. Por exemplo, para testar colisões, você poderia utilizar uma [Quadtree](https://en.wikipedia.org/wiki/Quadtree) e comparar apenas objetos próximos, enquanto uma Array nesse caso dependeria de verificar todos os objetos contra todos os outros objetos.


## Conclusão

Sim, VMs modernas de JavaScript podem otimizar bastante seu código, e isso significa que `Array.prototype.{filter, map, reduce, ...}` são tão otimizados quanto podem ser, enquanto ainda são funções genéricas. Esses métodos são eficientes para a maioria das aplicações que podem utilizá-los.

A performance dessas funções, entretanto, não resolverá problemas de escolher o algoritmo ou estrutura de dados errado para sua aplicação. Sempre dê uma olhada nos seus algoritmos e estruturas de dados antes de fazer qualquer outra otimização.

- - -

## Notas de rodapé

[1] **packed array** é uma array de tamanho N aonde todos os índices possuem um elemento. JavaScript também suporta **sparse arrays**, que são arrays de tamanho N que possuem um número menor de elementos do que N--então nem todos os índices possuem um elemento.

[2] **inlining** é uma otimização para chamadas de função aonde o compilador, ao invés de emitir uma chamada de função, emite o código dessa função (com os parâmetros devidamente substituídos). É como se o compilador fosse um grande programador-copia-e-cola. Essa é uma das otimizações mais importantes em linguagens OOP.

[3] VMs têm boa parte da biblioteca padrão de JavaScript implementada em JavaScript, ao invés de C++ (ou outra linguagem utilizada pela VM, como Java). Isso permite que a VM utilize as mesmas otimizações para essas funções, e permite usar **inlining** para chamadas dessas funções (o que não seria possível se elas fossem escritas em C++).

