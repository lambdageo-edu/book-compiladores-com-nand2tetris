# Compiladores — do zero ao Hack

Bem-vindo(a) à disciplina de Compiladores. Este livro constrói, parte a parte, um compilador completo para a linguagem **Jack** (Nand2Tetris) e a infraestrutura que o sustenta — máquina virtual, montador e a arquitetura Hack — no espírito de *[Crafting Interpreters](https://craftinginterpreters.com/contents.html)*: você acompanha o código sendo construído, método a método, capítulo a capítulo.

## Como usar este livro

- A linguagem de demonstração é **Java** — todo o código incremental do livro é escrito nela. Você é livre para implementar suas atividades práticas em qualquer linguagem.
- Cada capítulo parte de onde o anterior parou. Recomenda-se seguir a ordem.
- O livro está estruturado em quatro partes lógicas que cobrem o compilador Jack, o tradutor para a Máquina Virtual e o montador Assembly.
- Cada parte do livro conclui etapas completas de tradução que se relacionam diretamente com a pilha de software do Nand2Tetris.

## As quatro partes

| Parte | Foco | Projetos Relacionados (Nand2Tetris) |
|---|---|---|
| 1 — Front-end (Análise Sintática) | Linguagens, gramáticas, analisador léxico e analisador sintático para a linguagem Jack. | Projeto 10 (Compiler I - Syntax Analysis) |
| 2 — O Compilador (Geração de Código VM) | Representações intermediárias, especificação da VM Hack, tabela de símbolos e geração de código VM para Jack. | Projeto 11 (Compiler II - Code Generation) |
| 3 — O Tradutor VM (Geração de Código Assembly) | Especificação do Assembly Hack e tradução de aritmética, memória, controle de fluxo e funções de código VM para Assembly Hack. | Projetos 7 e 8 (VM Translator) |
| 4 — O Montador (Assembly para Código de Máquina) | Construção do Montador (Assembler), que converte o Assembly Hack em binário em duas passagens. | Projeto 6 (Assembler) |

## Pré-requisitos

- Programação básica (qualquer linguagem imperativa/orientada a objetos).
- Estruturas de dados elementares (pilhas, listas, árvores, tabelas hash).
- Não é necessário conhecimento prévio de Java, arquitetura de computadores ou teoria da computação — o livro constrói o que for preciso ao longo do caminho.

## Ambiente e ferramentas

Veja o [Apêndice D](apendices/d-ambiente-e-ferramentas.md) para instalação do software oficial do Nand2Tetris (CPU Emulator, VM Emulator, Assembler) e configuração do ambiente Java.

## Referências

Veja o [Apêndice E](apendices/e-referencias.md) para a bibliografia completa.
