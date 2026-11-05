# 12. VM Translator — Parte 1: aritmética e memória

Com o compilador Jack → VM completo (capítulos 9 e 10), fechamos a metade "de cima" da nossa cadeia de tradução. Este capítulo começa a outra metade: o **VM Translator**, que traduz código VM (a IR especificada no capítulo 7) para Assembly Hack — a linguagem especificada no capítulo 11, que o Assembler (capítulo 15) finalmente converte em código de máquina.

Nesta parte do livro, o estilo muda: em vez de código construído junto com você, capítulo a capítulo (como fizemos com o `JackTokenizer` e o `CompilationEngine`), o VM Translator e o Assembler são apresentados como **teoria + especificação + exercício guiado** — você implementa como atividade prática, comparando sua saída com os testes oficiais do Nand2Tetris. A razão é de escopo: VM Translator e Assembler não são "o compilador Jack" propriamente — são a infraestrutura de back-end que o sustenta.

## Por que o VM Translator é um tradutor, não um compilador

Note a diferença de natureza: o `CompilationEngine` compila uma linguagem de alto nível (Jack) para uma representação de mais baixo nível (VM) — uma tradução "para baixo" na hierarquia de abstração. O VM Translator, por sua vez, traduz uma linguagem-fonte (código VM) para **outra** linguagem de baixo nível (Assembly Hack) — ambas já próximas da máquina. É uma arquitetura em três estágios:

```
Código Jack (alto nível)
    ↓ CompilationEngine (capítulos 9-10)
Código VM (intermediário)
    ↓ VM Translator (capítulos 12-13)
Assembly Hack (baixo nível)
    ↓ Assembler (capítulo 15)
Código de máquina Hack
```

O trabalho do tradutor é, comando a comando, **realizar a abstração da VM sobre uma máquina que não a tem**: o Hack não tem pilha, não tem segmentos, não tem `add` de dois operandos na memória — tem só dois registradores (`A` e `D`) e uma RAM. Tudo o que a VM promete precisa ser construído com esses recursos.

## Revisão: a máquina de pilha que vamos implementar

Antes de traduzir, vale relembrar *o que* estamos implementando (a especificação completa está no capítulo 7). A VM é uma **máquina de pilha**: toda operação lê seus operandos do topo da pilha e deixa ali o resultado. O comando `x = 17 + 19`, por exemplo, vira:

```
push constant 17     // pilha: 17
push constant 19     // pilha: 17 19
add                  // pilha: 36
pop local 0          // pilha: (vazia)   x = 36
```

E qualquer expressão, por mais complexa, se reduz a uma sequência desse tipo — por exemplo, `d = (2 - x) + (y + 9)`, com `x`, `y` e `d` nas posições 0, 1 e 2 do segmento `local`:

```
push constant 2
push local 0         // x
sub                  // 2 - x
push local 1         // y
push constant 9
add                  // y + 9
add                  // (2 - x) + (y + 9)
pop local 2          // d = ...
```

Repare que esse código usa só dois tipos de comando: **acesso à memória** (`push`/`pop` num segmento) e **aritmética/lógica** (`add`, `sub`, ...). São exatamente esses dois grupos — os comandos que não atravessam fronteiras de função — que este capítulo traduz. Controle de fluxo e funções ficam para o capítulo 13.

Também vale relembrar *por que* existem oito segmentos: cada um corresponde a um tipo de variável de Jack, o que permite ao compilador esquecer os nomes e trabalhar só com "segmento + índice":

| Segmento | O que guarda | Exemplo em Jack |
|---|---|---|
| `argument` | parâmetros da subrotina corrente | `x`, `y` em `function int bar(int x, int y)` |
| `local` | variáveis locais (`var`) | `var int c;` |
| `static` | variáveis `static` da classe | `static int s1;` |
| `this` | campos (`field`) do objeto corrente | `field int f;` |
| `that` | elementos de um array | `a[i]` |
| `constant` | constantes (segmento virtual, só leitura) | `17` |
| `pointer` | as bases de `this` e `that` | usado por construtores, métodos e arrays |
| `temp` | rascunho do compilador | `pop temp 0` após um `do` |

Na classe `Foo` abaixo, por exemplo, a atribuição `let c = s1 + y;` compila para `push static 0`, `push argument 1`, `add`, `pop local 2` — os nomes desaparecem, e só os segmentos (definidos pelo **tipo** da variável) e os índices (definidos pela **ordem** de declaração) restam:

```
class Foo {
    static int s1, s2;
    function int bar(int x, int y) {
        var int a, b, c;
        let c = s1 + y;
        ...
    }
}
```

## Modelo de memória: Hack

O Hack tem uma memória de instruções (ROM) e um espaço de endereços de dados de 32K palavras de 16 bits, do qual só os endereços 0 a 24576 são usados: 16K de RAM propriamente dita, 8K mapeados na tela e um endereço para o teclado. Os primeiros endereços da RAM têm papéis pré-definidos, que sustentam todo o mapeamento de segmentos VM → memória real — o chamado **mapeamento padrão** da VM sobre o Hack:

| Endereço RAM | Registrador | Papel |
|---|---|---|
| `RAM[0]` | `SP` | Ponteiro de topo da pilha |
| `RAM[1]` | `LCL` | Endereço-base do segmento `local` |
| `RAM[2]` | `ARG` | Endereço-base do segmento `argument` |
| `RAM[3]` | `THIS` | Endereço-base do segmento `this` |
| `RAM[4]` | `THAT` | Endereço-base do segmento `that` |
| `RAM[5..12]` | — | Segmento `temp` (8 células) |
| `RAM[13..15]` | `R13`–`R15` | Uso interno do VM Translator (registradores de trabalho) |
| `RAM[16..255]` | — | Variáveis `static` (de todos os arquivos `.vm`) |
| `RAM[256..2047]` | — | Pilha |
| `RAM[2048..16383]` | — | Heap (objetos e arrays) |
| `RAM[16384..24575]` | `SCREEN` | Mapa de memória da tela |
| `RAM[24576]` | `KBD` | Teclado |

O princípio geral: os quatro segmentos "dinâmicos" (`local`, `argument`, `this`, `that`) têm seu endereço-base guardado em um registrador (`LCL`, `ARG`, `THIS`, `THAT`) que muda a cada chamada de função (capítulo 13) — acessar `local i` significa sempre `RAM[LCL + i]`. Já `temp` e `static` têm endereço fixo, calculável em tempo de tradução, sem indireção nenhuma.

Os símbolos `SP`, `LCL`, `ARG`, `THIS`, `THAT` e `R13`–`R15` são **pré-definidos** no Assembly Hack (capítulo 11): o código gerado pode escrever `@SP` em vez de `@0`, o que o torna bem mais legível.

## Antes do Assembly: ponteiros e pseudocódigo

Quase tudo que o tradutor gera é manipulação de **ponteiros**: `SP` não guarda o valor do topo da pilha, guarda o *endereço* da próxima posição livre; `LCL` não guarda uma variável local, guarda o *endereço* onde o segmento começa. Por isso vale adotar uma notação curta, emprestada de C, antes de escrever qualquer Assembly:

| Notação | Significado | Em Assembly Hack |
|---|---|---|
| `D = *p` | D recebe o valor **apontado** por `p` | `@p`, `A=M`, `D=M` |
| `*q = 9` | a célula apontada por `q` recebe 9 | `@9`, `D=A`, `@q`, `A=M`, `M=D` |
| `p--` | o ponteiro passa a apontar para a célula anterior | `@p`, `M=M-1` |
| `q++` | o ponteiro passa a apontar para a próxima célula | `@q`, `M=M+1` |

O padrão a memorizar é o par `@p` / `A=M`: a primeira instrução seleciona a célula **onde está o ponteiro**; a segunda copia o conteúdo dela (um endereço) para `A`, de modo que o próximo `M` já é a célula **apontada**.

Com essa notação, a pilha inteira se descreve em duas linhas. Empilhar um valor `x` é `*SP = x; SP++`; desempilhar para `D` é `SP--; D = *SP`. Por exemplo, `push constant 17`:

```
// *SP = 17
@17
D=A
@SP
A=M
M=D
// SP++
@SP
M=M+1
```

A estratégia para o resto do capítulo é sempre a mesma, em dois passos: primeiro descrever cada comando VM em **pseudocódigo com ponteiros**, depois traduzir o pseudocódigo para Assembly. O primeiro passo é onde está o raciocínio; o segundo é quase mecânico.

## Traduzindo `push`/`pop`

A tradução de `push`/`pop` se divide em dois casos, com custos bem diferentes.

**Segmentos dinâmicos** (`local`, `argument`, `this`, `that`) exigem calcular um endereço antes de acessar. Em pseudocódigo:

```
push local i   →   addr = LCL + i;  *SP = *addr;  SP++
pop local i    →   addr = LCL + i;  SP--;  *addr = *SP
```

Em Assembly, `push` carrega o registrador-base, soma o índice e desreferencia:

```
// push local i
@LCL
D=M            // D = valor de LCL (o endereço-base)
@i
A=D+A          // A = LCL + i  (addr)
D=M            // D = RAM[LCL + i]  (*addr)
@SP
A=M
M=D            // *SP = D
@SP
M=M+1          // SP++
```

`pop` para um segmento dinâmico é mais delicado: é preciso calcular o endereço-destino **e** desempilhar um valor, mas só temos os registradores `A`/`D` para trabalhar — nenhum dos dois resultados pode "esperar" sem ser guardado em algum lugar. A solução clássica é um registrador de uso geral (`R13`, uma das células reservadas da tabela acima) como área de rascunho para `addr`:

```
// pop local i
@LCL
D=M
@i
D=D+A          // D = LCL + i (addr)
@R13
M=D            // R13 = addr (guardado)
@SP
AM=M-1         // SP--; A = SP (nova posição do topo)
D=M            // D = *SP (valor desempilhado)
@R13
A=M            // A = addr (recuperado)
M=D            // *addr = valor
```

`argument`, `this` e `that` são idênticos, trocando `@LCL` por `@ARG`, `@THIS` ou `@THAT`.

**Segmentos com endereço fixo** (`constant`, `temp`, `pointer`, `static`) dispensam esse cálculo intermediário. `constant` nem acessa memória — o valor já *é* o índice (`*SP = i; SP++`):

```
// push constant i
@i
D=A            // D = i (não M — não há memória a ler; o valor é o próprio literal)
@SP
A=M
M=D
@SP
M=M+1
```

`temp i` é sempre `RAM[5+i]`, e `static i` é mapeado para um símbolo único por arquivo `.vm` (`NomeDoArquivo.i`) — o Assembler (capítulo 15) se encarrega de atribuir um endereço real a esse símbolo, exatamente como faz com qualquer variável, a partir de `RAM[16]`. Em ambos os casos, o endereço vai direto na A-instruction:

```
// push temp 2
@7             // 5 + 2
D=M
@SP
A=M
M=D
@SP
M=M+1
```

```
// pop static 3   (dentro de Foo.vm)
@SP
AM=M-1
D=M
@Foo.3
M=D
```

O símbolo `Foo.3` explica por que `static` é compartilhado por todas as subrotinas de uma classe e, ao mesmo tempo, separado entre classes: `static 3` em `Foo.vm` e `static 3` em `Bar.vm` viram `@Foo.3` e `@Bar.3` — duas variáveis distintas do Assembly.

**`pointer`** é o segmento que liga a VM aos registradores-base: `pointer 0` é o próprio registrador `THIS` (`RAM[3]`) e `pointer 1` é `THAT` (`RAM[4]`) — ou seja, `pointer i` é `RAM[3+i]`, um endereço fixo como `temp`. A diferença é o efeito: escrever em `pointer` não grava um dado, e sim **muda a base** dos segmentos `this`/`that`. É assim que o `pop pointer 0` gerado pelo compilador nos construtores e métodos (capítulo 10) ancora `this` no objeto:

```
// pop pointer 0
@SP
AM=M-1
D=M
@THIS          // RAM[3 + 0]
M=D            // THIS = valor desempilhado
```

```
// push pointer 1
@THAT          // RAM[3 + 1]
D=M
@SP
A=M
M=D
@SP
M=M+1
```

Depois de um `pop pointer 0`, todo `this i` traduzido pela regra dos segmentos dinâmicos já usa o novo endereço-base — é a mesma ideia do exemplo `push constant 8000`, `pop pointer 0` do capítulo 7, agora em Assembly.

## Traduzindo comandos aritméticos e lógicos

Em pseudocódigo, um comando binário (`add`, `sub`, `and`, `or`) desempilha dois valores, calcula, e empilha o resultado:

```
add   →   SP--;  y = *SP;  SP--;  x = *SP;  *SP = x + y;  SP++
```

A implementação evita metade desse trabalho fazendo a operação **"in-place"**: o resultado sobrescreve diretamente a posição de `x`, que é onde ele deve ficar:

```
// add
@SP
AM=M-1         // SP--; A = posição do segundo operando (y)
D=M            // D = y
A=A-1          // A = posição do primeiro operando (x)
M=D+M          // x = x + y (resultado fica exatamente onde deve)
```

Repare que **nenhum `M=M+1` é necessário**: a pilha encolheu de dois elementos para um só com um único decremento, e o resultado ocupa a posição que sobrou. `sub`, `and` e `or` mudam apenas a última linha (`M=M-D`, `M=D&M`, `M=D|M`) — atenção à ordem em `sub`: o resultado é `x - y`, ou seja, `M-D`.

Comandos unários (`neg`, `not`) são ainda mais diretos, porque não há segundo operando a desempilhar — o topo é modificado no lugar:

```
// neg
@SP
A=M-1
M=-M           // (para not: M=!M)
```

**Comparações** (`eq`, `lt`, `gt`) são o caso mais elaborado, porque o Hack não tem uma instrução de "comparar e produzir booleano" — é preciso simular com um salto condicional. Lembrando que, na VM, verdadeiro é `-1` (todos os bits em 1) e falso é `0`:

```
// eq
@SP
AM=M-1
D=M
A=A-1
D=M-D           // D = x - y (zero se iguais)
@TRUE_EQ_0
D;JEQ            // se D == 0, salta para o ramo "verdadeiro"
@SP
A=M-1
M=0             // falso: 0
@END_EQ_0
0;JMP
(TRUE_EQ_0)
@SP
A=M-1
M=-1             // verdadeiro: -1 (todos os bits em 1)
(END_EQ_0)
```

`lt` e `gt` seguem exatamente o mesmo esqueleto, trocando apenas `JEQ` por `JLT`/`JGT`. Os rótulos (`TRUE_EQ_0`, `END_EQ_0`) precisam de um contador único por comparação traduzida, para não colidir entre múltiplas comparações no mesmo programa — `TRUE_EQ_0`, `TRUE_LT_1`, `TRUE_EQ_2`, ...

!!! note "Um limite do esqueleto de comparação"
    O teste `x - y` pode **transbordar** quando `x` e `y` têm sinais opostos e magnitudes grandes: com `x = 20000` e `y = -20000`, a subtração dá 40000, que em 16 bits vira um número negativo — e `lt` responde "verdadeiro" para `20000 < -20000`. Os testes oficiais não exercitam esse caso, e o esqueleto acima é o que a maioria das implementações usa. Uma versão robusta compara primeiro os sinais de `x` e `y` e só subtrai quando são iguais — um bom desafio extra depois que todos os testes passarem.

## Uma tabela de custos

Vale a pena registrar, em número de instruções Assembly geradas, o custo de cada comando VM nas traduções mostradas — é uma boa intuição para entender por que um código VM aparentemente pequeno vira um Assembly bem maior:

| Comando VM | Instruções Assembly |
|---|---|
| `push constant` | 7 |
| `push local`/`argument`/`this`/`that` | 10 |
| `pop local`/`argument`/`this`/`that` | 12 |
| `push temp`/`static`/`pointer` | 7 |
| `pop temp`/`static`/`pointer` | 5 |
| `add`/`sub`/`and`/`or` | 5 |
| `neg`/`not` | 3 |
| `eq`/`lt`/`gt` | 15 |
| `label` | 0 (é só um símbolo) |
| `goto` | 2 |
| `if-goto` | 5 |

(As três últimas linhas antecipam o capítulo 13.) O exemplo `d = (2 - x) + (y + 9)` acima, com 8 comandos VM, vira 7 + 10 + 5 + 10 + 7 + 5 + 5 + 12 = 61 instruções Assembly.

## Arquitetura do tradutor

Como o Assembler do capítulo 15 e o analisador da Parte 1, o VM Translator se organiza em três módulos:

- **`Parser`** — lê um arquivo `.vm`, descarta espaços e comentários, e entrega um comando por vez, já decomposto: o **tipo** do comando (aritmético, `push`, `pop`; no capítulo 13, também `label`, `goto`, `if`, `function`, `call`, `return`), seu primeiro argumento (`arg1`: o nome do segmento, ou o próprio comando no caso aritmético) e o segundo (`arg2`: o índice).
- **`CodeWriter`** — gera o Assembly de cada comando: `writeArithmetic(comando)` e `writePushPop(comando, segmento, índice)`. Guarda o estado de que a tradução precisa: o nome do arquivo corrente (para os símbolos `static`) e o contador de rótulos das comparações.
- **`Main`** (ou `VMTranslator`) — recebe `Xxx.vm`, cria um `Parser` e um `CodeWriter` com saída em `Xxx.asm`, e percorre o arquivo comando a comando.

Uma convenção que facilita muito a depuração: antes do Assembly de cada comando, o `CodeWriter` escreve o **próprio comando VM como comentário**:

```
// push constant 17
@17
D=A
@SP
A=M
M=D
@SP
M=M+1
// push constant 19
@19
...
```

Comentários não custam nada (o Assembler os descarta), e quando o CPU Emulator mostrar um resultado errado, você vai saber exatamente qual comando VM gerou cada trecho.

## Testando: o plano de desenvolvimento

Os testes oficiais (pasta `projects/07`) foram pensados para serem feitos **em ordem**, cada um exercitando um pedaço novo do tradutor:

| Teste | O que exercita | O que o script `.tst` inicializa |
|---|---|---|
| `StackArithmetic/SimpleAdd` | `push constant` e `add` | `SP` |
| `StackArithmetic/StackTest` | todos os comandos aritméticos e lógicos, inclusive comparações | `SP` |
| `MemoryAccess/BasicTest` | `push`/`pop` em `constant`, `local`, `argument`, `this`, `that`, `temp` | `SP`, `LCL`, `ARG`, `THIS`, `THAT` |
| `MemoryAccess/PointerTest` | `pointer`, `this` e `that` | `SP` |
| `MemoryAccess/StaticTest` | `static` | `SP` |

Para cada teste, o ciclo é sempre o mesmo:

1. **(Opcional)** Carregue `XxxVME.tst` no **VM Emulator** — ele carrega o próprio `Xxx.vm` e o executa passo a passo, mostrando a pilha e os segmentos. É a melhor forma de entender o que o programa **deveria** fazer antes de olhar o seu Assembly.
2. Rode o seu tradutor sobre `Xxx.vm`, gerando `Xxx.asm`.
3. **Inspecione** o `Xxx.asm` (os comentários com os comandos VM ajudam aqui).
4. Carregue `Xxx.tst` no **CPU Emulator**: o script carrega o seu `Xxx.asm`, executa um número fixo de ciclos e compara o estado final da RAM com `Xxx.cmp`.

A coluna da direita da tabela merece atenção. Neste projeto, ainda não há `function`, `call` nem `return` — e são justamente esses comandos que, no capítulo 13, vão inicializar `SP`, `LCL`, `ARG`, `THIS` e `THAT`. Sem eles, quem faz esse papel é o **script de teste**: `BasicTest.tst`, por exemplo, começa com `set RAM[0] 256, set RAM[1] 300, set RAM[2] 400, set RAM[3] 3000, set RAM[4] 3010` antes de executar `repeat 600 { ticktock; }`. Por isso o seu `.asm` **não** deve inicializar esses registradores neste projeto — o teste já os preparou, e sobrescrevê-los faria o resultado não bater com o `.cmp`.

## Teste rápido

??? question "1. Traduza `push argument 3` para Assembly."
    `@ARG`, `D=M`, `@3`, `A=D+A`, `D=M`, `@SP`, `A=M`, `M=D`, `@SP`, `M=M+1` — o mesmo esqueleto de `push local i`, com `ARG` como base.

??? question "2. Em Jack, `let static 2 = argument 1` não é uma linha válida, mas a ideia é: copiar o segundo argumento para a terceira variável estática. Que comandos VM fazem isso, e qual o Assembly, dentro de `Foo.vm`?"
    `push argument 1`, `pop static 2`. O `push` é o esqueleto dos segmentos dinâmicos (10 instruções); o `pop` termina em `@Foo.2`, `M=D` (5 instruções).

??? question "3. Por que `pop local i` precisa de `R13`, mas `pop temp i` não?"
    Porque o endereço de `local i` (`LCL + i`) só é conhecido em tempo de execução e ocupa o registrador `D` enquanto ainda falta desempilhar o valor, que também precisa de `D`. O endereço de `temp i` (`5 + i`) é conhecido em tempo de tradução e vai direto numa A-instruction, depois que o valor já está em `D`.

??? question "4. Depois de `push constant 7`, `push constant 3`, `sub`, o que fica no topo da pilha? E se a última linha do `sub` fosse `M=D-M`?"
    `7 - 3 = 4`. Com `M=D-M`, o resultado seria `3 - 7 = -4`: na tradução in-place, `D` guarda o **segundo** operando (`y`) e `M` o primeiro (`x`), e por isso `sub` é `M=M-D`.

??? question "5. Por que o rótulo `TRUE_EQ_0` precisa do contador no nome?"
    Porque um mesmo arquivo pode ter vários `eq`. Se todos usassem `(TRUE_EQ)`, o Assembler encontraria o rótulo declarado várias vezes, e todos os saltos iriam para o mesmo lugar.

## Exercício guiado

Sua atividade prática nesta parte do curso é implementar um `VMTranslator` que recebe um arquivo `.vm` e produz o `.asm` correspondente, cobrindo:

1. Todos os comandos aritméticos/lógicos (`add`, `sub`, `neg`, `eq`, `gt`, `lt`, `and`, `or`, `not`).
2. `push` para todos os oito segmentos e `pop` para sete deles (não existe `pop constant`: `constant` é um segmento virtual, só de leitura).
3. Rótulos únicos para cada comparação traduzida (um contador incremental é suficiente).
4. Os comandos VM como comentários no `.asm` gerado.

Siga a ordem da tabela de testes acima: `SimpleAdd` só precisa de `push constant` e `add`, e já valida o esqueleto do `Parser` e do `CodeWriter`; cada teste seguinte acrescenta uma família de comandos.

## O que vem a seguir

Esta parte cobre apenas metade do VM Translator — falta o controle de fluxo em escopo de função (rótulos qualificados) e, principalmente, o protocolo completo de `call`/`return` com *stack frames*, que é o que torna possível compilar chamadas de função recursivas. É o assunto do capítulo 13, que completa o VM Translator (Project 8) antes de seguirmos para a consolidação da geração de código alvo (capítulo 14) e o Assembler (capítulo 15).
