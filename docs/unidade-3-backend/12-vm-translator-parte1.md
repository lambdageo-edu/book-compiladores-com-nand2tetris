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

## Modelo de memória: Hack

A máquina Hack tem uma ROM de instruções e uma RAM de 32K palavras de 16 bits. Os primeiros endereços da RAM têm papéis pré-definidos, que sustentam todo o mapeamento de segmentos VM → memória real:

| Endereço RAM | Registrador | Papel |
|---|---|---|
| `RAM[0]` | `SP` | Ponteiro de topo da pilha |
| `RAM[1]` | `LCL` | Endereço-base do segmento `local` |
| `RAM[2]` | `ARG` | Endereço-base do segmento `argument` |
| `RAM[3]` | `THIS` | Endereço-base do segmento `this` |
| `RAM[4]` | `THAT` | Endereço-base do segmento `that` |
| `RAM[5..12]` | — | Segmento `temp` (8 células) |
| `RAM[13..15]` | — | Uso interno do VM Translator (registradores de trabalho) |
| `RAM[16..255]` | — | Variáveis `static` |
| `RAM[256..2047]` | — | Pilha |
| `RAM[2048..16383]` | — | Heap (objetos e arrays) |
| `RAM[16384..24575]` | `SCREEN` | Mapa de memória da tela |
| `RAM[24576]` | `KBD` | Teclado |

O princípio geral: os quatro segmentos "dinâmicos" (`local`, `argument`, `this`, `that`) têm seu endereço-base guardado em um registrador (`LCL`, `ARG`, `THIS`, `THAT`) que muda a cada chamada de função (capítulo 13) — acessar `local i` significa sempre `RAM[LCL + i]`. Já `temp` e `static` têm endereço fixo, calculável em tempo de tradução, sem indireção nenhuma.

## Traduzindo `push`/`pop`

A tradução de `push`/`pop` se divide em dois casos, com custos bem diferentes.

**Segmentos dinâmicos** (`local`, `argument`, `this`, `that`) exigem calcular um endereço antes de acessar — carregar o registrador-base, somar o índice, então desreferenciar:

```
// push local i
@LCL
D=M            // D = valor de LCL (o endereço-base)
@i
A=D+A          // A = LCL + i
D=M            // D = RAM[LCL + i]
@SP
A=M
M=D            // RAM[SP] = D
@SP
M=M+1          // SP++
```

`pop` para um segmento dinâmico é mais delicado: é preciso calcular o endereço-destino **e** desempilhar um valor, mas só temos os registradores `A`/`D`/`M` para trabalhar — nenhum dos dois cálculos pode "esperar" sem ser guardado em algum lugar. A solução clássica é um registrador de uso geral (`R13`, uma das células reservadas da tabela acima) como área de rascunho:

```
// pop local i
@LCL
D=M
@i
D=D+A          // D = LCL + i (o endereço-destino)
@R13
M=D            // R13 = endereço-destino (guardado)
@SP
AM=M-1         // SP--; A = SP (nova posição do topo)
D=M            // D = valor desempilhado
@R13
A=M            // A = endereço-destino (recuperado)
M=D            // RAM[endereço-destino] = valor
```

**Segmentos com endereço fixo** (`constant`, `temp`, `pointer`, `static`) dispensam esse cálculo intermediário. `constant` nem acessa memória — o valor já *é* o índice:

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

`temp i` é sempre `RAM[5+i]`, e `static i` é mapeado para um símbolo único por arquivo `.vm` (tipicamente `NomeDoArquivo.i`) — o Assembler (capítulo 15) se encarrega de atribuir um endereço real a esse símbolo, exatamente como faz com qualquer variável. Em ambos os casos, o endereço vai direto na A-instruction:

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

Um comando binário (`add`, `sub`, `and`, `or`) desempilha dois valores, calcula, e empilha o resultado — mas a implementação evita uma escrita extra fazendo a operação **"in-place"**, sobrescrevendo diretamente a posição onde o resultado deve ficar:

```
// add
@SP
AM=M-1         // SP--; A = posição do segundo operando
D=M            // D = segundo operando
A=A-1          // A = posição do primeiro operando (SP já foi decrementado uma vez)
M=D+M          // RAM[primeiro] = primeiro + segundo (resultado fica exatamente onde deve)
```

Repare que **nenhum novo `M=M+1` é necessário**: a pilha já encolheu de dois elementos para um só (um `AM=M-1` implícito), e o resultado ocupa a posição que sobrou — sem incrementar `SP` de volta.

Comandos unários (`neg`, `not`) são ainda mais diretos, porque não há segundo operando a desempilhar:

```
// neg
@SP
A=M-1
M=-M
```

**Comparações** (`eq`, `lt`, `gt`) são o caso mais elaborado, porque a VM não tem um comando de "comparar e produzir booleano" nativo em Assembly — é preciso simular com um salto condicional:

```
// eq
@SP
AM=M-1
D=M
A=A-1
D=M-D           // D = primeiro - segundo (zero se iguais)
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

`lt` e `gt` seguem exatamente o mesmo esqueleto, trocando apenas `JEQ` por `JLT`/`JGT`. E, como no capítulo 9, os rótulos (`TRUE_EQ_0`, `END_EQ_0`) precisam de um contador único por comando de comparação traduzido, para não colidir entre múltiplas comparações no mesmo arquivo.

## Uma tabela de custos

Vale a pena registrar, em número de instruções Assembly geradas, o custo relativo de cada comando VM — é uma boa intuição para entender por que um código VM aparentemente pequeno vira um Assembly bem maior:

| Comando VM | Instruções Assembly (aprox.) |
|---|---|
| `push constant` | 7 |
| `push`/`pop` segmento dinâmico | 10–12 |
| `push temp`/`static`/`pointer` | 7 |
| `pop temp`/`static`/`pointer` | 5 |
| `add`/`sub`/`and`/`or` | 5 |
| `neg`/`not` | 3 |
| `eq`/`lt`/`gt` | ~16 |
| `label` | 0 (é só um símbolo) |
| `goto` | 2 |
| `if-goto` | 5 |

## Exercício guiado

Sua atividade prática nesta parte do curso é implementar um `VMTranslator` que recebe um arquivo `.vm` (ou uma pasta com vários) e produz o `.asm` correspondente, cobrindo:

1. Todos os comandos aritméticos/lógicos (`add`, `sub`, `neg`, `eq`, `gt`, `lt`, `and`, `or`, `not`).
2. `push`/`pop` para todos os oito segmentos.
3. Rótulos únicos para cada comparação traduzida (um contador incremental é suficiente).

Os testes oficiais do Nand2Tetris para esta parte (pasta `projects/07`) incluem `SimpleAdd`, `StackTest` (aritmética e lógica combinadas), `BasicTest`, `PointerTest` e `StaticTest` (os quatro segmentos e os dois casos especiais `pointer`/`static`) — rode cada `.asm` gerado no CPU Emulator e compare o estado final da RAM com o script `.tst`/`.cmp` de referência.

## O que vem a seguir

Esta parte cobre apenas metade do VM Translator — falta o controle de fluxo em escopo de função (rótulos qualificados) e, principalmente, o protocolo completo de `call`/`return` com *stack frames*, que é o que torna possível compilar chamadas de função recursivas. É o assunto do capítulo 13, que completa o VM Translator (Project 8) antes de seguirmos para a consolidação da geração de código alvo (capítulo 14) e o Assembler (capítulo 15).
