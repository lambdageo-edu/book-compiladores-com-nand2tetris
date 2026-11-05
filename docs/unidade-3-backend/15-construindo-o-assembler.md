# 15. Construindo o Assembler

Chegamos à última peça da cadeia de tradução: o **Assembler** (Project 6), que converte um arquivo `.asm` escrito na linguagem do capítulo 11 em um arquivo `.hack` — a sequência de instruções binárias de 16 bits que a CPU Hack executa diretamente. Como o VM Translator (capítulos 12 e 13), este capítulo traz teoria, especificação e arquitetura; a implementação completa é a sua atividade prática, comparando a saída com os testes oficiais do Nand2Tetris.

O caminho segue a ordem em que o próprio Nand2Tetris recomenda construir o Assembler: primeiro entender **para quem** ele escreve (a CPU Hack), depois montar uma versão que só traduz programas **sem símbolos**, e só então acrescentar os símbolos — que é onde mora o único problema realmente interessante, o da referência futura.

## Para quem o Assembler escreve: o computador Hack

Todo computador repete, sem parar, o mesmo ciclo: **buscar** (*fetch*) uma instrução na memória e **executá-la** (*execute*). O computador Hack faz isso com duas memórias separadas — uma variação da arquitetura de von Neumann às vezes chamada de **arquitetura de Harvard**:

- a **memória de instruções** (ROM, um chip de 32K palavras chamado `ROM32K`), de onde a CPU só lê o programa;
- a **memória de dados** (RAM), que a CPU lê e escreve: `RAM[0]` a `RAM[16383]` são dados comuns, `RAM[16384]` a `RAM[24575]` são o mapa da tela, e `RAM[24576]` é o teclado (capítulo 11).

Com as duas memórias separadas, a CPU pode buscar a próxima instrução e acessar um dado no mesmo ciclo, sem conflito. O preço é que o tamanho de cada uma fica fixo.

Quem decide qual instrução vem a seguir é o **contador de programa** (PC), um registrador que guarda o endereço, na ROM, da instrução atual:

```
se reset:                         PC = 0
senão, se a instrução manda saltar: PC = A
senão:                            PC = PC + 1
```

Rodar um programa é, portanto: carregar o `.hack` na ROM (no simulador, abrindo o arquivo; numa máquina real, gravando o chip) e apertar `reset`. O PC vai a 0, e a CPU começa a executar `ROM[0]`, `ROM[1]`, ... É por isso que um rótulo vale o **endereço na ROM** da próxima instrução: é esse o número que o PC precisa receber para "saltar para lá".

### O que a CPU faz com 16 bits

O arquivo `.hack` é texto: uma linha por instrução, cada linha com 16 caracteres `0` ou `1`. A CPU olha primeiro para o bit mais à esquerda, o **op-code**:

```
A-instruction:   0  v v v v v v v v v v v v v v v
                 │  └────────── valor (15 bits) ─────────┘
                 └─ op-code 0: "carregue este valor em A"

C-instruction:   1  1 1  a  c1 c2 c3 c4 c5 c6  d1 d2 d3  j1 j2 j3
                 │  └┬┘  └────────┬─────────┘  └───┬──┘  └───┬──┘
                 │   │         comp               dest      jump
                 │   └─ não usados (sempre 1)
                 └─ op-code 1: "calcule"
```

Numa **A-instruction**, os 15 bits restantes são um número, que vai direto para o registrador `A`. Numa **C-instruction**, cada grupo de bits é ligado a uma parte diferente do hardware:

- **`a c1..c6`** controlam a ULA: `a` escolhe se a segunda entrada é o registrador `A` (`a = 0`) ou o valor da memória `M = RAM[A]` (`a = 1`), e os bits `c` escolhem a operação;
- **`d1 d2 d3`** dizem quais destinos guardam o resultado da ULA: `d1` → registrador `A`, `d2` → registrador `D`, `d3` → `RAM[A]`. O resultado é oferecido aos três, mas só os destinos com bit `1` o armazenam;
- **`j1 j2 j3`** dizem em que condição saltar: `j1` "se o resultado for negativo", `j2` "se for zero", `j3` "se for positivo". A ULA informa se o resultado foi negativo ou zero; se a condição pedida for satisfeita, o PC recebe `A`.

Essa ligação direta entre bits e fios explica a forma das tabelas de tradução que veremos adiante — `JGE` é `011` porque significa "salte se zero **ou** positivo" — e explica também por que o Assembler não pode errar um único bit: um bit trocado não gera uma instrução inválida, gera **outra** instrução, perfeitamente executável, que faz outra coisa.

## O desafio do tradutor

Vamos acompanhar um único programa ao longo do capítulo. Ele soma `1 + 2 + ... + RAM[0]` e guarda o resultado em `RAM[1]`: a variável `i` começa em `RAM[0]` e é decrementada até zero, acumulando em `sum`:

```
// Calcula RAM[1] = 1 + 2 + ... + RAM[0]
    @R0
    D=M
    @i
    M=D        // i = RAM[0]
    @sum
    M=0        // sum = 0
(LOOP)
    @i
    D=M
    @END
    D;JEQ      // se i == 0, vá para END
    @sum
    D=M
    @i
    D=D+M
    @sum
    M=D        // sum = sum + i
    @i
    M=M-1      // i = i - 1
    @LOOP
    0;JMP      // volte para LOOP
(END)
    @sum
    D=M
    @R1
    M=D        // RAM[1] = sum
(HALT)
    @HALT
    0;JMP      // laço infinito: todo programa Hack termina assim
```

Do ponto de vista do tradutor, um programa Assembly tem três tipos de elemento:

1. **espaços em branco**: linhas vazias, indentação, comentários de linha inteira e comentários ao fim da linha;
2. **instruções**: A-instructions (`@...`) e C-instructions (`dest=comp;jump`);
3. **símbolos**: referências como `@i`, `@R0` e `@LOOP`, e declarações de rótulo como `(LOOP)`.

Os dois primeiros são tratados linha a linha, sem nenhuma dificuldade. O terceiro exige olhar o programa inteiro. Por isso vamos construir o Assembler em **duas etapas**: primeiro, um Assembler que só aceita programas **sem símbolos**; depois, o tratamento de símbolos, por cima dele. O Nand2Tetris fornece testes para as duas etapas (veja "Validação", no fim do capítulo).

## Etapa 1: um Assembler para programas sem símbolos

Se o programador tivesse escrito números no lugar de cada símbolo — `@0` em vez de `@R0`, `@16` em vez de `@i`, `@20` em vez de `@END`, e assim por diante —, o nosso programa ficaria assim (já sem comentários nem rótulos):

```
@0
D=M
@16
M=D
@17
M=0
@16
D=M
@20
D;JEQ
...
```

Para traduzir um programa desses, bastam três regras.

### Espaços em branco: ignore

Linhas vazias e comentários não geram código, então o primeiro passo é descartá-los: para cada linha, corte tudo a partir de `//`, remova os espaços e, se não sobrar nada, pule a linha. O que sobra é uma instrução por linha.

### A-instructions: decimal para binário

Uma A-instruction `@valor` vira `0` seguido do valor em binário, com 15 bits:

```
@21    →    0 000000000010101    →    0000000000010101
```

Em Java, uma forma direta é completar com zeros à esquerda a representação binária do número:

```java
String toBinary16(int value) {
    String bits = Integer.toBinaryString(value);           // "10101"
    return "0".repeat(16 - bits.length()) + bits;          // "0000000000010101"
}
```

(Como o valor tem no máximo 15 bits — até 32767 —, o bit mais à esquerda sai sempre `0`, que é exatamente o op-code de uma A-instruction.)

### C-instructions: três campos, três tabelas

Uma C-instruction é traduzida em duas fases. Primeiro, ela é **quebrada** nos seus campos, pelas duas marcas que a sintaxe oferece:

- se há um `=`, o que vem antes dele é o `dest`;
- se há um `;`, o que vem depois dele é o `jump`;
- o que sobra no meio é o `comp`.

Depois, cada campo é traduzido **independentemente** pela sua tabela, e os pedaços são concatenados depois do prefixo `111`. As tabelas (também reunidas no apêndice C) são:

| `comp` com `a = 0` | `comp` com `a = 1` | `c1 c2 c3 c4 c5 c6` |
|---|---|---|
| `0` |  | `1 0 1 0 1 0` |
| `1` |  | `1 1 1 1 1 1` |
| `-1` |  | `1 1 1 0 1 0` |
| `D` |  | `0 0 1 1 0 0` |
| `A` | `M` | `1 1 0 0 0 0` |
| `!D` |  | `0 0 1 1 0 1` |
| `!A` | `!M` | `1 1 0 0 0 1` |
| `-D` |  | `0 0 1 1 1 1` |
| `-A` | `-M` | `1 1 0 0 1 1` |
| `D+1` |  | `0 1 1 1 1 1` |
| `A+1` | `M+1` | `1 1 0 1 1 1` |
| `D-1` |  | `0 0 1 1 1 0` |
| `A-1` | `M-1` | `1 1 0 0 1 0` |
| `D+A` | `D+M` | `0 0 0 0 1 0` |
| `D-A` | `D-M` | `0 1 0 0 1 1` |
| `A-D` | `M-D` | `0 0 0 1 1 1` |
| `D&A` | `D&M` | `0 0 0 0 0 0` |
| `D|A` | `D|M` | `0 1 0 1 0 1` |

| `dest` | `d1 d2 d3` | Guarda o resultado em | | `jump` | `j1 j2 j3` | Salta se |
|---|---|---|---|---|---|---|
| (vazio) | `0 0 0` | nenhum lugar | | (vazio) | `0 0 0` | nunca |
| `M` | `0 0 1` | `RAM[A]` | | `JGT` | `0 0 1` | resultado > 0 |
| `D` | `0 1 0` | `D` | | `JEQ` | `0 1 0` | resultado = 0 |
| `MD` | `0 1 1` | `RAM[A]` e `D` | | `JGE` | `0 1 1` | resultado ≥ 0 |
| `A` | `1 0 0` | `A` | | `JLT` | `1 0 0` | resultado < 0 |
| `AM` | `1 0 1` | `A` e `RAM[A]` | | `JNE` | `1 0 1` | resultado ≠ 0 |
| `AD` | `1 1 0` | `A` e `D` | | `JLE` | `1 1 0` | resultado ≤ 0 |
| `AMD` | `1 1 1` | `A`, `RAM[A]` e `D` | | `JMP` | `1 1 1` | sempre |

Repare que as duas colunas de `comp` compartilham os mesmos bits `c`: a única diferença entre `D+A` e `D+M` é o bit `a`. E repare na lógica das outras duas tabelas, que vem direto do hardware: em `dest`, cada bit é um destino (`MD` = `M` + `D` = `001` + `010` = `011`); em `jump`, cada bit é uma condição (`JGE` = "zero" + "positivo" = `010` + `001` = `011`; `JMP` aceita as três).

Um exemplo completo, campo a campo — `MD=D+1`:

| Passo | Resultado |
|---|---|
| quebrar | `dest = MD`, `comp = D+1`, `jump` vazio |
| `comp = D+1` | `a = 0`, `c = 011111` → `0011111` |
| `dest = MD` | `011` |
| `jump` vazio | `000` |
| concatenar | `111` + `0011111` + `011` + `000` = **`1110011111011000`** |

Mais dois, que aparecem em todo programa: `D;JGT` (sem `dest`) vira `111` + `0001100` + `000` + `001` = `1110001100000001`, e `0;JMP` (o salto incondicional) vira `111` + `0101010` + `000` + `111` = `1110101010000111`.

### O programa inteiro, sem símbolos

Com essas três regras, o Assembler da Etapa 1 traduz o nosso programa por inteiro. À esquerda, o endereço de cada instrução na ROM; no meio, a instrução; à direita, o binário gerado:

```
 0  @0        0000000000000000
 1  D=M       1111110000010000
 2  @16       0000000000010000
 3  M=D       1110001100001000
 4  @17       0000000000010001
 5  M=0       1110101010001000
 6  @16       0000000000010000
 7  D=M       1111110000010000
 8  @20       0000000000010100
 9  D;JEQ     1110001100000010
10  @17       0000000000010001
11  D=M       1111110000010000
12  @16       0000000000010000
13  D=D+M     1111000010010000
14  @17       0000000000010001
15  M=D       1110001100001000
16  @16       0000000000010000
17  M=M-1     1111110010001000
18  @6        0000000000000110
19  0;JMP     1110101010000111
20  @17       0000000000010001
21  D=M       1111110000010000
22  @1        0000000000000001
23  M=D       1110001100001000
24  @24       0000000000011000
25  0;JMP     1110101010000111
```

Guarde esta listagem: ela é **exatamente** a saída que o Assembler completo deve produzir para a versão com símbolos do programa. A Etapa 2 é só o caminho de `@i` até `@16`.

## Etapa 2: tratando símbolos

### Três tipos de símbolo

Um símbolo, em Hack, é sempre um nome que **está no lugar de um número**. Há três origens para esse número:

| Tipo | Exemplo | Qual número representa | Quem define |
|---|---|---|---|
| **pré-definido** | `@R0`, `@SCREEN` | um endereço especial da RAM | a especificação da linguagem (tabela abaixo) |
| **rótulo** | `@LOOP`, `@END` | o endereço, na **ROM**, da instrução que vem logo depois de `(LOOP)` | o programador, com a declaração `(LOOP)` |
| **variável** | `@i`, `@sum` | um endereço da **RAM** escolhido pelo Assembler, a partir de 16 | ninguém: qualquer símbolo que não seja pré-definido nem rótulo é uma variável |

Os pré-definidos são 23:

| Símbolo | Valor | Símbolo | Valor |
|---|---|---|---|
| `R0` ... `R15` | 0 ... 15 | `SP` | 0 |
| `SCREEN` | 16384 | `LCL` | 1 |
| `KBD` | 24576 | `ARG` | 2 |
| | | `THIS` | 3 |
| | | `THAT` | 4 |

(Vários deles têm o mesmo valor: `SP`, `R0` e o endereço `0` são a mesma coisa. Os nomes diferentes só documentam a intenção — `SP` é usado pelo VM Translator para o ponteiro da pilha, `R13`–`R15` como registradores de rascunho.)

As **variáveis** recebem endereços na ordem em que **aparecem pela primeira vez** no programa: a primeira variável nova recebe `16`, a segunda `17`, e assim por diante. No nosso exemplo, `i` aparece antes de `sum`, então `i = 16` e `sum = 17` — exatamente os números que usamos na Etapa 1.

Todos os três tipos são resolvidos da mesma forma: uma **tabela de símbolos**, que associa cada nome ao seu número. Traduzir `@símbolo` é consultar a tabela e traduzir o número, como na Etapa 1. A única questão é **quando** cada símbolo entra na tabela.

### O problema central: referência futura

Os pré-definidos podem entrar na tabela antes de qualquer leitura, e as variáveis podem entrar no momento em que aparecem pela primeira vez. Já os rótulos trazem um problema. Se o Assembler traduzisse o arquivo `.asm` em uma única leitura, de cima para baixo, ele bateria de frente com isto:

```
@END        // Qual é o endereço de END? Ainda não sabemos!
D;JEQ
... (dezenas de instruções) ...
(END)       // END só é definido AQUI
```

Ao processar `@END`, o montador ainda não leu a declaração `(END)` — ela vem depois, no arquivo. Não é possível gerar o binário de `@END` sem saber a que endereço da ROM esse símbolo se refere, e não saberemos isso até ler o programa inteiro. Esse é o problema da **referência futura** (*forward reference*). Pior: sem ter visto `(END)`, o montador nem sabe que `END` é um rótulo — pela regra acima, trataria `END` como uma **variável** nova e lhe daria um endereço de RAM.

### A solução: duas passagens

A saída elegante é dividir a montagem em fases, cada uma com um objetivo só:

**Inicialização — os pré-definidos.** Crie a tabela de símbolos e coloque nela os 23 símbolos pré-definidos.

**Primeira passagem — só os rótulos.** Percorra o arquivo do início ao fim, mantendo um contador de endereço da ROM. Toda vez que encontrar um **rótulo** (`(Xxx)`), registre `Xxx → contador_atual` na tabela — e **não** incremente o contador (rótulos são pseudo-comandos, não ocupam espaço na ROM). Toda vez que encontrar uma instrução real (A ou C), apenas incremente o contador. Nesta passagem nada é traduzido. Ao final, todo rótulo do programa tem um endereço conhecido — não importa se ele foi declarado antes ou depois de ser usado.

```
romAddr = 0
para cada linha do arquivo:
    se é rótulo (Xxx):
        tabela[Xxx] = romAddr        // registra; NÃO incrementa
    senão (é A-instruction ou C-instruction):
        romAddr = romAddr + 1
```

**Segunda passagem — gerar o binário.** Percorra o arquivo de novo, agora traduzindo cada instrução real, como na Etapa 1. Rótulos são ignorados (já cumpriram seu papel na primeira passagem e não geram código). Uma A-instruction com um número literal (`@10`) converte-se diretamente; uma A-instruction com um símbolo consulta a tabela — se ele já estiver lá (um pré-definido, um rótulo ou uma variável já vista antes), usa o endereço registrado; caso contrário, é a **primeira** aparição de uma variável, e o montador aloca para ela o próximo endereço de RAM disponível, começando em 16. Uma C-instruction é traduzida pelas três tabelas da Etapa 1.

```
ramAddr = 16     // variáveis começam em RAM[16]
para cada linha do arquivo:
    se é rótulo: ignora (não gera código)
    se é A-instruction (@símbolo):
        se símbolo é numérico: endereco = número
        senão se símbolo está na tabela: endereco = tabela[símbolo]
        senão:                                     // nova variável
            tabela[símbolo] = ramAddr
            endereco = ramAddr
            ramAddr = ramAddr + 1
        emite 16 bits: 0 + endereco em binário (15 bits)
    se é C-instruction (dest=comp;jump):
        emite 16 bits: 111 + comp(7 bits) + dest(3 bits) + jump(3 bits)
```

Note por que a ordem importa: como os rótulos entram na tabela **antes** da segunda passagem, quando esta encontra `@END`, `END` já está lá — e nunca é confundido com uma variável.

## Um exemplo completo, passo a passo

Vamos aplicar as três fases ao programa do início do capítulo.

**Inicialização.** A tabela começa com os 23 pré-definidos: `R0`–`R15` (0–15), `SP` (0), `LCL` (1), `ARG` (2), `THIS` (3), `THAT` (4), `SCREEN` (16384) e `KBD` (24576).

**Primeira passagem** — só os rótulos importam. Para cada linha, o valor de `romAddr` antes de processá-la e o que acontece:

| Comando | Tipo | `romAddr` antes | Ação |
|---|---|---|---|
| `@R0` | A | 0 | `romAddr` → 1 |
| `D=M` | C | 1 | `romAddr` → 2 |
| `@i` | A | 2 | `romAddr` → 3 |
| `M=D` | C | 3 | `romAddr` → 4 |
| `@sum` | A | 4 | `romAddr` → 5 |
| `M=0` | C | 5 | `romAddr` → 6 |
| `(LOOP)` | L | **6** | registra `LOOP → 6`; **não** incrementa |
| `@i` | A | 6 | `romAddr` → 7 |
| `D=M` | C | 7 | `romAddr` → 8 |
| `@END` | A | 8 | `romAddr` → 9 |
| `D;JEQ` | C | 9 | `romAddr` → 10 |
| `@sum` | A | 10 | `romAddr` → 11 |
| `D=M` | C | 11 | `romAddr` → 12 |
| `@i` | A | 12 | `romAddr` → 13 |
| `D=D+M` | C | 13 | `romAddr` → 14 |
| `@sum` | A | 14 | `romAddr` → 15 |
| `M=D` | C | 15 | `romAddr` → 16 |
| `@i` | A | 16 | `romAddr` → 17 |
| `M=M-1` | C | 17 | `romAddr` → 18 |
| `@LOOP` | A | 18 | `romAddr` → 19 |
| `0;JMP` | C | 19 | `romAddr` → 20 |
| `(END)` | L | **20** | registra `END → 20`; **não** incrementa |
| `@sum` | A | 20 | `romAddr` → 21 |
| `D=M` | C | 21 | `romAddr` → 22 |
| `@R1` | A | 22 | `romAddr` → 23 |
| `M=D` | C | 23 | `romAddr` → 24 |
| `(HALT)` | L | **24** | registra `HALT → 24`; **não** incrementa |
| `@HALT` | A | 24 | `romAddr` → 25 |
| `0;JMP` | C | 25 | `romAddr` → 26 |

Tabela ao final da primeira passagem: os pré-definidos, mais `{ LOOP: 6, END: 20, HALT: 24 }`. O programa tem 26 instruções reais (ROM 0 a 25); as três declarações de rótulo não ocupam espaço.

**Segunda passagem** — agora sim, traduzindo:

| ROM | Instrução | Resolução | Binário |
|---|---|---|---|
| 0 | `@R0` | pré-definido: `R0 = 0` | `0000000000000000` |
| 1 | `D=M` | `111` + comp `M` + dest `D` + jump `—` | `1111110000010000` |
| 2 | `@i` | **variável nova** → `i = 16` (entra na tabela) | `0000000000010000` |
| 3 | `M=D` | `111` + comp `D` + dest `M` + jump `—` | `1110001100001000` |
| 4 | `@sum` | **variável nova** → `sum = 17` (entra na tabela) | `0000000000010001` |
| 5 | `M=0` | `111` + comp `0` + dest `M` + jump `—` | `1110101010001000` |
| | `(LOOP)` | rótulo: ignorado, não gera código | — |
| 6 | `@i` | variável já vista: `i = 16` | `0000000000010000` |
| 7 | `D=M` | `111` + comp `M` + dest `D` + jump `—` | `1111110000010000` |
| 8 | `@END` | rótulo da 1ª passagem: `END = 20` | `0000000000010100` |
| 9 | `D;JEQ` | `111` + comp `D` + dest `—` + jump `JEQ` | `1110001100000010` |
| 10 | `@sum` | variável já vista: `sum = 17` | `0000000000010001` |
| 11 | `D=M` | `111` + comp `M` + dest `D` + jump `—` | `1111110000010000` |
| 12 | `@i` | variável já vista: `i = 16` | `0000000000010000` |
| 13 | `D=D+M` | `111` + comp `D+M` + dest `D` + jump `—` | `1111000010010000` |
| 14 | `@sum` | variável já vista: `sum = 17` | `0000000000010001` |
| 15 | `M=D` | `111` + comp `D` + dest `M` + jump `—` | `1110001100001000` |
| 16 | `@i` | variável já vista: `i = 16` | `0000000000010000` |
| 17 | `M=M-1` | `111` + comp `M-1` + dest `M` + jump `—` | `1111110010001000` |
| 18 | `@LOOP` | rótulo da 1ª passagem: `LOOP = 6` | `0000000000000110` |
| 19 | `0;JMP` | `111` + comp `0` + dest `—` + jump `JMP` | `1110101010000111` |
| | `(END)` | rótulo: ignorado, não gera código | — |
| 20 | `@sum` | variável já vista: `sum = 17` | `0000000000010001` |
| 21 | `D=M` | `111` + comp `M` + dest `D` + jump `—` | `1111110000010000` |
| 22 | `@R1` | pré-definido: `R1 = 1` | `0000000000000001` |
| 23 | `M=D` | `111` + comp `D` + dest `M` + jump `—` | `1110001100001000` |
| | `(HALT)` | rótulo: ignorado, não gera código | — |
| 24 | `@HALT` | rótulo da 1ª passagem: `HALT = 24` | `0000000000011000` |
| 25 | `0;JMP` | `111` + comp `0` + dest `—` + jump `JMP` | `1110101010000111` |

Tabela ao final da segunda passagem: a anterior, mais `{ i: 16, sum: 17 }`. Compare a coluna "Binário" com a listagem da Etapa 1: é idêntica.

Repare no ponto crucial: `@END` (ROM 8), quando alcançado na segunda passagem, já resolve imediatamente para `20` — porque a **primeira** passagem já tinha visitado `(END)` antes de a segunda sequer começar. `@LOOP` (ROM 18) e `@HALT` (ROM 24), que aparecem **depois** das suas declarações, também funcionariam numa passagem só; é o `@END` que exige as duas. Essa separação de responsabilidades — uma passagem só para mapear rótulos, sem gerar nada; outra só para gerar código, sem se preocupar com posições futuras — resolve o problema da referência futura sem nenhum *backtracking* ou reprocessamento.

## Arquitetura modular

A implementação se organiza naturalmente em quatro módulos com responsabilidades bem separadas — a mesma disciplina de "uma classe, uma responsabilidade" que seguimos desde o `Scanner`/`Parser` da Parte 1:

| Módulo | Responsabilidade | Interface sugerida |
|---|---|---|
| **Parser** | Lê o `.asm`, ignora espaços e comentários, classifica cada comando e extrai suas partes | `hasMoreCommands()`, `advance()`, `commandType()` (`A_COMMAND`, `C_COMMAND`, `L_COMMAND`), `symbol()`, `dest()`, `comp()`, `jump()` |
| **Code** | Traduz cada mnemônico para sua codificação binária, segundo as tabelas da Etapa 1 | `dest(String)` → 3 bits, `comp(String)` → 7 bits, `jump(String)` → 3 bits |
| **SymbolTable** | Um mapa símbolo → endereço, pré-carregado com os 23 símbolos pré-definidos | `addEntry(símbolo, endereço)`, `contains(símbolo)`, `getAddress(símbolo)` |
| **Main** | Orquestra a inicialização e as duas passagens, usando os três módulos acima | — |

O `Parser` é deliberadamente burro: não sabe nada sobre símbolos, endereços ou binário — só sintaxe. Para o comando `D=M+1;JGT`, por exemplo, ele só sabe dizer que é um `C_COMMAND` com `dest()` = `"D"`, `comp()` = `"M+1"` e `jump()` = `"JGT"`; e, para `@LOOP` ou `(LOOP)`, que `symbol()` = `"LOOP"`. O `Code` é uma coleção de tabelas — praticamente uma cópia executável das tabelas da Etapa 1: `comp("M+1")` = `"1110111"`, `dest("D")` = `"010"`, `jump("JGT")` = `"001"`. O `SymbolTable` é só um dicionário. Toda a "inteligência" das duas passagens — quando registrar um rótulo, quando alocar uma variável — vive no `Main`, o único módulo que conhece os outros três:

```java
// esboço do módulo Main — primeira passagem
int romAddr = 0;
while (parser.hasMoreCommands()) {
    parser.advance();
    if (parser.commandType() == L_COMMAND) {
        symbolTable.addEntry(parser.symbol(), romAddr);
    } else {
        romAddr++;
    }
}

// segunda passagem: recomeça a leitura do início do arquivo
parser = new Parser(inputFile);
int nextRam = 16;
while (parser.hasMoreCommands()) {
    parser.advance();
    switch (parser.commandType()) {
        case L_COMMAND -> { /* ignora */ }
        case A_COMMAND -> {
            String sym = parser.symbol();
            int address;
            if (isNumeric(sym)) {
                address = Integer.parseInt(sym);
            } else if (symbolTable.contains(sym)) {
                address = symbolTable.getAddress(sym);
            } else {
                address = nextRam++;
                symbolTable.addEntry(sym, address);
            }
            output.write(toBinary16(address));
        }
        case C_COMMAND -> {
            String bits = "111"
                + code.comp(parser.comp())
                + code.dest(parser.dest())
                + code.jump(parser.jump());
            output.write(bits);
        }
    }
}
```

Para `D=M+1;JGT`, o `case C_COMMAND` monta `"111"` + `"1110111"` + `"010"` + `"001"` = `1111110111010001`.

## Validação

O Nand2Tetris fornece os programas de teste do Project 6 e um Assembler de referência (pasta `tools/`), que gera o `.hack` esperado — comparação exata de texto, de novo, é o critério de correção. Também dá para carregar o `.hack` gerado no CPU Emulator e ver o programa rodando (apêndice D).

| Programa | O que faz | Versão sem símbolos |
|---|---|---|
| `Add.asm` | calcula `RAM[0] = 2 + 3` | o próprio `Add.asm` (não tem símbolos) |
| `Max.asm` | `RAM[2] = max(RAM[0], RAM[1])` | `MaxL.asm` |
| `Rect.asm` | desenha um retângulo no canto superior esquerdo da tela, com 16 pixels de largura e `RAM[0]` de altura | `RectL.asm` |
| `Pong.asm` | o jogo Pong, gerado a partir de código Jack pelo compilador e pelo VM Translator — um programa longo, cheio de rótulos e variáveis | `PongL.asm` |

A ordem de teste segue as duas etapas do capítulo:

1. **Etapa 1**: o Assembler sem tratamento de símbolos deve traduzir corretamente `Add.asm`, `MaxL.asm`, `RectL.asm` e `PongL.asm` — as versões `L` são os mesmos programas, já com todo símbolo trocado por número.
2. **Etapa 2**: com a tabela de símbolos e as duas passagens, `Max.asm`, `Rect.asm` e `Pong.asm` devem gerar **exatamente** o mesmo `.hack` que as suas versões `L`.

É a mesma disciplina incremental que usamos desde o capítulo 2: cada etapa é testada antes de servir de base para a seguinte.

## Teste rápido

??? question "1. Traduza `AM=M-1` para binário."
    `dest = AM` → `101`; `comp = M-1` → `a = 1`, `c = 110010` → `1110010`; `jump` vazio → `000`. Resultado: `111` + `1110010` + `101` + `000` = `1111110010101000`.

??? question "2. Que instrução Assembly corresponde ao binário `1110001100000001`?"
    Começa com `1`: é uma C-instruction. `a c1..c6` = `0001100` → `comp = D`; `d` = `000` → sem `dest`; `j` = `001` → `JGT`. É `D;JGT`: "salte para o endereço em `A` se `D > 0`".

??? question "3. Por que a primeira passagem não incrementa o contador ao encontrar `(LOOP)`?"
    Porque uma declaração de rótulo não é uma instrução: não gera código e não ocupa um endereço na ROM. O rótulo apenas dá um nome ao endereço da **próxima** instrução real — que é justamente o valor atual do contador.

??? question "4. Num programa que usa, nesta ordem, `@contador`, `@SCREEN`, `@FIM`, `@x` e `@contador`, e que declara `(FIM)` mais adiante, que endereços recebem as variáveis?"
    `contador = 16` e `x = 17`. `SCREEN` é pré-definido (16384), e `FIM` é um rótulo — já está na tabela desde a primeira passagem, então não consome um endereço de variável. A segunda aparição de `@contador` reusa o 16.

## Fechando o livro

Com o Assembler completo, a cadeia de tradução está fechada de ponta a ponta:

```
Jack (Parte 1: léxico + sintático)
    ↓ Compilador Jack → VM (Parte 2)
Código VM
    ↓ VM Translator (capítulos 12-13)
Assembly Hack
    ↓ Assembler (este capítulo)
Código de máquina Hack
    ↓ CPU Hack
Execução
```

Cada elo dessa corrente — tokenizador, parser, tabela de símbolos, gerador de código VM, tradutor de VM para Assembly, montador — foi construído (ou especificado, nos capítulos de teoria + atividade) neste livro. É o ciclo completo que abrimos no capítulo 0: do texto-fonte ao hardware, sem nenhuma "caixa-preta" no meio do caminho.
