# 0. Introdução

Sejam bem-vindos à disciplina de Compiladores. Ao final deste livro, você terá construído — peça por peça — um compilador completo para uma linguagem de programação real, além do tradutor e do montador que transformam a saída desse compilador em código executável por um processador de verdade. Não vamos apenas estudar a teoria por trás dessa "mágica"; vamos colocar a mão na massa.

## Compilador, interpretador, máquina virtual

Um **compilador** traduz um programa escrito em uma linguagem-fonte para um programa equivalente em outra linguagem — tipicamente uma linguagem de mais baixo nível, mais próxima do hardware. Um **interpretador**, por outro lado, executa diretamente o programa-fonte (ou uma representação dele), produzindo o resultado sem gerar um programa traduzido como produto final. Um interpretador simples pode ser visto como uma rotina que percorre a árvore sintática (AST) do programa e, para cada tipo de nó, executa uma ação correspondente:

```java
@Override
public Void visitWhileStmt(Stmt.While stmt) {
    while (isTruthy(evaluate(stmt.condition))) {
        execute(stmt.body);
    }
    return null;
}
```

Entre esses dois extremos existe um espaço de projeto rico. Um compilador pode gerar código para uma **máquina virtual** — um processador hipotético, simples de implementar em software — em vez de gerar código nativo diretamente. Essa é exatamente a estratégia que vamos adotar neste curso: um front-end traduz a linguagem-fonte para uma representação intermediária baseada em pilha (a VM Hack), e um back-end separado traduz essa representação intermediária para o código de máquina de um processador real (o Hack, do projeto Nand2Tetris). Separar essas duas etapas é o que torna o projeto de um semestre inteiro tratável: cada etapa é pequena o suficiente para ser compreendida e implementada isoladamente, mas juntas elas formam uma cadeia de tradução completa, do código-fonte ao hardware.

Vale comparar as três estratégias lado a lado, porque cada uma resolve um problema diferente:

| Estratégia | Vantagens | Desvantagens |
|---|---|---|
| Interpretador (percorre a AST) | Simplicidade de implementação, retorno imediato | Execução mais lenta — reinterpreta a árvore a cada execução |
| Compilador direto (linguagem → código de máquina) | Desempenho máximo | Pouca portabilidade — um back-end por arquitetura de hardware |
| Compilador com VM (linguagem → IR → código de máquina) | Portabilidade (a IR é a mesma para qualquer hardware) + desempenho razoável | Mais componentes para implementar (duas traduções em vez de uma) |

Esse último modelo — que os autores do Nand2Tetris chamam de **compilação em duas camadas** (*two-tier compilation*) — é o que vamos construir: uma camada traduz Jack para a VM Hack (portável, independente de hardware), e outra traduz a VM Hack para o Assembly de uma arquitetura real. É o mesmo modelo usado pela JVM (Java → bytecode → código nativo) ou pelo CLR do .NET, só que em uma escala didática.

Segundo Keith Cooper, um bom compilador contém um microcosmo da ciência da computação: algoritmos gulosos, técnicas de busca heurística, algoritmos em grafos, programação dinâmica, autômatos finitos e de pilha, algoritmos de ponto fixo. Ele lida com alocação dinâmica, sincronização, nomeação, localidade e gerência de hierarquia de memória. Não é exagero dizer que construir um compilador é uma forma de revisar, de forma aplicada, boa parte do que se aprende em um curso de Ciência da Computação.

## O pipeline de um compilador

É comum visualizar um compilador em três grandes fases:

1. **Front-end** — análise: entende o programa-fonte e verifica se ele é válido.
2. **Middle-end** — tradução: converte o programa entendido em uma representação intermediária (IR).
3. **Back-end** — geração de código: traduz a representação intermediária para o código-alvo final.

Cada uma dessas fases se divide, por sua vez, em etapas menores:

- **Análise léxica**: agrupa os caracteres do texto-fonte em *tokens* (palavras-chave, identificadores, números, símbolos).
- **Análise sintática**: organiza os tokens de acordo com a gramática da linguagem, verificando que a estrutura do programa é válida.
- **Análise semântica**: verifica regras que a gramática sozinha não consegue expressar (tipos, escopo, uso de identificadores declarados).
- **Geração de código intermediário**: traduz o programa (já validado) para uma representação intermediária.
- **Geração de código alvo**: traduz a representação intermediária para o código final executável.

Este livro segue esse pipeline clássico de livro-texto — a mesma ordem usada por praticamente todo curso introdutório de compiladores — e não a ordem "bottom-up" (hardware → VM → linguagem) do livro original do Nand2Tetris. Construiremos primeiro o front-end (Parte 1), depois o middle-end — o compilador Jack → VM (Parte 2) — e por fim o back-end, em duas partes: o tradutor de VM (Parte 3) e o montador (Parte 4), sempre com um artefato funcional ao final de cada etapa.

## Por que o Nand2Tetris

O projeto [Nand2Tetris](https://www.nand2tetris.org/) (Nisan & Schocken) propõe construir um sistema de computação completo a partir de portas NAND, subindo até uma linguagem de programação de alto nível. Vamos usar apenas a metade "de cima" dessa pilha — a parte de compiladores — mas ela já nos dá tudo que precisamos: uma linguagem orientada a objetos completa (**Jack**), uma máquina virtual baseada em pilha bem especificada (a **VM Hack**) e uma arquitetura de máquina real e simples (o **Hack**), com seu montador. É um projeto pequeno o suficiente para caber em um semestre e completo o suficiente para não deixar buracos conceituais — você vai implementar cada elo da corrente entre o texto-fonte e a máquina.

Nosso compilador será dividido em dois projetos que se comunicam através da VM Hack:

- Um **compilador** que traduz programas Jack para código VM.
- Um **tradutor de VM** (VM Translator) que traduz código VM para Assembly Hack, que por sua vez é traduzido para código de máquina por um **montador** (Assembler).

Ao longo do curso:

- A análise léxica pode ser implementada manualmente ou com o auxílio de expressões regulares.
- Usaremos um analisador sintático **preditivo** (descendente recursivo), implementado manualmente.
- Não geraremos uma AST explícita: a geração de código será feita diretamente durante a análise sintática, produzindo código VM.
- No fim, esse código VM será traduzido para Assembly Hack e, em seguida, para código de máquina.

## Como este livro está organizado

| Parte | Capítulos | Foco | Fecha com |
|---|---|---|---|
| 1 — Front-end | 0–5 | Fundamentos teóricos, a linguagem Jack, análise léxica e sintática | Um parser completo para Jack (Project 10) |
| 2 — O Compilador | 6–10 | Representações intermediárias, especificação da VM Hack, tabela de símbolos, geração de código VM | Compilador Jack → VM completo (Project 11) |
| 3 — O Tradutor VM | 11–14 | Especificação do Assembly Hack, VM Translator (partes 1 e 2), consolidação da geração de código alvo | VM Translator completo (Projects 7 e 8) |
| 4 — O Montador | 15 | Montagem em duas passagens | Assembler Hack (Project 6) |

A linguagem de demonstração usada no livro é **Java** — todo o código é construído junto com você, capítulo a capítulo, no espírito de [*Crafting Interpreters*](https://craftinginterpreters.com/contents.html). Mas você não precisa usar Java nas suas atividades práticas: é livre para implementar em qualquer linguagem com a qual se sinta confortável. O que importa é entender cada etapa a fundo o suficiente para reproduzi-la.

Vamos começar.
