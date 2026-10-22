# 3. A linguagem Jack

Este capítulo apresenta a linguagem que vamos compilar ao longo de todo o livro: **Jack**, desenvolvida no projeto Nand2Tetris com um único objetivo — ser fácil de compilar sem deixar de ser uma linguagem orientada a objetos "de verdade", com classes, objetos, herança de sintaxe do Java e uma biblioteca padrão razoável. Nenhum código de compilador é escrito neste capítulo; ele é puramente sobre a linguagem-alvo que os capítulos 4, 5, 9 e 10 vão aprender a analisar e traduzir.

## Por que Jack é fácil de compilar

Jack tem uma sintaxe inspirada em Java, mas com uma lista deliberada de simplificações que removem justamente as partes de uma linguagem real que mais complicam um compilador:

| Característica | Jack | Motivo da escolha |
|---|---|---|
| Paradigma | Orientado a objetos, baseado em classes | Suficiente para ensinar tabela de símbolos com escopo e geração de código de objetos |
| Herança / polimorfismo | Não existem | Remove *dispatch* de métodos e resolução de tipos por hierarquia |
| Tipagem | Estática, mas fracamente verificada | O compilador não precisa (neste curso) implementar checagem de tipos completa |
| Precedência de operadores | **Não existe** — o compilador avalia da esquerda para a direita | Elimina toda a complexidade de tabelas de precedência na gramática de expressões |
| Tipos primitivos | Apenas `int`, `boolean`, `char` (sem `float`) | Simplifica geração de código e representação de valores (tudo cabe em uma palavra de 16 bits) |
| Campos de instância | Sempre privados | Não há modificadores de visibilidade para tratar |
| Métodos | Sempre públicos | Idem |
| Sobrecarga | Não existe (nomes de subroutine únicos dentro da classe, inclusive construtores) | Simplifica resolução de chamadas |

Cada uma dessas escolhas de projeto tem uma tradução direta em "uma linha a menos para o compilador se preocupar" — é uma lição de projeto de linguagem tanto quanto de compiladores: a complexidade de compilar uma linguagem é, em grande parte, uma escolha de projeto da própria linguagem.

## Um primeiro programa

```java
class Main {
    function void main() {
        var String nome;
        let nome = Keyboard.readLine("Qual o seu nome? ");
        do Output.printString("Ola ");
        do Output.printString(nome);
        do Output.println();
        return;
    }
}
```

Um programa Jack é uma **coleção de classes**, cada uma em seu próprio arquivo `NomeDaClasse.jack`. Para ser executável, uma das classes deve se chamar `Main` e conter uma função `main`. A estrutura geral de uma classe é:

```java
class NomeDaClasse {
    // variáveis de classe (static) e de instância (field)

    // construtor
    // funções (métodos de classe)
    // métodos (métodos de instância)
}
```

## Tipos e variáveis

Jack é **estaticamente tipada**: toda variável é declarada com um tipo que não muda durante a execução. Existem apenas três tipos primitivos:

| Tipo | Valores |
|---|---|
| `int` | inteiro de 16 bits, complemento de dois (-32768 a 32767) |
| `boolean` | `true` ou `false` (internamente, `true = -1`, `false = 0`) |
| `char` | um caractere Unicode |

Toda classe declarada pelo programador também é, automaticamente, um novo tipo — variáveis desse tipo armazenam **referências** (endereços no heap), não cópias. Isso significa que atribuir uma variável de classe a outra faz as duas apontarem para o **mesmo objeto**:

```java
var Point p1, p2;
let p1 = Point.new(5, 10);
let p2 = p1;              // p2 aponta para o MESMO objeto que p1
do p1.moveTo(20, 30);
do Output.printInt(p2.getX());  // imprime o X atualizado, não o original
```

Dois tipos compostos vêm prontos na biblioteca padrão: `String` (cadeia de caracteres, entre aspas duplas: `"Ola mundo"`) e `Array` (estrutura contígua e homogênea, alocada com `Array.new(tamanho)` e acessada com `arr[i]`).

### Escopo de classe: `field` e `static`

```java
class Point {
    field int x, y;       // variável de instância: uma cópia por objeto, sempre privada
    static int pointCount; // variável de classe: única, compartilhada por todas as instâncias
}
```

`field` (variável de instância) é sempre privado — o acesso de fora da classe exige métodos getter/setter explícitos — e só é visível nos métodos e no construtor, nunca nas `function`s, que não operam sobre um objeto. `static` (variável de classe) é única e compartilhada por todas as instâncias. Assim como `field`, só é visível dentro da classe — mas, ao contrário de `field`, pode ser usada também nas `function`s. Para expô-la a outras classes, é preciso uma subroutine de acesso, como uma `function int getPointCount()` chamada com `Point.getPointCount()`: a gramática de Jack não tem uma forma `Classe.variavel`.

### Escopo de subroutine: parâmetros e `var`

Parâmetros são declarados como em Java/C: `method void setX(int x)`. Variáveis locais usam a palavra reservada `var`, com duas restrições importantes que **não** existem em linguagens mais modernas:

1. Toda declaração `var` deve vir **no início** da subroutine, antes de qualquer comando.
2. Não é possível atribuir um valor no momento da declaração — `var int sum = 0;` é um erro sintático. A inicialização é sempre um comando `let` separado, depois de todas as declarações `var`.

```java
function void main() {
    var int sum;   // ok
    var int i;     // ok — ainda estamos na zona de declarações
    let sum = 0;   // primeira atribuição, depois de todas as declarações
    // var int j;  // ERRO: declaração fora do início da subroutine
}
```

Toda variável não inicializada explicitamente recebe um valor padrão: `0` para `int`, `false` para `boolean`, `null` para tipos de classe.

## Expressões: sem precedência de operadores

Esta é talvez a decisão de projeto mais marcante de Jack: **não existe precedência entre operadores**. A especificação da linguagem não define nenhuma prioridade (só garante que expressões entre parênteses são avaliadas primeiro) e deixa a ordem a cargo do compilador; o compilador de referência do Nand2Tetris, e o que construímos neste livro, avaliam da esquerda para a direita. Cabe ao programador usar parênteses para expressar a ordem desejada:

```java
let x = 2 + 3 * 4;     // = 20: avaliado como (2 + 3) * 4 — prefira explicitar
let x = 2 + (3 * 4);   // = 14
let x = (2 + 3) * 4;   // = 20
```

Essa escolha não é apenas didática para quem programa em Jack — ela é o que torna a gramática de expressões do capítulo 5 tão mais simples do que a de uma linguagem com precedência (não precisamos de uma cadeia `expr → term → factor → ...` codificando níveis de precedência: um único não-terminal `expression` já resolve tudo, como veremos).

Os operadores disponíveis, todos de um único caractere (mais uma simplificação a favor de um léxico trivial):

- Aritméticos binários: `+` `-` `*` `/`
- Aritmético unário: `-`
- Lógicos binários: `&` (e), `|` (ou)
- Lógico unário: `~` (não)
- Relacionais: `<` `>` `=` (observe: **não existem** `<=` nem `>=`, justamente porque exigiriam dois caracteres)

Literais válidos em uma expressão: inteiros sem sinal (`1256`), strings (`"Ola"`), `true`, `false`, e `null` (referência não definida). Uma expressão também pode conter chamadas de subroutine: `4 + (5 * Math.sqrt(16))`.

## Comandos

Jack tem exatamente cinco tipos de comando — um conjunto propositalmente mínimo:

- **`let`** — atribuição: `let a = 10;` ou, para elementos de array, `let a[i] = 10;`
- **`do`** — chama uma subroutine que não retorna valor útil (uma `void`), descartando o resultado: `do Output.printString("...");`
- **`if` / `else`** — a única estrutura de seleção. Diferente de C/Java, as chaves são **obrigatórias**, mesmo para um único comando:

  ```java
  if (a > b) {
      return a;
  } else {
      return b;
  }
  ```

- **`while`** — a única estrutura de iteração, também com chaves obrigatórias:

  ```java
  function int fatorial(int n) {
      var int p, i;
      let p = 1;
      let i = 1;
      let n = n + 1;
      while (i < n) {
          let p = p * i;
          let i = i + 1;
      }
      return p;
  }
  ```

- **`return`** — devolve um valor (ou nada, para subroutines `void`).

Não existem `for`, `switch`, `break` ou `continue` — tudo se expressa com `while` e `if`. De novo: menos construções sintáticas para o parser tratar.

## Construtores, métodos e funções

Uma classe pode declarar três tipos de subroutine, cada uma com um papel distinto:

| Palavra-chave | Papel | Recebe `this` implícito? |
|---|---|---|
| `constructor` | Cria e inicializa uma nova instância; por convenção, chama-se `new` e retorna `this` | Não (ele o cria) |
| `method` | Opera sobre uma instância existente | Sim |
| `function` | Opera apenas sobre a classe (equivalente a um método estático) | Não |

```java
class Point {
    field int x, y;
    static int pointCount;

    constructor Point new(int ax, int ay) {
        let x = ax;
        let y = ay;
        let pointCount = pointCount + 1;
        return this;
    }

    method int getX() {
        return x;
    }
}
```

Um construtor pode ter qualquer nome — por convenção, `new` —, e uma classe pode ter mais de um, desde que com nomes diferentes (não há sobrecarga). O tipo de retorno de um construtor é sempre a própria classe. A chamada usa o nome completo, `NomeDaClasse.new(...)`:

```java
var Point pt;
let pt = Point.new(10, 20);
```

Métodos são chamados sobre uma instância (`objeto.metodo(...)`); funções, sobre a classe (`Classe.funcao(...)`). Essa distinção sintática — que não existe em Java, onde tudo é `objeto.metodo` ou `Classe.metodoEstatico` mas o compilador resolve por assinatura — é explícita em Jack através da palavra-chave de declaração, o que facilita bastante a análise semântica no capítulo 8.

Dentro da própria classe vale uma regra a mais: um **método** pode ser chamado sem qualificador — `metodo(...)` significa "chame `metodo` sobre o objeto corrente", o equivalente a `this.metodo(...)` em Java. **Funções e construtores**, ao contrário, são sempre chamados pelo nome completo, mesmo de dentro da classe:

```java
class Point {
    // ...
    method void moveTo(int nx, int ny) {
        do erase();                                  // método de Point, sobre este objeto
        let x = nx;
        let y = ny;
        do draw();                                   // idem
        do Output.printInt(Point.getPointCount());   // função: sempre com o nome da classe
        return;
    }
}
```

O compilador do capítulo 10 depende dessa regra: uma chamada sem qualificador é sempre tratada como chamada de método sobre `this`.

## Biblioteca padrão (Jack OS)

Jack não tem tipos ou operações embutidas além dos três primitivos — tudo o mais (strings, arrays, matemática, entrada/saída, gráficos, gerência de memória) é fornecido por oito classes de uma biblioteca padrão, o "sistema operacional Jack":

| Classe | Papel |
|---|---|
| `Math` | Operações aritméticas que a linguagem não tem nativamente — `multiply`, `divide`, `min`, `max`, `sqrt`. O compilador traduz `*` e `/` em chamadas a `Math.multiply`/`Math.divide`. |
| `String` | Construção e manipulação de cadeias de caracteres (`length`, `charAt`, `appendChar`, `intValue`, ...). |
| `Array` | Alocação de arrays (`Array.new(tamanho)`); o acesso `arr[i]` é sintaxe da linguagem, não um método. |
| `Output` | Impressão de texto na tela (23 linhas × 64 colunas): `printString`, `printInt`, `println`. |
| `Screen` | Gráficos em nível de pixel (512×256): `drawPixel`, `drawLine`, `drawRectangle`, `drawCircle`. |
| `Keyboard` | Leitura de teclado: `readChar`, `readLine`, `readInt`. |
| `Memory` | Acesso direto à RAM e gerência de heap: `peek`, `poke`, `alloc`, `deAlloc`. |
| `Sys` | Serviços do sistema: `halt`, `error`, `wait`. |

Perceba algo importante para os capítulos de geração de código (9 e 10): `Memory.alloc` é quem de fato aloca espaço para um novo objeto quando você chama `Classe.new(...)` — o `constructor` não faz mágica, ele delega para essa função do sistema operacional Jack.

## A gramática de Jack (visão geral)

O front-end de Jack (léxico + sintático, capítulos 4 e 5) trabalha sobre uma gramática livre de contexto pequena e sem ambiguidade — uma consequência direta das simplificações de projeto vistas acima (sem precedência de operadores, sem herança, sem sobrecarga). A notação usada segue a convenção do próprio Nand2Tetris: `'x'` para terminais literais, `x` (sem aspas) para não-terminais, `()` para agrupar, `x|y` para alternativas, `x?` para "zero ou um", `x*` para "zero ou mais".

**Elementos léxicos** (5 categorias de token — o assunto do capítulo 4):

| Terminal | Regra |
|---|---|
| `keyword` | `class`, `constructor`, `function`, `method`, `field`, `static`, `var`, `int`, `char`, `boolean`, `void`, `true`, `false`, `null`, `this`, `let`, `do`, `if`, `else`, `while`, `return` |
| `symbol` | `{ } ( ) [ ] . , ; + - * / & | < > = ~` |
| `integerConstant` | Um número decimal entre 0 e 32767 (o `-` de um número negativo é o operador unário) |
| `stringConstant` | Sequência de caracteres Unicode, sem aspas duplas nem quebra de linha |
| `identifier` | Sequência de letras, dígitos e `_`, não iniciando com dígito |

**Elementos sintáticos** (o assunto do capítulo 5):

| Não-terminal | Regra |
|---|---|
| `class` | `'class'` className `'{'` classVarDec* subroutineDec* `'}'` |
| `classVarDec` | (`'static'`\|`'field'`) type varName (`','` varName)* `';'` |
| `type` | `'int'` \| `'char'` \| `'boolean'` \| className |
| `subroutineDec` | (`'constructor'`\|`'function'`\|`'method'`) (`'void'`\|type) subroutineName `'('` parameterList `')'` subroutineBody |
| `parameterList` | ((type varName) (`','` type varName)*)? |
| `subroutineBody` | `'{'` varDec* statements `'}'` |
| `varDec` | `'var'` type varName (`','` varName)* `';'` |
| `statements` | statement* |
| `statement` | letStatement \| ifStatement \| whileStatement \| doStatement \| returnStatement |
| `letStatement` | `'let'` varName (`'['` expression `']'`)? `'='` expression `';'` |
| `ifStatement` | `'if'` `'('` expression `')'` `'{'` statements `'}'` (`'else'` `'{'` statements `'}'`)? |
| `whileStatement` | `'while'` `'('` expression `')'` `'{'` statements `'}'` |
| `doStatement` | `'do'` subroutineCall `';'` |
| `returnStatement` | `'return'` expression? `';'` |
| `expression` | term (op term)* |
| `term` | integerConstant \| stringConstant \| keywordConstant \| varName \| varName `'['` expression `']'` \| subroutineCall \| `'('` expression `')'` \| unaryOp term |
| `subroutineCall` | subroutineName `'('` expressionList `')'` \| (className\|varName) `'.'` subroutineName `'('` expressionList `')'` |
| `expressionList` | (expression (`','` expression)*)? |
| `op` | `+` \| `-` \| `*` \| `/` \| `&` \| `|` \| `<` \| `>` \| `=` |
| `unaryOp` | `-` \| `~` |
| `keywordConstant` | `true` \| `false` \| `null` \| `this` |

A gramática completa, com todos os detalhes, está no [Apêndice A](../apendices/a-gramatica-jack.md) — vamos voltar a ela linha por linha nos capítulos 4 e 5, à medida que cada regra vira um método do tokenizador ou do parser.

## Ambiente de desenvolvimento

Para experimentar Jack sem instalar nada, use a IDE online do próprio projeto: [nand2tetris.github.io/web-ide/compiler](https://nand2tetris.github.io/web-ide/compiler). Para instalação local do conjunto de ferramentas oficial (compilador de referência, VM Emulator, CPU Emulator), veja o [Apêndice D](../apendices/d-ambiente-e-ferramentas.md).

## O que vem a seguir

Com a linguagem-alvo em mãos, os próximos dois capítulos constroem o front-end do nosso compilador: primeiro o `JackTokenizer` (capítulo 4), que transforma o texto-fonte na sequência de tokens da tabela léxica acima; depois o `CompilationEngine` (capítulo 5), que reconhece a estrutura sintática descrita pela gramática. Ambos, no espírito deste livro, construídos com você, método a método.
