# Como a v8 otimiza os métodos .map/filter e similares?

> Resposta original para a pergunta do Quora ["Are JavaScript functions like map(), reduce(), and filter() already optimized for traversing array?"](https://www.quora.com/Are-JavaScript-functions-like-map-reduce-and-filter-already-optimized-for-traversing-array/answer/Quildreen-Motta)

**EDIT:** Desde 2017, a v8 agora otimiza os arrays empacotados de JS quando usando os métodos integrados de Array (`map`, `filter`, etc), incluindo passá-los internamente em muitos casos, dando a eles a mesma performance de `for-loop`s comuns. Eu assumiria que outros navegadores se esforçaram para fazer o mesmo, visto que, atualmente, esses padrões são muito comuns em JavaScript.

Depende da VM. Essas funções são tão otimizadas quanto qualquer outro código na v8, a diferença é que o algoritmo usado por `map()`, `reduce()` e derivados, é muito diferente do `for-loop` que muitas pessoas usam para iterar arrays, porque ignoram que **arrays em JavaScript podem ter buracos** (por exemplo: qualquer array pode ser esparso, então o tamanho não reflete o número de elementos que o array tem).

## Por que Array.prototype.filter() não é rápido?

Para exemplificar, vamos dar uma olhada na implementação de `Array.prototype.filter` na v8, que é implementado em JavaScript¹: https://github.com/v8/v8/blob/73c9be9b31d2e7fa86f4effd36fcc2047a6bf9b3/src/js/array.js#L1179-L1213

Lendo esse código, você irá notar que há um comentário dizendo que a v8 não pode tornar o código realmente eficiente, e isso é devido a semântica em `Array.prototype.filter`. A função aceita arrays esparsos, então a implementação precisa verificar se há algum `index` (índice) presente ou não no array (linha 1158). Essa verificação em particular, assim como verificar se uma chave está presente em um objeto, é lenta. Além disso, funções passadas como argumento para `filter()` não são garantidas como puras, e elas aceitam o array original como argumento, então nada te previne de escrever:

```javascript
xs.filter(function(element, index, array) {
  delete array[~~(Math.random() * array.length)];
  return true;
})
```

A possibilidade desses efeitos colaterais torna difícil das coisas serem otimizadas, então essas mesmas coisas tomam abordagens mais conservadoras para que seu programa possa rodar o mais rápido possível enquanto mantendo o comportamento correto.

## Funções de Ordem Maior como filter() podem ser rápidas?

Você também irá notar que o código que você vê em `Array.prototype.filter()` **não é** o mesmo código que as pessoas normalmente escrevem em um `for-loop` para iterar sobre o array. Comparar a performance de um `for-loop` com aquela implementação de `Array.prototype.filter()` é como comparar maçãs à laranjas — você não está fazendo a mesma coisa.

Para ter certeza que estamos comparando as mesmas coisas, iremos reimplementar `map()` e então escrever o equivalente desse código usando um `for-loop`.

```javascript
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
function forLoop(xs) {
  var ys = [];
  for (var i = 0; i < xs.length; ++i) {
    ys[i] = xs[i] + 1;
  }
  return ys;
}
forLoop(xs)
```

Agora ambas as funções fazem a mesma coisa. Nós podemos proceder para checar se a v8 trata ambas da mesma forma. Isso pode ser feito analisando a representação intermediária que a v8 usa quando compila. [Vyacheslav Egorov](http://mrale.ph/), uma das pessoas trabalhando no compilador da v8, escreveu uma ferramenta chamada IRHydra (http://mrale.ph/irhydra/2/), que pode ser usada para analisar essa RI.

Para o exemplo acima, a v8 irá gerar milhares de linhas de código assembly (veja [0-map.js](https://gist.github.com/robotlolita/61574cd59bea3ee9c9b2) e [0-for.js](https://gist.github.com/robotlolita/c3349204fb5bd1ff0533)), que podemos carregar no IRHydra. O preâmbulo de ambas as funções é muito parecido, com a diferença de que `map()` aceita 2 parâmetros, enquanto `forLoop()` aceita 1. Da mesma forma, `map()` precisa checar mais valores, mas é somente uma checagem antes do looping, então não tem tanto impacto:

![Representação no IRHydra - `map` à esquerda, `forLoop` à direita](../assets/irhydra-map-for.png)

O parte pré-loop tem o código para alocar um novo array e checar os objetos para garantir que estamos sob as mesmas suposições nas quais o código foi otimizado (se você tivesse passado uma `String` ou um `Object` para ambos os `map()` e `forLoop()`, isso teria violado o `CheckMaps` e faria a v8 recompilar a função ou a desmelhoraria).

A parte interessante é o loop. Se você olhar o loop de ambos `map()` e `forLoop()`, você irá perceber que as instrução são quase iguais! A v8 percebeu que era seguro passar internamente a função passada para `map()`, o que fez os dois pedaços de códigos serem iguais (note a falta de chamadas de função na versão de `map()` e a instrução `Add` no mesmo lugar):

![Representação no IRHydra - `map` à esquerda, `forLoop` à direita](../assets/irhydra-map-for-loops.png)

## Eu posso saber a performance sem tudo isso?

Você talvez diga "Mas isso é muito trabalho!".

— _Posso, de alguma forma, prever como meu código irá performar sem fazer tudo isso?_  
— Não. Não pode.

A única forma de saber como algo performa nas VMs modernas de JavaScript é perfilando cuidadosamente, e se você realmente tiver que fazer isso, olhar o mecanismo interno da VM. Mesmo a seção anterior tendo sido toda sobre a v8, as VMs modernas alcançam performances similares através de diferentes tipos de análises e otimizações, então o que funciona bem na v8 talvez não funcione tão bem para o SpiderMonkey, ou JSC, ou Chakra, ou Nashorn, ou qualquer outra VM. Perfilar sua aplicação **real** em cada plataforma que você queira suportar, é, ainda, a melhor forma de obter feedbacks úteis sobre sua performance e decidir o que fazer para direcionar quaisquer problemas de performance que você talvez tenha.

## Micro-otimizações não importam!

Como [Mattias Petter Johansson](https://www.quora.com/profile/Mattias-Petter-Johansson) disse em sua resposta para essa questão, micro-otimizações não importam tanto para o código do mundo real. Melhores algoritmos, sim. Na verdade, o ganho de velocidade na seção de benchmark, comparado a `Array.prototype.map`, foi alcançado por ter um melhor algoritmo. Entretanto, esse _algoritmo melhor_ não funciona tão bem com a semântica de JavaScript. Isto é, enquanto era mais rápido, também era errado, porque falhava com arrays esparsos e objetos que funcionam como arrays! Um humano poderia muito bem determinar que este determinado algoritmo é o correto _para a aplicação dele_, porque ele sabe qual é o tipo de dado que ele está lidando. O compilador e o padrão da linguagem não pode fazer isso, então você acaba ficando com a versão sobrecarregada na biblioteca padrão para dar suporte a todos esses outros tipos de casos de uso.

Além do mais, não é claro que `Array.prototype.map()` é o gargalo na sua aplicação. Você só sabe o que é válido de otimizar ao perfilar cuidadosamente seu código. E se `Array.prototype.map()` é lento em um caminho frequente do seu código, talvez `Array` não seja a estrutura de dados correta para o seu caso de uso. Por exemplo, se você está testando por colisão, usar uma `QuadTree` cortaria o número de objetos que você precisaria comparar contra uma grande quantidade, enquanto usando um `Array` iria requerer que você checasse cada objeto contra cada outro objeto (não apenas os próximos).

## Conclusão

Sim, VMs modernas de JavaScript podem otimizar as coisas muito bem, e isso significa que `Array.prototype.(filter, map, reduce, ...)` são todas o mais otimizadas possível, mesmo sendo o mais generalizado possível. Esses métodos são **rápidos o suficiente** para qualquer aplicação que precise deles.

Entretanto, a performance dessas funções não irá concertar os problemas de uso errado de algoritmo ou estrutura de dados da sua aplicação. Sempre revisite seus algoritmos e estruturas de dados antes de fazer qualquer otimização.

---

¹: Auto-hospedar a biblioteca padrão (que é escrever a biblioteca padrão da VM de JS em JavaScript) é uma coisa comum entre as VMs modernas de JavaScript, e isso faz com que as bibliotecas padrão tenham as mesmas otimizações que são aplicadas em outro código (passar funções internamente funciona, por exemplo), sem ter a sobrecarga de se comunicar com a camada nativa.

---

#### Notas de rodapé

[[1] 1956 - Improve performance of array builtins - v8 - Monorail](https://bugs.chromium.org/p/v8/issues/detail?id=1956)