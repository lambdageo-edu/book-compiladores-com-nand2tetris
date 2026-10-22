# 7. A VM Hack (especificação)

Este capítulo é a especificação de referência da VM Hack — a representação intermediária baseada em pilha para a qual o compilador Jack (capítulos 9 e 10) vai gerar código, e que o VM Translator (capítulos 12 e 13) vai traduzir para Assembly Hack. Não há código de compilador aqui; é a "planta" que orienta o restante da Parte 2 e o VM Translator.

## Modelo de memória

Cada célula de memória da máquina Hack armazena 16 bits, usados para representar de forma uniforme três coisas: um inteiro em complemento de dois (-32768 a 32767), um booleano (`0` para falso, `-1` — todos os bits em 1 — para verdadeiro), ou um endereço de memória.

A memória é organizada em **oito segmentos**, cada um endereçado por índice a partir de 0:

| Segmento | Propósito | Usado pelo compilador Jack para |
|---|---|---|
| `local` | Variáveis locais da subroutine em execução | variáveis declaradas com `var` |
| `argument` | Argumentos recebidos pela subroutine | parâmetros da subroutine |
| `static` | Variáveis de classe (compartilhadas por todas as instâncias) | variáveis declaradas com `static` |
| `this` | Campos do objeto atual | variáveis declaradas com `field` |
| `that` | Elemento de array corrente | acesso `arr[i]` |
| `constant` | Pseudo-segmento somente leitura para constantes literais | literais inteiros |
| `pointer` | Duas células: `pointer 0` guarda o endereço-base de `this`, `pointer 1` o de `that` | ancorar `this` no objeto e `that` no elemento de array |
| `temp` | Oito células de uso livre, compartilhadas por toda a VM | valores descartados e auxiliares |

`this` e `that` são segmentos "móveis": não têm endereço fixo, e sim o endereço que estiver gravado em `pointer 0` e `pointer 1`. Gravar um endereço `a` em `pointer 0` faz `this i` passar a significar `RAM[a + i]`; o mesmo vale para `pointer 1` e `that i`:

```
push constant 8000
pop pointer 0        // this passa a começar em RAM[8000]
push constant 17
pop this 1           // RAM[8001] = 17
```

É esse mecanismo de endereçamento indireto que o compilador vai usar para acessar campos de objetos (via `this`) e elementos de arrays (via `that`).

Além dos segmentos, duas áreas adicionais completam o modelo: a **pilha** (*stack*), usada para armazenar valores temporários durante a avaliação de expressões e para implementar o protocolo de chamada de função (capítulo 13); e o **heap**, a região de RAM dedicada ao armazenamento de objetos e arrays, gerenciada dinamicamente pelas funções `Memory.alloc`/`Memory.deAlloc` do sistema operacional Jack.

## Comandos de acesso à memória

Apenas dois comandos movem dados entre a pilha e os segmentos:

```
push segmento índice   // empilha o valor de segmento[índice]
pop segmento índice    // desempilha o topo e armazena em segmento[índice]
```

`segmento` é um dos oito acima (exceto que não existe `pop constant`, já que constantes não são graváveis); `índice` é um inteiro não negativo.

```
push local 2       // empilha local[2]
push constant 10   // empilha o literal 10
pop argument 0      // desempilha o topo para argument[0]
```

## Comandos aritméticos e lógicos

Nove comandos operam sobre a pilha — sete binários (consomem dois valores do topo, empilham um resultado) e dois unários (consomem um, empilham um):

| Comando | Aridade | Semântica |
|---|---|---|
| `add` | binário | soma |
| `sub` | binário | subtração |
| `neg` | unário | negação aritmética |
| `eq` | binário | `-1` se iguais, `0` caso contrário |
| `gt` | binário | `-1` se o primeiro é maior |
| `lt` | binário | `-1` se o primeiro é menor |
| `and` | binário | E bit a bit |
| `or` | binário | OU bit a bit |
| `not` | unário | complemento bit a bit |

Não existem comandos de multiplicação ou divisão na VM — como veremos no capítulo 9, o compilador Jack traduz os operadores `*` e `/` em chamadas às funções `Math.multiply`/`Math.divide` da biblioteca padrão, não em comandos VM nativos.

## Controle de fluxo

Linguagens de alto nível têm `if`, `while`, `for` — a VM reduz tudo isso a três primitivas:

```
label rótulo       // declara uma posição nomeada no código
goto rótulo        // salto incondicional
if-goto rótulo     // desempilha o topo; se for verdadeiro (≠ 0), salta
```

Rótulos têm escopo de função — dois `label LOOP` em funções diferentes não colidem (o capítulo 13 detalha a qualificação `funcao$rótulo` usada pelo VM Translator para garantir isso na tradução final para Assembly). É com apenas essas três primitivas que o capítulo 9 vai mapear `if`/`else` e `while` de Jack.

## Funções

Três comandos governam todo o protocolo de chamada:

```
function nome nLocais   // declara uma função, com N variáveis locais (inicializadas em 0)
call nome nArgs         // chama uma função, passando os N argumentos já empilhados
return                  // desempilha um valor e devolve o controle ao chamador
```

Uma peça importante do design: **mesmo subroutines que não retornam valor útil precisam empilhar algo antes do `return`** — por convenção, `push constant 0`. É simetria pura: todo `call` espera encontrar um valor no topo da pilha ao retornar, então toda função — `void` ou não — precisa deixar exatamente um valor lá.

```
function Main.dobro 0
    push constant 2
    push argument 0
    call Math.multiply 2
    return

function Main.main 1
    push constant 5
    call Main.dobro 1
    pop local 0
    push constant 0
    return
```

O nome de uma função é sempre `Classe.nomeDaSubroutine` — é assim que construtores, métodos e funções (na terminologia Jack) se tornam, todos igualmente, "funções" no nível da VM. O mecanismo completo de `call`/`return` (salvar e restaurar o estado do chamador, o *stack frame*) é o assunto do capítulo 13; por ora, basta saber a interface: argumentos entram empilhados, o resultado sai empilhado.

## Um programa completo

Todo programa Jack começa executando `Main.main` — mais precisamente, a máquina começa por `Sys.init`, uma função do sistema operacional Jack que inicializa as demais classes do SO e então chama `Main.main` (o capítulo 13 mostra esse *bootstrap*). Um exemplo simples — somar de 1 a `n` e imprimir o resultado — ilustra push/pop, aritmética, controle de fluxo e chamada de função juntos:

```
function Soma.soma 2        // variáveis locais: sum (0), i (1)
    push constant 0
    pop local 0              // sum = 0
    push constant 1
    pop local 1              // i = 1
label LOOP
    push local 1
    push argument 0
    gt                        // i > n ?
    if-goto FIM
    push local 0
    push local 1
    add
    pop local 0               // sum = sum + i
    push local 1
    push constant 1
    add
    pop local 1               // i = i + 1
    goto LOOP
label FIM
    push local 0
    return

function Main.main 1
    push constant 10
    pop local 0               // n = 10
    push local 0
    call Soma.soma 1
    call Output.printInt 1
    pop temp 0                // descarta o valor devolvido por Output.printInt
    push constant 0
    return
```

O `pop temp 0` logo depois de `call Output.printInt 1` é o outro lado da convenção vista acima. Como toda função devolve um valor — inclusive as `void`, que devolvem 0 —, quem chama uma função apenas pelo efeito colateral precisa descartar esse valor, para que a pilha não acumule lixo.

## O mapeamento padrão de Jack sobre a VM

A VM não sabe nada de classes, objetos ou métodos. Para que programas Jack compilados separadamente (e o sistema operacional Jack, que é ele próprio um conjunto de arquivos `.vm`) funcionem juntos, todo compilador Jack → VM segue as mesmas convenções — o **mapeamento padrão**. Ele é parte da especificação, não da implementação: os capítulos 8 a 10 apenas o colocam em prática.

**Arquivos e subroutines.** Cada `Xxx.jack` é compilado para um `Xxx.vm` separado. Cada subroutine `yyy` da classe `Xxx` — seja `function`, `method` ou `constructor` — vira uma função VM chamada `Xxx.yyy`. Uma `function` ou um `constructor` com *k* parâmetros vira uma função VM de *k* argumentos; um `method` com *k* parâmetros vira uma função VM de *k* + 1 argumentos, porque `argument 0` é sempre uma referência ao objeto sobre o qual o método opera.

**Variáveis.**

| Variável Jack | Segmento VM |
|---|---|
| `var` (local da subroutine) | `local` |
| parâmetro da subroutine | `argument` |
| `static` | `static` do `.vm` da classe |
| `field` do objeto corrente | `this`, depois de apontar `pointer 0` para o objeto |
| elemento `arr[i]` | `that 0`, depois de apontar `pointer 1` para `arr + i` |

**Subroutines.**

- Um **método** começa ancorando `this` no objeto recebido: `push argument 0`, `pop pointer 0`.
- Um **construtor** começa alocando o objeto no heap com `Memory.alloc(n)`, em que *n* é o número de campos da classe, ancora `this` no endereço devolvido (`pop pointer 0`) e termina devolvendo esse endereço (`push pointer 0`, `return`).
- Funções e métodos `void` devolvem `constant 0`.

**Chamadas.** Quem chama empilha os argumentos e executa `call`. Se o alvo é um método, o primeiro valor empilhado é a referência ao objeto. Depois de um `do`, o valor devolvido é descartado com `pop temp 0`.

**Constantes.** `null` e `false` são `0`; `true` é `-1`, obtido com `push constant 1` seguido de `neg` (ou, equivalentemente, `push constant 0` seguido de `not`).

**Serviços do sistema operacional.** O sistema operacional Jack é formado por oito classes já compiladas — `Math.vm`, `Memory.vm`, `Screen.vm`, `Output.vm`, `Keyboard.vm`, `String.vm`, `Array.vm` e `Sys.vm` —, que precisam estar no mesmo diretório dos `.vm` gerados pelo compilador (o VM Emulator do Nand2Tetris já traz uma implementação embutida delas, então nesse caso não é preciso copiá-las). O compilador recorre a elas implicitamente:

| Construção Jack | Serviço usado |
|---|---|
| `x * y`, `x / y` | `Math.multiply`, `Math.divide` |
| constante `"abc"` | `String.new(3)` seguido de `String.appendChar` para cada caractere |
| `Classe.new(...)` (dentro do construtor) | `Memory.alloc(n)` |

## O que vem a seguir

Com a especificação da VM Hack e o mapeamento padrão em mãos, sabemos exatamente *onde* cada variável Jack precisa morar. O próximo capítulo constrói a estrutura que guarda essa informação durante a compilação — a **tabela de símbolos** —, e os capítulos 9 e 10 evoluem o `CompilationEngine` da Parte 1 para efetivamente gerar este código, expressão por expressão, comando por comando — fechando o compilador Jack → VM (Project 11) por completo. Os capítulos 12 e 13, na Parte 3, tratam do outro lado: como traduzir esse código VM para Assembly Hack.

Uma versão tabular desta especificação, para consulta rápida durante esses capítulos, está no [Apêndice B](../apendices/b-especificacao-vm.md).
