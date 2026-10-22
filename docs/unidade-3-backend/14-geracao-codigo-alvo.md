# 14. Geração de código alvo (consolidação teórica)

Antes de construir o Assembler (capítulo 15), vale um momento de retrospectiva: o que exatamente aconteceu nos capítulos 11 a 13, e por que essa arquitetura em particular?

## O que já construímos

Percorremos, sem perceber explicitamente, uma tradução em **duas etapas** dentro do que costuma ser chamado, em conjunto, de "geração de código alvo":

```
Código VM (IR, capítulo 7)
    ↓ VM Translator (capítulos 12-13)
Assembly Hack (simbólico, capítulo 11)
    ↓ Assembler (capítulo 15)
Código de máquina Hack (binário)
```

O VM Translator não é, estritamente, "geração de código alvo" no sentido mais estrito do termo (ele traduz entre duas representações textuais, ambas ainda "legíveis" por um humano treinado) — mas cumpre exatamente o papel que a geração de código alvo cumpre em um compilador tradicional: transformar uma representação independente de máquina em uma representação específica de uma arquitetura real. A diferença sutil é que aqui esse trabalho foi dividido em duas sub-etapas (VM → Assembly, Assembly → binário) em vez de uma só (IR → binário direto) — uma divisão que, como vimos no capítulo 6, tem tudo a ver com simplicidade de implementação.

## Por que uma etapa intermediária extra

Um compilador "clássico" de livro-texto costuma ir direto de uma IR para código de máquina binário. O Nand2Tetris insere Assembly Hack como uma parada intermediária **legível por humanos** entre a VM e o binário, por uma razão pedagógica e prática ao mesmo tempo: cada etapa da tradução — VM → Assembly (capítulos 12-13) e Assembly → binário (capítulo 15) — se torna pequena e independentemente verificável. Você pode ler o `.asm` gerado pelo seu VM Translator e confirmar, linha a linha, que ele faz sentido, antes de sequer pensar no Assembler. Compiladores de produção fazem escolhas semelhantes por razões de engenharia: o LLVM, por exemplo, tem uma IR textual (a "LLVM IR") que pode ser inspecionada exatamente por esse motivo — depurar em texto é mais fácil que depurar em binário.


## O que cada etapa resolve

Cada tradutor da cadeia elimina uma abstração específica — e só ela. Ver a cadeia desse jeito ajuda a localizar um bug: se o problema envolve nomes de variáveis, ele está no compilador; se envolve endereços de RAM, no VM Translator; se envolve bits, no Assembler.

| Etapa | Abstração eliminada | Como | Capítulos |
|---|---|---|---|
| Jack → VM | Nomes de variáveis, classes, objetos e estruturas de controle | Tabela de símbolos (nome → segmento + índice), `pointer`/`this`/`that` para objetos e arrays, rótulos para `if`/`while` | 7–10 |
| VM → Assembly | Pilha e segmentos virtuais | Registradores `SP`, `LCL`, `ARG`, `THIS`, `THAT` em `RAM[0..4]`; cada comando VM vira uma sequência fixa de instruções; `call`/`return` montam e desmontam o frame | 11–13 |
| Assembly → binário | Símbolos e mnemônicos | Duas passagens: rótulos → endereços de ROM, variáveis → endereços de RAM a partir de 16; mnemônicos → bits pelas tabelas do capítulo 11 | 11, 15 |

Repare que nenhuma etapa "olha para trás": o VM Translator não sabe que o código veio de Jack, e o Assembler não sabe que o Assembly veio de uma VM. É essa independência que permite testar cada tradutor sozinho, com os arquivos de teste de cada projeto do Nand2Tetris.

## O que vem a seguir

Falta o último elo: o Assembler. O capítulo 15 mostra como converter o Assembly Hack especificado no capítulo 11 em código binário, resolvendo os símbolos em duas passagens.
