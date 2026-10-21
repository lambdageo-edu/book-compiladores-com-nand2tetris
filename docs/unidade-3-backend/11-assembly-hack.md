# 11. Assembly Hack (especificação)

Este capítulo especifica, por completo, a linguagem Assembly Hack — a linguagem simbólica para a qual o VM Translator (capítulos 11-12) traduz, e que o Assembler (capítulo 15) vai finalmente converter em código de máquina binário. É referência de consulta, não um capítulo incremental — mas vale lê-lo por inteiro pelo menos uma vez antes de atacar o Assembler, porque cada peça (registradores, instruções, símbolos) é usada o tempo todo no capítulo seguinte.

## Por que uma linguagem simbólica

Projetistas de processadores definem uma linguagem assembly — sequências de instruções mnemônicas que representam diretamente as operações do processador, mais símbolos que facilitam a escrita de programas (rótulos, comentários, constantes nomeadas). Tudo isso é, no fim, mapeado para código de máquina — números binários — por um montador. A vantagem é pura ergonomia: `@LOOP` é imensamente mais legível que o endereço numérico que ele representa, mas os dois significam exatamente a mesma coisa para a CPU.

## A arquitetura Hack

A máquina Hack é uma arquitetura de 16 bits, simples o suficiente para caber inteira em um curso, mas Turing-completa (capaz de executar qualquer algoritmo computável):

- **RAM**: uma sequência de registros de 16 bits, `RAM[0]`, `RAM[1]`, ... — a memória de dados.
- **ROM**: uma sequência de registros de 16 bits, `ROM[0]`, `ROM[1]`, ... — a memória de instruções (o programa).
- **CPU**: executa instruções de 16 bits, uma por vez.
- Três registradores de 16 bits acessíveis pelo programador: `D` (dados), `A` (endereço/dado), e `M` — que não é bem um registrador, mas uma notação para "a célula de RAM atualmente endereçada por `A`" (`M = RAM[A]`).

Um programa Hack é, literalmente, uma sequência de instruções de dois tipos: **A-instructions** e **C-instructions**.

## A-instruction

Formato: `@valor`, onde `valor` é um número (0 a 32767) ou um símbolo que se refere a um número. Seu único efeito é carregar esse valor no registrador `A`:

```
@10        // A = 10
@i         // A = endereço associado ao símbolo i
(LOOP)     // declara um rótulo — não é uma instrução, é um marcador de posição
@LOOP      // A = endereço associado ao rótulo LOOP
```

Em binário, uma A-instruction sempre começa com o bit `0`, seguido de 15 bits representando o valor: `0 vvvvvvvvvvvvvvv`.

## C-instruction

Toda operação de cálculo, atribuição e desvio condicional/incondicional é uma C-instruction, no formato:

```
dest=comp;jump
```

Cada uma das três partes é opcional (mas a ordem é fixa): se `dest` estiver vazio, o `=` é omitido; se `jump` estiver vazio, o `;` é omitido. Em binário, uma C-instruction sempre começa com `111`, seguida por 13 bits que codificam `comp` (7 bits), `dest` (3 bits) e `jump` (3 bits).

### `comp` — o que calcular

A ULA (unidade lógica e aritmética) de Hack computa uma dentre um conjunto fixo de operações sobre `D`, `A` (ou `M`, alternativamente) e as constantes `0`, `1`, `-1`:

| Mnemônico | Operação | Mnemônico | Operação |
|---|---|---|---|
| `0`, `1`, `-1` | constantes | `D+1`, `A+1`/`M+1` | incremento |
| `D`, `A`/`M` | valor do registrador | `D-1`, `A-1`/`M-1` | decremento |
| `!D`, `!A`/`!M` | negação bit a bit (NOT) | `D+A`/`D+M` | soma |
| `-D`, `-A`/`-M` | negação aritmética | `D-A`/`D-M`, `A-D`/`M-D` | subtração |
| | | `D&A`/`D&M` | E bit a bit |
| | | `D\|A`/`D\|M` | OU bit a bit |

Cada expressão que usa `A` tem uma variante gêmea que usa `M` em seu lugar (`A+1`/`M+1`, `D-A`/`D-M`, etc.) — a diferença, no binário, é um único bit (chamado `a`) que seleciona entre os dois.

### `dest` — onde armazenar

| Mnemônico | Significado |
|---|---|
| (vazio) | resultado descartado |
| `M` | `RAM[A]` |
| `D` | registrador `D` |
| `MD` | ambos |
| `A` | registrador `A` |
| `AM`, `AD`, `AMD` | combinações |

### `jump` — o que fazer a seguir

Por padrão, a CPU busca e executa a próxima instrução em sequência. Um `jump` desvia esse fluxo, testando o valor computado por `comp` contra zero:

| Mnemônico | Salta se |
|---|---|
| (vazio) | nunca (nunca salta) |
| `JGT` | `comp > 0` |
| `JEQ` | `comp == 0` |
| `JGE` | `comp >= 0` |
| `JLT` | `comp < 0` |
| `JNE` | `comp != 0` |
| `JLE` | `comp <= 0` |
| `JMP` | sempre |

O destino do salto é sempre o endereço atualmente carregado em `A` — por isso um `goto` em Assembly Hack é sempre um par de instruções: uma A-instruction que carrega o destino, seguida de uma C-instruction que salta.

```
// if Memory[3] == 5 then goto 100 else goto 200
@3
D=M     // D = RAM[3]
@5
D=D-A   // D = D - 5
@100
D;JEQ   // se D == 0, salta para ROM[100]
@200
0;JMP   // senão, salta para ROM[200]
```

## Símbolos

Escrever programas inteiros em endereços numéricos seria impraticável — Assembly Hack define três categorias de símbolo, todos usados através de uma A-instruction (`@símbolo`).

### Símbolos pré-definidos

- **Registradores virtuais** `R0`–`R15`: referem-se diretamente a `RAM[0]`–`RAM[15]`.
- **Ponteiros de segmento** `SP`, `LCL`, `ARG`, `THIS`, `THAT`: referem-se a `RAM[0]`–`RAM[4]` — os mesmos endereços de `R0`–`R4`, apenas com nomes mnemônicos para o papel que cumprem na VM (capítulos 7, 11 e 12).
- **Mapas de E/S** `SCREEN` (RAM 16384) e `KBD` (RAM 24576): endereço-base do mapa de memória da tela e o registrador de teclado, respectivamente.

### Rótulos

Declarados com `(NomeDoRotulo)`, um rótulo associa um símbolo ao endereço **ROM** da próxima instrução real do programa (rótulos não geram código — são um pseudo-comando, puramente para o montador). Um rótulo pode ser usado — via `@NomeDoRotulo` — em qualquer ponto do programa, inclusive **antes** de sua declaração; é exatamente esse caso ("referência futura") que motiva a arquitetura de duas passagens do Assembler, no capítulo 15.

### Variáveis

Qualquer símbolo usado em `@símbolo` que não seja pré-definido nem declarado como rótulo é, por eliminação, uma **variável** — e o Assembler aloca automaticamente um endereço de RAM único para ela, começando em `RAM[16]` (os primeiros 16 endereços já estão reservados para os símbolos pré-definidos).

## Convenções de escrita

Por legibilidade (e por convenção adotada por praticamente todo material do Nand2Tetris): variáveis em minúsculas, rótulos em maiúsculas, indentação nas instruções que não são rótulos. Comentários começam com `//` e são ignorados pelo Assembler.

## Programas de exemplo

**Soma de duas posições de memória:**

```
// Computa RAM[2] = RAM[0] + RAM[1]
@0
D=M
@1
D=D+M
@2
M=D
@END
(END)
0;JMP        // loop infinito — todo programa Hack termina assim
```

**Somatória de 1 a N** (`RAM[1] = 1 + 2 + ... + RAM[0]`) — primeiro em pseudocódigo, depois mapeado mecanicamente, bloco por bloco:

```
n = R0
i = 1
sum = 0
LOOP:
    if i > n goto STOP
    sum = sum + i
    i = i + 1
    goto LOOP
STOP:
    R1 = sum
```

vira:

```
@R0
D=M
@n
M=D          // n = R0
@i
M=1          // i = 1
@sum
M=0          // sum = 0

(LOOP)
@i
D=M
@n
D=D-M
@STOP
D;JGT        // if i > n goto STOP
@sum
D=M
@i
D=D+M
@sum
M=D          // sum = sum + i
@i
M=M+1        // i = i + 1
@LOOP
0;JMP

(STOP)
@sum
D=M
@R1
M=D          // RAM[1] = sum

(END)
@END
0;JMP
```

**Arrays via aritmética de ponteiros** — a mesma técnica usada pelo compilador Jack (capítulo 10) para `arr[i]`, aqui explícita:

```
// for (i = 0; i < n; i++) arr[i] = -1;
// suponha arr = 100, n = 10
@100
D=A
@arr
M=D          // arr = 100
@10
D=A
@n
M=D          // n = 10
@i
M=0          // i = 0

(LOOP)
@i
D=M
@n
D=D-M
@END
D;JEQ        // if i == n goto END
@arr
D=M
@i
A=D+M        // A = arr + i (endereço do elemento)
M=-1         // RAM[arr+i] = -1
@i
M=M+1        // i++
@LOOP
0;JMP
(END)
@END
0;JMP
```

Repare que `A=D+M` seguido de `M=-1` é exatamente o padrão de "endereçamento indireto" que já vimos no capítulo 10 — a diferença é que ali o compilador Jack gerava esse padrão via `pointer`/`that`; aqui, sem a camada da VM, ele aparece diretamente.

## Entrada e saída

O teclado é lido através do símbolo `KBD` (`RAM[24576]`): contém `0` se nenhuma tecla está pressionada, ou o código da tecla caso contrário. A tela é um mapa de memória de 8K palavras a partir de `SCREEN` (`RAM[16384]`) — 256 linhas de 512 pixels, cada linha ocupando 32 palavras de 16 bits consecutivas, um bit por pixel (`1` = preto, `0` = branco):

```
@SCREEN
M=1     // 0000000000000001 — acende o pixel mais à esquerda da primeira linha
```

## O que vem a seguir

Com a especificação completa da linguagem em mãos, o capítulo 15 constrói o Assembler: o programa que lê um arquivo `.asm` como este e produz o binário `.hack` correspondente — resolvendo, no processo, exatamente os símbolos que acabamos de especificar.

As tabelas de codificação (`comp`/`dest`/`jump`) e os símbolos pré-definidos estão consolidados, para consulta rápida, no [Apêndice C](../apendices/c-especificacao-assembly-hack.md).
