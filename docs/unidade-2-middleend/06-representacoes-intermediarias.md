# 6. Representações intermediárias

Com o analisador sintático pronto (Parte 1), o compilador já *entende* a estrutura de um programa Jack — falta decidir *para onde* traduzi-lo. Antes de especificar a VM Hack (capítulo 7), construir a tabela de símbolos que liga os identificadores de Jack aos segmentos dessa VM (capítulo 8) e começar a gerar código (capítulos 9 e 10), vale um passo atrás: por que compiladores usam uma representação intermediária (IR) em vez de traduzir a linguagem-fonte diretamente para código de máquina? Este capítulo é teoria pura — não constrói nenhum código — mas justifica cada decisão de projeto dos capítulos seguintes.

## O problema que uma IR resolve

Um compilador "monolítico", que traduz diretamente de uma linguagem-fonte para uma arquitetura-alvo específica, tem um problema de escala: para **N** linguagens-fonte e **M** arquiteturas-alvo, seriam necessários **N × M** compiladores completos, cada um do zero. Introduzir uma representação intermediária comum quebra esse produto em uma soma: um **front-end** por linguagem (que traduz para a IR) e um **back-end** por arquitetura (que traduz da IR para o alvo) — **N + M** componentes, cada um reutilizável independentemente dos demais.

```
Front-ends (linguagens)        IR comum        Back-ends (arquiteturas)
Clang (C/C++)      ─┐                       ┌─ x86-64
Rust                ├──────→  LLVM IR  ─────┼─ ARM
Swift               ┘                       └─ WebAssembly
```

O exemplo mais bem-sucedido dessa arquitetura é o **LLVM**: dezenas de linguagens-fonte (C/C++ via Clang, Rust, Swift, Julia, Kotlin...) compilam para a mesma LLVM IR, e um conjunto compartilhado de otimizações e back-ends atende a todas elas de uma vez. Uma melhoria no otimizador do LLVM beneficia simultaneamente Rust, Swift e C++ — sem que nenhuma dessas linguagens precise reimplementar nada.

Formalmente (Aho, Lam, Sethi & Ullman): uma IR é um conjunto de estruturas de dados que captura o significado semântico de um programa, abstrato o suficiente para ser independente da linguagem-fonte, mas concreto o suficiente para estar próximo da máquina-alvo. Uma boa IR busca completude (não perde informação semântica), clareza (estrutura compreensível, não um binário opaco), simplicidade (fácil de gerar e de processar) e portabilidade.

## Representações gráficas: árvore de derivação e AST

A **árvore de derivação** (*parse tree*), produzida diretamente pelo analisador sintático (Parte 1), inclui **todos** os símbolos da gramática — parênteses, ponto-e-vírgula, palavras-chave estruturais. Ela é fiel à sintaxe, mas carregada de detalhes irrelevantes para o significado do programa.

A **árvore sintática abstrata** (AST) retém apenas a estrutura essencial: operadores, operandos, e a hierarquia que expressa precedência e associatividade — mas descarta tudo que é "só sintaxe". Para uma AST não importa se um bloco é demarcado por chaves, por `begin`/`end`, ou por indentação; importa apenas "isto é um comando condicional, com esta condição, este ramo verdadeiro e este ramo falso".

```
Parse tree de "2 + 3 * 4"                AST equivalente

            expr                                +
          /  |   \                              / \
      expr   +    term                         2   *
       |        /  |   \                          / \
     term    term  *  factor                     3   4
       |       |        |
    factor  factor      4
       |       |
       2       3
```

A árvore de derivação acima segue a gramática `expr → expr + term | term`, `term → term * factor | factor`, `factor → número` — repare que ela guarda até o `+` e o `*` como folhas, enquanto a AST os promove a nós internos e descarta os não-terminais intermediários.

Atenção: o exemplo acima segue a precedência usual da matemática (`*` antes de `+`), como em C ou Java. Em Jack não é assim — como vimos no capítulo 3, a linguagem não define precedência entre operadores, e o compilador avalia da esquerda para a direita: em Jack, `2 + 3 * 4` é compilado como `(2 + 3) * 4`.

Este livro, no entanto, **não constrói uma AST persistente**. Como veremos nos capítulos 9 e 10, nosso compilador gera código VM diretamente durante a análise sintática — a árvore existe apenas implicitamente, na pilha de chamadas recursivas do parser. Essa escolha (comum em compiladores de um único passe, *single-pass*) simplifica bastante a implementação, ao custo de tornar mais difíceis otimizações que exigem enxergar o programa inteiro de uma vez — um custo aceitável para os objetivos didáticos deste curso.

## Representações lineares: códigos de N endereços

Uma IR **linear** modela pseudocódigo para uma máquina abstrata — uma escolha natural, já que tanto o código-fonte quanto o código de máquina são, em última instância, sequências lineares de instruções. Cooper classifica essas representações pelo número de operandos que cada instrução referencia:

**Código de três endereços.** Cada instrução tem, no máximo, três operandos: dois de entrada e um de saída (`x = y op z`). Ficou popular com a ascensão das arquiteturas RISC nos anos 80–90, que também favorecem esse padrão "dois operandos, um resultado". Expressões complexas são decompostas em uma sequência de instruções elementares, usando variáveis temporárias:

```
// while (i <= n) { sum = sum + i; i = i + 1; }
L1: if i > n goto L2
    t1 = sum + i
    sum = t1
    t2 = i + 1
    i = t2
    goto L1
L2: ...
```

**Código de dois endereços.** Modela máquinas com operações *destrutivas* (o resultado sobrescreve um dos operandos: `x = x op y`). Caiu em desuso à medida que restrições de memória se tornaram menos críticas — um código de três endereços consegue expressar a mesma coisa sem perder informação.

**Código de um endereço — máquinas de pilha.** Modela o comportamento de máquinas acumuladoras ou de pilha, onde os operandos são implícitos (o topo da pilha). Um código de pilha é extremamente compacto, porque a pilha elimina a necessidade de nomear temporários:

```
// a - 2 + b
push a
push 2
sub
push b
add
```

O preço dessa compactação é que resultados e argumentos são sempre transitórios — a menos que o código os mova explicitamente para a memória (algo que veremos em detalhe no capítulo 7, com os segmentos `local`, `argument`, etc.).

## Bytecode e máquinas virtuais

Quando uma IR de pilha é compilada para um formato binário compacto — tipicamente um byte por opcode —, o resultado é chamado **bytecode**. Uma **máquina virtual** (VM) é o interpretador de software que executa esse bytecode como se fosse uma máquina real, oferecendo portabilidade (o mesmo bytecode roda em qualquer plataforma que tenha a VM) e um ambiente controlado (gerência de memória, isolamento). Java (JVM), Python, C#/.NET e WebAssembly são todos exemplos de linguagens que compilam para bytecode executado por uma VM baseada em pilha.

```python
def f(a, b):
    return a - 2 * b
```

vira, em CPython, o bytecode:

```
LOAD_FAST 0 (a)
LOAD_CONST 1 (2)
LOAD_FAST 1 (b)
BINARY_MULTIPLY
BINARY_SUBTRACT
RETURN_VALUE
```

(Saída de `dis.dis(f)` até o Python 3.10. A partir do 3.11, as duas operações aparecem como `BINARY_OP`, com o operador indicado ao lado — a estrutura de pilha é a mesma.)

Note a semelhança estrutural com o pseudocódigo de pilha acima — `LOAD_FAST`/`LOAD_CONST` são um `push`, e cada operação binária consome dois valores do topo e empilha o resultado. É exatamente esse modelo — pilha de operandos, um opcode por instrução — que a **VM Hack** do Nand2Tetris adota, e que vamos especificar em detalhe no próximo capítulo.

## A escolha deste livro: VM Hack como IR

Resumindo o espectro: ASTs preservam mais estrutura, mas são mais difíceis de gerar diretamente durante o parsing e mais custosas para interpretar (é preciso percorrer a árvore). Códigos de três endereços são o padrão em compiladores de produção com otimizador (GCC, LLVM), porque casam bem com arquiteturas RISC modernas — mas exigem alocação de registradores/temporários, uma complexidade extra. Código de pilha é o mais simples de **gerar** (a notação pós-fixa cai direto de uma travessia pós-ordem da árvore de derivação, como veremos no capítulo 9) e o mais simples de **interpretar** — ao custo de gerar mais instruções por operação complexa e não ser diretamente otimizável sem reconstrução.

Como este curso não implementa um otimizador — o foco é corretude e arquitetura, não desempenho —, a VM Hack (uma IR de pilha) é a escolha certa: simples de gerar diretamente durante a análise sintática (fechando o capítulo 9), simples de traduzir para Assembly Hack depois (capítulos 12 e 13), e didaticamente transparente em cada etapa.

## O que vem a seguir

O próximo capítulo especifica a VM Hack por completo — seus 8 segmentos de memória e todos os seus comandos. É essa especificação que define o *alvo* do compilador: só depois de conhecê-la faz sentido construir a tabela de símbolos (capítulo 8), cuja função é justamente dizer em que segmento e em que posição cada variável de Jack vai morar.
