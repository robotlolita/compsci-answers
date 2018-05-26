# Como funciona a coerção de tipos em JavaScript?

> Resposta originalmente para a pergunta do Quora ["Why is ({}) true but ({} ==true) false in Javascript?"](https://www.quora.com/Why-is-true-but-true-false-in-Javascript/answer/Quildreen-Motta)


Por que algo como `({})` é `true`, mas `({} == true)` é `false` em JavaScript? Bem, não é exatamente o que acontece, mas o motivo é "polimorfismo de coerção[1]". Em outras palavras, os operadores em JavaScript aceitam um número menor de valores do que os valores possíveis na linguagem. E ao invés de lançar uma exceção ou parar o programa quando você usa um valor que o operador não aceita, a operação vai tentar converter o valor para um dos "tipos" aceitos, usando algumas regras (descritas na especificação).

> **NOTA**  
> Nunca passe objetos para operadores em JS, a não ser que aquele operador tenha sido projetado para trabalhar com objetos (poucos operadores são!), e use tipos consistentes. Se um lado de um operador binário está usando um número, por exemplo, o outro lado também deve ser um número. Do contrário você vai ter um programa que é difícil de prever, e de quebra alguns problemas de segurança.

Operadores em JavaScript geralmente funcionam em cima de valores primitivos. Ou seja: números, strings, e booleanos. Adicionalmente a linguagem possui alguns "não-valores" que são especiais: `null` e `undefined`. Existem também objetos, que podem ser qualquer outra coisa (arrays, datas, funções, etc).

Por exemplo, o operador `<` funciona em cima de números, então os dois argumentos dessa operação precisam ser números. Mas o que acontece se você passa algo que não é um número para esse operador, como em `[] < 2`? Bem algumas coisas poderiam acontecer:

- A linguagem poderia te dizer que `[] < 2` não faz sentido, porque `[]` não é um número, e sim uma array vazia. Isto é algo que as pessoas consideram uma escolha razoável, e é o que você vê em várias outras linguagens (por exemplo, Java);

- A linguagem poderia dizer que `[]` ou `2` (ou ambos) têm que decidir o que `<` significa para eles. Em linguagens orientadas a objetos aonde os operadores são apenas chamadas de métodos (e.g.: Python, Ruby, Lua, Smalltalk, Scala, CLOS, ...) é isso que acontece. O que o operador `<` faz vai ser decidido pelos argumentos desse operador (geralmente só o argumento da esquerda). Daí `[] < 2` podereia significar `"[] tem menos do que 2 elementos?"` enquanto `1 < 2` poderia significar `"O número 1 é menor que o número 2?"`, por exemplo.

- A linguagem podereia tentar converter os operandos para um tipo comum que a operação aceite, e daí executar a operação. Nesse caso os dois operandos poderiam ser convertidos para números. É isso que JavaScript faz.

Em princípio, isso não deveria ser tão complicado quanto as pessoas dizem. Operadores funcionam em Strings ou Números, então qualquer valor que não seja de um desses tipos precisa dizer como pode ser convertido para uma String, e como pode ser convertido para um Número. Eu deixei booleanos de lado aqui porque frequentemente eles funcionam como "tipos especiais" de números.

O primeiro exemplo da pergunta inicial, se assumirmos um contexto como `if ({}) { ... }`, é simples: ToBoolean[2] é executado no valor, e para objetos o resultado dessa operação (interna) é sempre `true`.

Vamos ver o outro exemplo em mais detalhes abaixo. Mas lembre-se: toda vez que você pensar "nossa, isso é bem ruim hein?", não se preocupe muito. Vai ficar pior :>


## A História de Dois Métodos: `.valueOf()` e `.toString()`

Okay, eu disse que em princípio cada valor diz para a linguagem como vai se converter para Strings e para Números, certo? Bem, em JavaScript isso é feito através de dois métodos em cada objeto. O método `.toString()` define como o valor vai ser convertido para uma String. E o método `.valueOf()` diz como o valor vai ser convertido para número.

```js
const x = {
  valueOf(){ return 3 }
};
 
  x < 2;
= x.valueOf() < 2
= 3 < 2
= false;
 
const y = {
  toString(){ return "Hello, " }
}
 
  y + "world"
= y.toString() + "world"	// veja as notas sobre + no final do artigo
= "Hello, " + "world"
= "Hello, world"
```

`x` define o método `valueOf`, que é chamado quando o valor precisa ser convertido para um número. Enquanto isso, `y` define o método `toString`, que é chamado quando o valor precisa ser convertido para uma string. Até aí sem muitos problemas.

Mas é aqui que as coisas começam a ficar mais complicadas. JavaScript permite que você implemente só um desses dois métodos em um objeto. Existem regras na especificação para converter Strings para Números, e Números para Strings. Então se você implementar só um dos métodos, sempre que uma operação precisar do método que não está implementado, ela usa o método implementado, e então converte o resultado para o tipo necessário.

A gente pode ver isso mudando um pouco os valores no exemplo anterior:

```js
  y < 2
= y.toString() < 2		// y não implementa .valueOf
= "Hello, " < 2
= Number("Hello, ") < 2
= NaN < 2
= false
 
// Como é `NaN`, `y > 2` e `y == 2` são ambos `false`.
 
  x + "world"
= x.valueOf() + "world"		// veja as notas sobre + no final do artigo
= 3 + "world"
= (3).toString() + "world"
= "3world"
```

Note que, como `.valueOf()` e `.toString()` são funções arbitrárias definidas pelo usuário, elas podem retornar valores que não são primitivos! Por causa disso JavaScript vai tentar os dois métodos se você definir os dois. Se a expressão demanda uma String, JavaScript vai tentar `.toString()` primeiro, e se não der certo usar o `.valueOf()`. Se a expressão requer um Número a ordem é inversa: `.valueOf()` depois `.toString()`.

Se nenhum desses métodos estiver disponível, ou nenhum dos dois retornar um valor que é um primitivo, JavaScript pára a execução do programa e informa que não pode executar a operação. Vamos ver isso com alguns exemplos:

```js
const a = { valueOf(){ return [] }, toString(){ return '1' } };
 
  a + 2
= a.valueOf() + 2
= [] + 2			// oops, essa expressão é ignorada
= a.toString() + 2
= "1" + 2
= "12"				// String sempre tem prioridade em +
 
const b = { valueOf(){ return 1 }, toString(){ return [] } };
const array = [1, 2];
 
  array[b]	
= array[b.toString()]
= array[[]]				// oops, essa expressão é ignorada
= array[b.valueOf()]
= array[1]
= array[(1).toString()]
= array['1']
= 2
 
// Objectos comuns herdam `.valueOf` e `.toString` de
// Object.prototype. Desde a versão de 2009 (ES5) podemos
// criar um objeto que não herda de nenhum outro objeto.
const c = Object.create(null);
 
c + 2	   // => Error: Cannot convert object to primitive value
array[c] // => Error: Cannot convert object to primitive value
 
// Definindo valores que não são funções para essas propriedades tem o
// mesmo efeito
const d = { valueOf: null, toString: null };
d + 2	   // => Error: Cannot convert object to primitive value
array[d] // => Error: Cannot convert object to primitive value
```

Okay, isso foi bastante coisa, né? Eu vou admitir que menti um pouquinho sobre a simplicidade do conceito. Mas, como eu disse: vai piorar.

## `@@toPrimitive` ao resgate...?

Em versões anteriores da linguagem, todas essas conversões de objeto para primitivos eram tratadas pela operação interna `ToPrimitive`[2]. Essa operação é chamada com uma "dica" de qual tipo a expressão espera (`string` ou `number`), o que tem um efeito em qual dos métodos a operação vai chamar primeiro: `.valueOf()` ou `.toString()`. Com a adição de Símbolos[3] na linguagme, a especificação agora permite que os usuários definam um comportamento diferente para essa operação interna através do símbolo `@@toPrimitive`.

Embora isso torne a conversão de tipos mais complexa, ao menos se você retornar um valor que não é um primitivo dessa operação, JavaScript termina o programa com um erro de tipo, ao invés de tentar chamar `.valueOf` ou `.toString`. É uma dor de cabeça a menos.

Quando `@@toPrimitive` é chamado ele vai receber um argumento do tipo String, que pode ser:

- `"string"`, se o método deve retornar uma representação textual do valor (por exemplo, num acesso de propriedade);
- `"number"`, se o método deve retornar uma representação numérica do valor (por exemplo, em operadores relacionais);
- e `"default"`, se a linguagem não tem idéia de qual tipo usar, e fica a cargo do método decidir qual tipo retornar (operadores que podem trabalhar com mais de um tipo, como `==` e `+` fazem isso).

```js
const x = { 
  [Symbol.toPrimitive](hint) {
    return hint === "default" ?  "oops, o resultado é "
    :      hint === "number"  ?  42
    :      hint === "string"  ?  "tails"
    : (() => { throw new TypeError(`Dica desconhecida ${hint}`) })();
  }
};
 
x + 1;						        // => "oops, o resultado é 1"
({ tails: "dang" })[x]		// => "dang"
x > 41;						        // => true
```

E com isso a gente cobre a maior parte do conceito.

Mas, obviamente, a coisa fica bem pior.


## Como você sabe qual método vai ser chamado?

Você não sabe.

E eu realmente quero dizer isso: você. não. tem. como. saber.

Essa seção pode te dar alguma base para analisar o porquê das coisas darem errado quando operadores estão envolvidos, mas você não deve usar não-primitivos como argumentos para operadores (quando eles não esperam um objeto). É imprevisível, e um risco grande de segurança!

1. Se nenhum dos três métodos estiverem definidos no objeto (o melhor caso), então usar o objeto em qualquer operador esperando um primitivo vai sempre ser um `TypeError`.

2. Se apenas `@@toPrimitive` for definido, então esse método é sempre chamado em VMs mais recentes de JavaScript (que implementam ES2015), mas em VMs antigas será sempre um `TypeError`.

3. Se apenas `.toString()` ou `.valueOf()` forem definidos, aquele método sempre será chamado. Isso é previsível, mas você não tem como saber o contexto em que o seu valor está sendo usado dentro desses métodos para retornar uma representação adequada.

4. Se ambos `.toString()` e `.valueOf()` forem definidos, então o método que será chamado depende do oeprador *e* dos outros operandos. É difícil de prever, e se você retornar representações diferentes pode acabar com conversões inconsistentes (ver a tabela de operadores no final do artigo).

5. Se os três métodos forem definidos, então `@@toPrimitive` vai ser chamado em VMs mais recentes, e `.toString()` ou `.valueOf()` vai ser chamado em VMs antigas. Você tem que se preocupar não só com as inconsistências entre `.valueOf()` e `.toString()` mas também se `@@toPrimitive` vai ser chamado nas VMs que você suporta.

Você pode pensar que poderia evitar esse problema simplesmente não implementando qualquer desses três métodos em seus objetos, mas tem outros problemas com esse pensamento:

- A maioria dos objetos que seu programa vai trabalhar com não são objetos definidos por você. Isso inclui todos os objetos built-in. Todos esses objetos vai implementar pelo menos um dos três métodos acima.

- Algumas ferramentas (REPLs) e bibliotecas não vão funcionar a não ser que você implemente pelo menos `.toString()` ou `.valueOf()`. Não é apenas uma questão de passar o primitivo correto para elas também, porque isso costuma acontecer muito mais tarde, e a chamada que depende desse comportamento é interna na biblioteca. É claro que você vai ter uma vida melhor se não usar bibliotecas e ferramentas assim, se puder--elas provavelmente não funcionarão com objetos que não herdam de `Object.prototype` de qualquer forma.

- As capabilidades de reflection e meta-programação em JavaScript permitem que qualquer pessoa modifique qualquer método em qualquer objeto. A qualquer momento. A linguagem também permite modificar de qual objeto um objeto herda, o que pode causar comportamentos bem confusos! Monkey-patching não é algo raro, e shims/polyfills também fazem isso, e algumas vezes até mesmo com comportamento que é diferente da especificação.

Para mostrar como isso pode acontecer, vamos a mais um exemplo. O objeto built-in `Array.prototype` define sua própria implementação de `.toString()`, e herda de `Object.prototype` que define sua própria implementação de `.valueOf()`--nesse caso o método apenas retorna o próprio objeto.

```js
var a = [1];
 
  a + 2
= a.valueOf() + 2		  // Object.prototype.valueOf.call(a) = a
= a.toString() + 2		// Array.prototype.toString.call(a) = '1'
= "1" + 2
= "12"
 
// Podemos alterar Object.prototype.valueOf
Object.prototype.valueOf = function() {
  return 42;
}
 
  a + 2
= a.valueOf() + 2		// Object.prototype.valueOf.call(a) = 42
= 42 + 2
= 44
 
// E podemos alterar qual objeto uma array via herdar
const myArray = Object.create(Array.prototype);
myArray.valueOf = function(){ return 'hello' };
Object.setPrototypeOf(a, myArray);
 
// `a` agora herda de `myArray`,
// que herda de `Array.prototype`,
// que herda de `Object.prototype`,
// que não herda de nenhum objeto ([[Prototype]] = null)
  a + 2
= a.valueOf() + 2		// myArray.valueOf.call(a) = 'hello'
= "hello" + 2
= "hello2"
```

- Enquanto é normal que alguém tentaria evitar esses tipos de Monkey-Patching hoje em dia (costumávamos fazer isso bastante alguns anos atrás, entretanto), códigos maliciosos podem usar isso para explorar vulnerabilidades em código que parece tão inocente quanto `a + 1`. Existe um problema maior aqui com linguagens mainstream sem mecanismos apropriados de segurança (como Object-Capability Security[4]), mas isso é assunto para outra hora.

Em suma, *sempre* converta seus valores explícitamente. Não dependa de operadores fazendo isso para você. TypeScript e Flow podem te ajudar a encontrar esses problemas no seu próprio código se você dedicar um pouco de tempo anotando suas funções com tipos.


## Operadores e Regras de Conversão

Os únicos operadores que não convertem seus operandos em JavaScript são o operador de igualdade estrita (===), `new`, `super`, `yield`, `delete`, `void`, `typeof`, atualização de variável (=), e o operador vírgula (`,`). Todas as outras construções e operadores na linguagem vão tentar converter valores para uma forma primitiva. Algumas dessas conversões são simples (e.g.: chamando `ToBoolean`), a maioria não é.

Uma última nota antes do sumário de operadores: `ToString` é a operação `ToPrimitive` com a dica `string`, seguida de uma conversão do resultado para uma string. `ToNumber` é o mesmo, mas com a dica `number` e uma conversão para número.

```js
// Template Strings em uma tag
`some ${x}`	=> "some ".concat(ToString(x))
 
// Acesso de propriedades
object[x]	=> object[ToString(x)]
 
// Expressões de atualização de variável
x++, ++x		=> x = ToNumber(x) + 1
x--, --x		=> x = ToNumber(x) - 1
 
// Operadores unários
-x		=> -ToNumber(x)
+x		=> ToNumber(x)
~x		=> ToNumber(x)
!x		=> ToBoolean(x)   // (para objetos, sempre true)
 
// Operadores matemáticos
x ** y, x * y, x / y, x % y, x - y	
=> ToNumber(x), ToNumber(y)
 
// Operador additivo
x + y	=> ToPrimitive(x), ToPrimitive(y)
      : se um dos primitivos é uma string, `ToString()` é chamado neles
      : do contrário, `ToNumber()` é chamado neles
 
// É importante notar que quando ToPrimitive é chamado sem
// uma dica (como é o caso aqui), o valor padrão para a dica
// é `"number"`, que significa que a operação vai tentar invocar
// `.valueOf()` primeiro, se o método existir, mesmo se um dos
// operadores já for uma string primitiva!


// Operadores de movimentação de bits
x << y, x >> y, x >>> y		=> ToNumber(x), ToNumber(y)
 
// Operadores relacionais
x < y, x <= y, x > y, x >= y	=> ToNumber(x), ToNumber(y)
 
// Testes de instância
a instanceof b	=> ToBoolean(b[Symbol.hasInstance](a))
 
// Testes de existência de propriedade
a in b	=> ToPrimitive(a, "string")
        :  Se um símbolo não for retornado, `ToString()` é chamado no valor
 
// Operadores de igualdade
a == b, a != b
 
// Se forem do mesmo tipo, o mesmo que ===
// Se todos forem objetos, o mesmo que ===
// Se todos forem primitivos, `ToNumber()` é chamado em valores que não são números
// Se um for objeto, converte para primitivo (`ToPrimitive(x)`), e então para um
//   número, a não ser que um dos operandos se torne uma string depois da conversão

// Operadores de combinação de bits
a & b, a | b, a ^ b		=> ToNumber(a), ToNumber(b)
 
// Operadores lógicos
a && b, a || b 			=> ToBoolean(a), ToBoolean(b)
a ? b : c				    => ToBoolean(a)
```

Além de operadores, a maioria das funções na biblioteca padrão de JavaScript vai converter os argumentos para um tipo específico. Por exemplo, `Number("300")` vai chamar `ToNumber()` em seu argumento, e `"hello".endsWith(["lo"])` vai converter a array para uma string (e assumindo que as funções built-in não tiverem sido modificadas, retornar `true`).

Algumas construções sintáticas também convertem valores. Aonde valores booleanos são esperados, os valores vão passar pela operação `ToBoolean`. Isso quer dizer que `if ({}) { ... }` realmente significa `if (ToBoolean({})) { ... }`.

- - -

Ufa. Bem, agora podemos explicar os dois exemplos da primeira pergunta desse artigo!

Vários operadores usam `ToBoolean` para converter seus operandos para um booleano. Aonde apenas booleanos fazem sentido, isso não é tão ruim. `ToBoolean` não é uma operação configurável, então para objetos o retorno é sempre `true`.

```js
  if ({}) { ... }
= if (ToBoolean({})) { ... }
= if (true) { ... }
= ...
 
  {} || {}
= ToBoolean({}) || ToBoolean({})
= true || true
= true
```

Com o operador de igualdade abstrata (==) as coisas não são tão felizes. Assumindo que as funções e tipos built-in não tenham sido modificadas, e que o argumento seja um objeto comum, a gente tem essa execução:

```js
  ({}) == true
= ({}) == ToNumber(true)
= ({}) == 1
= ToPrimitive({}, "default") == 1
= ToNumber({}.valueOf()) == 1
= // falha porque {}.valueOf() returns {}
= ToNumber({}.toString()) == 1
= ToNumber("[object Object]") == 1
= NaN == 1
= false
```

Mas isso poderia ser bem diferente se em qualquer lugar daquele objeto ou sua cadeia heranças um método `@@toPrimitive` fosse definido. Ou mesmo `.valueOf()` ou `.toString()`.

```js
Object.prototype[Symbol.toPrimitive] = function(hint) {
  if (hint === "default") { return 1 }
  else { return this.toString() }
}
 
  ({}) == true
= ({}) == ToNumber(true)
= ({}) ==  1
= ToPrimitive({}, "default") == 1
= ToNumber(Object.prototype[Symbol.toString].call({})) == 1
= ToNumber(1) == 1
= 1 == 1
= true
```

- - -

#### Footnotes

[1] Polimorfismo de Coerção é o nome dado por Cardelli para o que operadores em JavaScript fazem. Para mais detalhes você pode ver [minha resposta do Quora sobre polimorfismo](https://www.quora.com/Object-Oriented-Programming-What-is-a-concise-definition-of-polymorphism/answer/Quildreen-Motta) (em inglês).

[2] ToBoolean, ToNumber, ToString, e ToPrimitive são operações internas definidas na [especificação de JavaScript](https://tc39.github.io/ecma262/#sec-toboolean).

[3] Símbolos são objetos que podem ser usados como nome de propriedade. Esses nomes usam uma comparação de referência, de forma que símbolos podem definir propriedades únicas sem a possibilidade de colisão. [A especificação define vários símbolos comuns](https://tc39.github.io/ecma262/#table-1).

[4] Object-Capability Security é um conceito de segurança aonde o sistema define permissões baseando-se nos objetos que podem ser acessados. Dessa forma cada objeto no sistema tem uma lista limitada de objetos que podem acessar (geralmente concedida por objetos com mais permissões). [Minha resposta do Quora](https://www.quora.com/What-are-some-advanced-concepts-in-programming-that-most-average-programmers-have-never-heard-of/answer/Quildreen-Motta) (em inglês) dá uma introdução sobre o conceito, e a tese de doutorado do Mark Miller, [Towards a Unified Approach to Access Control and Concurrency Control](http://www.erights.org/talks/thesis/) faz uma exposição detalhada do conceito.
