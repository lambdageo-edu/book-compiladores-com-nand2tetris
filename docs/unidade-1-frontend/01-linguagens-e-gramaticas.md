# 1. Linguagens e gramáticas

Antes de escrever qualquer código, precisamos de uma linguagem precisa para descrever... linguagens. Este capítulo estabelece o ferramental formal — expressões regulares, autômatos finitos e gramáticas livres de contexto — que vamos usar em todo o resto do livro. Ele não constrói nenhum código ainda, mas toda decisão de projeto dos capítulos 4 e 5 (o analisador léxico e o analisador sintático de Jack) parte diretamente do que veremos aqui.

## Linguagens regulares

A análise léxica — a etapa que agrupa caracteres em tokens — trabalha inteiramente no nível das **linguagens regulares**: conjuntos de cadeias de caracteres simples o suficiente para serem reconhecidos por uma máquina com memória finita, sem necessidade de pilha ou contagem.

### Autômato finito

Formalmente, um autômato finito é descrito por uma quíntupla M = (S, Σ, δ, s0, SA):

1. **S** — o conjunto finito de estados;
2. **Σ** — o alfabeto de entrada;
3. **δ** — o conjunto de transições, cada uma uma tripla (sᵢ, Σₜ, sf): do estado sᵢ, ao ler um símbolo em Σₜ, o autômato passa ao estado sf;
4. **s0** — o estado inicial;
5. **SA** — o conjunto de estados finais (de aceitação).

Um autômato é **determinístico** quando, de cada estado, não existem duas transições saindo com o mesmo símbolo — ou seja, o próximo estado é sempre unicamente determinado pelo estado atual e pelo símbolo lido. Essa propriedade é o que torna um autômato finito determinístico (AFD) direto de traduzir em código, como veremos adiante.

### Expressões regulares

Para qualquer autômato finito, podemos descrever a linguagem que ele reconhece através de uma notação textual compacta: a **expressão regular** (RE). Formalmente, o conjunto de expressões regulares sobre um alfabeto Σ é definido indutivamente:

1. Se `a ∈ Σ`, então `a` é uma RE que indica o conjunto `{a}`.
2. Se `r` e `s` são REs indicando os conjuntos `L(r)` e `L(s)`, então:
   - `r | s` é a **alternação** (união) de `L(r)` e `L(s)`;
   - `rs` é a **concatenação** de `L(r)` e `L(s)`;
   - `r*` é o **fechamento de Kleene** de `L(r)` (zero ou mais repetições).
3. `ε` é uma RE indicando o conjunto contendo apenas a cadeia vazia.

Por convenção de precedência (da mais alta para a mais baixa): parênteses, fechamento (`*`), concatenação, alternação (`|`). Como abreviação, intervalos de caracteres são escritos entre colchetes com reticências — `[0...9]` significa `(0|1|2|...|9)`.

**Exemplo — identificadores estilo Algol** (uma letra seguida de zero ou mais letras, dígitos ou `_`):

```
([A...Z] | [a...z]) ([A...Z] | [a...z] | [0...9] | _)*
```

**Exemplo — inteiro sem sinal** (zero, ou um dígito não-zero seguido de dígitos):

```
0 | [1...9] [0...9]*
```

Toda expressão regular tem um autômato finito equivalente que a reconhece — essa equivalência (Kleene) é o que garante que qualquer coisa que possamos descrever com REs também podemos implementar como uma máquina de estados.

## Reconhecimento de tokens: de diagramas a código

Um analisador léxico real não trabalha com a definição formal de autômato — ele trabalha com **diagramas de transição**, uma representação gráfica direta das REs de cada tipo de token, e com uma receita mecânica para transformar esses diagramas em código.

Considere o reconhecimento de operadores relacionais (`<`, `<=`, `<>`, `=`, `>`, `>=`) de uma linguagem estilo Pascal:

```
lexemas: <, <=, <>, =, >, >=          token: relop           atributo: LT, LE, NE, EQ, GT, GE
```

O diagrama de transição correspondente tem um estado inicial, estados intermediários que consomem `<` ou `>` e aguardam um possível `=` a seguir, e estados finais que devolvem o token com o atributo apropriado. Um detalhe importante: ao ler um caractere que não faz parte do lexema atual (por exemplo, ler `x` depois de `<`), o autômato precisa **recuar uma posição** na entrada antes de aceitar — esse caractere pertence ao próximo token, não ao atual.

A receita para transformar um diagrama de transição em código é sempre a mesma:

- uma variável guarda o **estado corrente**;
- um `switch` seleciona o comportamento com base nesse estado;
- cada `case` do switch lê o próximo caractere e decide para qual estado transitar em seguida (ou se deve aceitar e retornar um token).

```java
Token GetRelop() {
    Token t = new Token(RELOP);
    while (true) {
        switch (state) {
            case 0:
                c = GetChar();
                if (c == '<') state = 1;
                else if (c == '=') state = 5;
                else if (c == '>') state = 6;
                else fail();
                break;
            // ... demais estados ...
            case 8:
                UngetChar();
                t.attribute = GT;
                return t;
        }
    }
}
```

O que `fail()` faz depende da estratégia de recuperação de erro adotada: tipicamente, ele deve devolver o apontador de entrada ao início do lexema não reconhecido e permitir que **outro** diagrama de transição seja tentado (afinal, o analisador léxico precisa lidar com todos os tipos de token, não só um).

### Identificadores, palavras-chave e números

Identificadores e palavras-chave compartilham exatamente o mesmo padrão léxico (uma letra seguida de letras/dígitos) — a diferença entre eles não é léxica, é apenas uma questão de **tabela**: qualquer lexema reconhecido pelo diagrama de identificador que já esteja pré-cadastrado como palavra-chave (`if`, `while`, `class`, ...) é reclassificado como o token daquela palavra-chave; o restante é um identificador de fato. Essa é a abordagem que vamos usar no capítulo 4 para o `JackTokenizer`.

Números seguem um padrão semelhante, mas com mais estados (parte inteira, ponto decimal opcional, expoente opcional):

```
num → dígitos ('.' dígitos)? ('E' [+-]? dígitos)?
```

Espaços em branco (`delim → (branco|tab|newline)+`) merecem um caso especial: o token correspondente normalmente **não é devolvido** ao analisador sintático — ele apenas avança o analisador léxico até o próximo caractere significativo.

### Combinando diagramas

Um analisador léxico completo precisa lidar com todos os tipos de token ao mesmo tempo. Há três estratégias possíveis:

1. Testar os diagramas de cada token **sequencialmente** (permite dar prioridade natural às palavras-chave sobre identificadores).
2. Rodar todos os diagramas **em paralelo** e escolher a cadeia reconhecida mais longa — a estratégia conhecida como ***maximal munch*** (o "próximo" token é sempre o prefixo válido mais longo possível).
3. **Combinar** todos os diagramas em uma única máquina de estados, unificando os estados iniciais.

Na prática, um tokenizador manual (como o que construiremos no capítulo 4) tipicamente usa uma versão simplificada da estratégia (1): olha o primeiro caractere para decidir a *categoria* do próximo token (dígito → número; letra → identificador/palavra-chave; aspas → string; símbolo → símbolo) e então aplica maximal munch dentro daquela categoria quando necessário (por exemplo, para não parar em `<` quando o lexema real é `<=`).

## Linguagens livres de contexto

Expressões regulares e autômatos finitos não têm memória — eles não conseguem "contar" nem lidar com aninhamento arbitrário (parênteses balanceados, blocos aninhados). As construções de uma linguagem de programação são inerentemente recursivas (`cmd → if expr then cmd else cmd`), e para descrevê-las precisamos de um formalismo mais poderoso: a **gramática livre de contexto** (GLC).

Formalmente, uma GLC é uma quádrupla G = (T, NT, S, P):

- **T** — símbolos terminais (as categorias sintáticas devolvidas pelo analisador léxico: os tokens);
- **NT** — símbolos não-terminais (variáveis sintáticas que dão estrutura às produções);
- **S** — o não-terminal inicial (será `class`, na gramática de Jack);
- **P** — as produções, cada uma da forma `NT → (T ∪ NT)*`.

### Notação BNF

A forma mais comum de escrever uma GLC é a **Backus-Naur Form** (BNF): `::=` (ou `→`) para produção, `<...>` para não-terminais, `|` para alternativas. Diagramas sintáticos são a contraparte gráfica: retângulos para não-terminais, círculos/elipses para terminais.

### Derivação e ambiguidade

Uma derivação reescreve, passo a passo, um não-terminal pelo corpo de uma de suas produções — uma construção *top-down*. Por exemplo, com a gramática:

```
Expr → ( Expr ) | Expr Op nome | nome
Op   → + | - | x | ÷
```

a cadeia `(a + b) x c` pode ser derivada tanto pela substituição sempre do não-terminal **mais à direita** (derivação mais à direita) quanto sempre do **mais à esquerda** (derivação mais à esquerda). Quando a gramática é bem comportada, ambas produzem a mesma árvore de derivação.

Uma gramática é **ambígua** quando alguma cadeia admite mais de uma derivação mais à esquerda (ou mais à direita) distinta — ou seja, mais de uma árvore de derivação válida. Um exemplo clássico: `E → E+E | E*E | (E) | a` é ambígua para `a + a * a` (o `*` pode ser agrupado com o segundo `a` de duas formas diferentes, uma delas violando a precedência esperada). Não existe um procedimento geral para detectar ou eliminar ambiguidade — cada caso exige uma reescrita específica da gramática, tipicamente introduzindo não-terminais que codificam a precedência desejada (é o que faz a gramática de expressões com `expr`, `term` e `fact` que aparece logo adiante, na seção sobre recursão à esquerda).

### Gramáticas LL e análise preditiva

Este livro constrói, no capítulo 5, um analisador sintático **descendente recursivo** — um método *preditivo*, que decide qual produção aplicar olhando apenas o próximo token (nenhum *backtracking*). Um parser descendente comum, na ausência de uma gramática bem comportada, pode precisar de tentativa e erro: experimenta uma produção e, se ela falhar mais adiante, recua e tenta outra. Esse recuo (*backtracking*) é caro e indesejável — e é exatamente o que um analisador **preditivo** evita, ao garantir que o próximo token, sozinho, já determina sem ambiguidade qual produção usar.

Gramáticas que permitem essa escolha unívoca são chamadas **LL(1)** — a leitura é feita da esquerda para a direita (*Left-to-right*), construindo a derivação mais à esquerda (*Leftmost*), com exatamente 1 símbolo de antecipação (*lookahead*). Formalmente, seja **FIRST(α)** o conjunto de símbolos terminais que podem aparecer no início de alguma cadeia derivada de α:

1. se α começa com um terminal, esse terminal está em FIRST(α);
2. se α começa com um não-terminal, todos os terminais que iniciam alguma produção desse não-terminal entram em FIRST(α);
3. se α pode derivar a cadeia vazia (ε), então ε também está em FIRST(α).

Para um não-terminal com produções alternativas `A → α | β`, o parser preditivo só funciona se **FIRST(α) e FIRST(β) forem disjuntos**: ao ver o token de lookahead, o parser escolhe α se o token estiver em FIRST(α), ou β se estiver em FIRST(β) — sem qualquer ambiguidade. Essa é a formalização exata do que faremos, de modo informal, nos métodos `if`/`else if` dos capítulos 2, 4 e 5: cada `case` de um `switch(lookahead)` (ou cada ramo de um `if`/`else if`) corresponde a uma produção cujo FIRST contém aquele token:

```c
void inst() {
    switch (lookahead) {
        case EXPR: match(EXPR); match(';'); break;
        case IF:   match(IF); match('('); match(EXPR); match(')'); inst(); break;
        case FOR:  match(FOR); match('('); optexpr(); match(';');
                   optexpr(); match(';'); optexpr(); match(')'); inst(); break;
        default:   error("syntax error");
    }
}
```

Gramáticas que precisam satisfazer as duas condições LL(1) — sem recursão à esquerda, com conjuntos FIRST disjuntos entre alternativas — são chamadas **gramáticas LL**, e o método que as reconhece, **análise preditiva**. Duas transformações mecânicas resolvem a maior parte dos casos que violam essas condições:

**Eliminação de recursão à esquerda imediata.** Uma gramática tem recursão à esquerda imediata quando existe uma produção `A → Aα`. Ela pode sempre ser reescrita:

```
A → Aα | β        vira        A → βR
                               R → αR | ε
```

Aplicando ao exemplo clássico de expressões aritméticas:

```
expr → expr + term | term          expr → term plus
term → term * fact | fact          plus → + term plus | ε
fact → (expr) | id                 term → fact mult
                                    mult → * fact mult | ε
                                    fact → (expr) | id
```

Quando a recursão não é imediata (por exemplo, `S → Aa | b` e `A → Ac | Sd | ε`, em que `S` é recursivo à esquerda de forma indireta via `A`), um algoritmo sistemático resolve o caso geral: ordene os não-terminais `A₁, ..., Aₙ`; para cada `Aᵢ`, substitua toda produção `Aᵢ → Aⱼγ` (com `j < i`) pelas produções de `Aⱼ` expandidas; em seguida elimine a recursão imediata resultante em `Aᵢ`.

**Fatoração à esquerda.** Um analisador preditivo não pode escolher entre duas produções que começam com o mesmo símbolo:

```
inst → if expr then inst | if expr then inst else inst | other
```

A fatoração adia a decisão, introduzindo um não-terminal auxiliar:

```
A → αβ1 | αβ2        vira        A → αR
                                  R → β1 | β2

inst → if expr then inst opt
opt  → else inst | ε
```

Essas duas transformações — eliminação de recursão à esquerda e fatoração à esquerda — são exatamente o que torna possível transformar uma gramática em um analisador descendente recursivo direto, um método por não-terminal. É a técnica que aplicaremos, na prática, ao longo dos capítulos 2 e 5.

### Tradução dirigida por sintaxe

Um analisador preditivo reconhece — mas, sozinho, não *traduz*. Um **esquema de tradução dirigido por sintaxe** estende a gramática associando uma **ação semântica** a cada produção, escrita entre chaves na posição em que deve ser executada:

```
Gramática                          Esquema de tradução

expr  → expr + digit                expr  → expr + digit { print('+') }
      | expr - digit                      | expr - digit { print('-') }
      | digit                             | digit
digit → 0 | ... | 9                 digit → 0 { print('0') } | ... | 9 { print('9') }
```

A regra de ouro é posicional: se a ação aparece depois do símbolo `X` na produção, ela deve ser copiada para o método gerado logo **depois** da chamada/casamento correspondente a `X`. Construir um tradutor dirigido por sintaxe é, portanto, apenas um passo além de construir um analisador preditivo: constrói-se o parser preditivo normalmente, e depois se copiam as ações semânticas para dentro de cada método, respeitando essa posição relativa. É exatamente esse o nome — e o método — que dá título a esta disciplina, e é o que faremos concretamente já no capítulo 2 (onde as "ações" são `System.out.println("push ...")`, `println("add")` etc.) e, em escala completa, nos capítulos 9 e 10 da Parte 2.

### Os limites de uma gramática

Gramáticas livres de contexto descrevem a maior parte, mas não toda a sintaxe de uma linguagem de programação: regras como "todo identificador deve ser declarado antes do uso" ou "o número de argumentos de uma chamada deve bater com a declaração da função" não são expressáveis em uma GLC. As cadeias de tokens aceitas pelo analisador sintático formam, na verdade, um **superconjunto** da linguagem — cabe à análise semântica (capítulo 8) impor as regras que restam.

## O que vem a seguir

Com REs, autômatos, GLCs e as duas transformações de gramática (eliminação de recursão à esquerda e fatoração à esquerda) em mãos, o próximo capítulo usa exatamente esse ferramental para construir, de forma incremental, um pequeno tradutor de expressões aritméticas — o aquecimento antes de atacarmos a linguagem Jack por completo.
