# 15. Construindo o Assembler

Chegamos à última peça da cadeia de tradução: o **Assembler** (Project 6), que converte um arquivo `.asm` escrito na linguagem do capítulo 11 em um arquivo `.hack` — a sequência de instruções binárias de 16 bits que a CPU Hack executa diretamente. Como o VM Translator (capítulos 12-13), este capítulo é teoria + especificação + arquitetura — a implementação completa é sua atividade prática, comparando a saída com os testes oficiais do Nand2Tetris.

## O problema central: referência futura

Se o Assembler traduzisse o arquivo `.asm` em uma única leitura, de cima para baixo, ele bateria de frente com um problema irresolúvel:

```
@END        // Qual é o endereço de END? Ainda não sabemos!
0;JMP
... (dezenas de instruções) ...
(END)       // END só é definido AQUI
```

Ao processar `@END`, o montador ainda não leu a declaração `(END)` — ela vem depois, no arquivo. Não é possível gerar o binário de `@END` sem saber a que endereço ROM esse símbolo se refere, e não saberemos isso até ler o programa inteiro. Esse é o problema da **referência futura** (*forward reference*), e ele não tem solução em uma única passagem.

## A solução: duas passagens

A saída elegante é dividir a montagem em duas fases, cada uma com um objetivo só:

**Primeira passagem — construir a tabela de símbolos (só rótulos).** Percorra o arquivo do início ao fim, mantendo um contador de endereço ROM. Toda vez que encontrar um **rótulo** (`(Xxx)`), registre `Xxx → contador_atual` na tabela — e **não** incremente o contador (rótulos são pseudo-comandos, não ocupam espaço em ROM). Toda vez que encontrar uma instrução real (A ou C), apenas incremente o contador. Ao final desta passagem, todo rótulo do programa tem um endereço ROM conhecido — não importa se ele foi declarado antes ou depois de ser usado.

```
romAddr = 0
para cada linha do arquivo:
    se é rótulo (Xxx):
        tabela[Xxx] = romAddr        // registra; NÃO incrementa
    senão (é A-instruction ou C-instruction):
        romAddr = romAddr + 1
```

**Segunda passagem — gerar o binário.** Percorra o arquivo de novo, agora traduzindo cada instrução real. Rótulos são ignorados (já cumpriram seu papel na primeira passagem — não geram código). Uma A-instruction com um número literal (`@10`) converte-se diretamente; uma A-instruction com um símbolo consulta a tabela — se já estiver lá (um rótulo, ou uma variável já vista antes), usa o endereço registrado; caso contrário, é a **primeira** aparição de uma variável, e o montador aloca para ela o próximo endereço de RAM disponível, começando em 16. Uma C-instruction é decomposta em `comp`/`dest`/`jump` e cada parte é traduzida pelas tabelas do capítulo 11.

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

## Um exemplo completo, passo a passo

Considere o programa abaixo, uma variação da somatória do capítulo 11: aqui, `i` começa em `RAM[0]` e é decrementado até zero, e o resultado vai para `RAM[1]`:

```
@R0
D=M
@i
M=D
@sum
M=0
(LOOP)
@i
D=M
@END
D;JEQ
@sum
D=M
@i
D=D+M
@sum
M=D
@i
M=M-1
@LOOP
0;JMP
(END)
@sum
D=M
@R1
M=D
```

**Primeira passagem** — só rótulos importam:

| Comando | Tipo | `romAddr` antes | Ação |
|---|---|---|---|---|
| `@R0` | A | 0 | → 1 |
| `D=M` | C | 1 | → 2 |
| `@i` | A | 2 | → 3 |
| `M=D` | C | 3 | → 4 |
| `@sum` | A | 4 | → 5 |
| `M=0` | C | 5 | → 6 |
| `(LOOP)` | L | **6** | registra `LOOP → 6`, não incrementa |
| ... | ... | ... | ... segue incrementando |
| `(END)` | L | **20** | registra `END → 20`, não incrementa |
| ... | ... | ... | ... |

Tabela ao final da primeira passagem: `{ LOOP: 6, END: 20 }` (mais os pré-definidos: `SP: 0, LCL: 1, ..., R0: 0, ..., R15: 15, SCREEN: 16384, KBD: 24576`).

**Segunda passagem** — agora sim, traduzindo:

| Instrução | Resolução | Binário |
|---|---|---|
| `@R0` | pré-definido, `R0 = 0` | `0000000000000000` |
| `D=M` | `comp=M, dest=D, jump=—` | `1111110000010000` |
| `@i` | símbolo novo → aloca `RAM[16]`; tabela agora tem `i: 16` | `0000000000010000` |
| `M=D` | `comp=D, dest=M` | `1110001100001000` |
| `@sum` | símbolo novo → aloca `RAM[17]` | `0000000000010001` |
| `(LOOP)` | rótulo → ignorado, não gera linha | — |
| `@i` | já na tabela (`16`) | `0000000000010000` |
| `@END` | já na tabela, da 1ª passagem (`20`) | `0000000000010100` |
| ... | ... | ... |

Repare no ponto crucial: `@END`, quando alcançado na segunda passagem, já resolve imediatamente para `20` — porque a **primeira** passagem já tinha visitado `(END)` antes de a segunda passagem sequer começar. É essa separação de responsabilidades — uma passagem só para mapear rótulos, sem gerar nada; outra só para gerar código, sem se preocupar com posições futuras — que resolve o problema da referência futura sem qualquer *backtracking* ou reprocessamento.

## Arquitetura modular

A implementação se organiza naturalmente em quatro módulos com responsabilidades bem separadas — a mesma disciplina de "uma classe, uma responsabilidade" que seguimos desde o `Scanner`/`Parser` da Parte 1:

| Módulo | Responsabilidade |
|---|---|
| **Parser** | Lê o `.asm` linha a linha, ignora espaços/comentários, classifica cada comando (`A_COMMAND`, `C_COMMAND`, `L_COMMAND`) e extrai suas partes (`symbol()`, `dest()`, `comp()`, `jump()`) |
| **SymbolTable** | Um mapa símbolo → endereço, pré-carregado com os símbolos pré-definidos do capítulo 11 (`addEntry`, `contains`, `getAddress`) |
| **Code** | Traduz cada mnemônico (`dest`, `comp`, `jump`) para sua codificação binária, segundo as tabelas do capítulo 11 |
| **Main** | Orquestra as duas passagens, usando os três módulos acima |

O `Parser` é deliberadamente burro — ele não sabe nada sobre símbolos ou endereços, só sintaxe. O `SymbolTable` é só um dicionário. O `Code` é uma coleção de tabelas de tradução (praticamente uma cópia executável das tabelas do capítulo 11). Toda a "inteligência" das duas passagens — quando registrar um rótulo, quando alocar uma variável — vive no `Main`, que é o único módulo que orquestra os outros três.

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

// segunda passagem
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

## Validação

O Nand2Tetris fornece os programas `.asm` de teste (`Add.asm`, `Max.asm`, `Rect.asm`, `Pong.asm`) e o Assembler de referência (pasta `tools/`), que gera o `.hack` esperado — comparação exata de texto, de novo, é o critério de correção. Vale testar em ordem crescente de complexidade: primeiro os programas sem símbolo nenhum (`Add.asm` e as versões `MaxL.asm`, `RectL.asm` e `PongL.asm`, que já vêm com os símbolos trocados por números), depois com rótulos, depois com variáveis, depois combinando os dois — exatamente a mesma disciplina incremental que usamos desde o capítulo 2.

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
