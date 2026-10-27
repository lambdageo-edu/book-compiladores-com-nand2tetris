# 8. Tabela de símbolos

O capítulo 7 definiu o alvo: cada variável de Jack mora em um segmento da VM (`local`, `argument`, `static` ou `this`), em uma posição dada por um índice. Para gerar um único `push` ou `pop`, portanto, o compilador precisa responder a uma pergunta toda vez que encontra um identificador: *o que é isso?* Uma variável local? Um campo do objeto? Um parâmetro? De que tipo? Em que posição, dentro do seu segmento, ela vive? Este capítulo constrói a estrutura de dados que responde a essas perguntas — a **tabela de símbolos** — e a integra ao `CompilationEngine` da Parte 1, fechando a lacuna de análise semântica que faltava para começar a gerar código de verdade nos capítulos 9 e 10.

## Por que uma tabela de símbolos

Um programa de alto nível introduz e manipula muitos identificadores. Toda vez que o compilador encontra um identificador `xxx`, ele precisa saber: é um nome de variável, de classe, de função? Se é variável, é campo de um objeto ou argumento de uma subroutine? Qual seu tipo — `int`, `boolean`, `char`, ou uma classe? Essas perguntas precisam ser respondidas **toda vez** que `xxx` aparece no código-fonte — não apenas na primeira. A tabela de símbolos existe exatamente para isso: sempre que um identificador é declarado, o compilador registra sua descrição; sempre que ele é referenciado, o compilador consulta a tabela e obtém tudo que precisa para gerar o código correspondente.

## As quatro categorias de variável em Jack

Jack tem exatamente quatro categorias (`kind`) de variável, cada uma com um tempo de vida e — pelo mapeamento padrão do capítulo 7 — um segmento de memória VM associado:

| Categoria (`kind`) | Onde é declarada | Tempo de vida | Segmento VM |
|---|---|---|---|
| `static` | Início da classe, com `static` | Todo o programa; compartilhada por todas as instâncias | `static` |
| `field` | Início da classe, com `field` | Todo o objeto; uma cópia por instância | `this` |
| `arg` | Lista de parâmetros da subroutine | Duração da chamada | `argument` |
| `var` | Início da subroutine, com `var` | Duração da chamada | `local` |

Como `static`/`field` vivem no nível da **classe** e `arg`/`var` vivem no nível da **subroutine**, o compilador precisa manter, a qualquer momento, exatamente **duas** tabelas de símbolos ativas: uma de classe (que persiste durante toda a compilação de uma classe) e uma de subroutine (que é **limpa** toda vez que uma nova subroutine começa a ser compilada).

## API da tabela de símbolos

A tabela expõe uma API pequena e direta:

| Método | Papel |
|---|---|
| `startSubroutine()` | Limpa a tabela de subroutine (chamado ao entrar em cada `constructor`/`function`/`method`) |
| `define(name, type, kind)` | Registra um novo identificador na tabela apropriada (classe se `kind` for `static`/`field`; subroutine se for `arg`/`var`) |
| `varCount(kind)` | Quantas variáveis de uma dada categoria já foram definidas no escopo atual |
| `resolve(name)` | Busca um identificador, devolvendo categoria, tipo e índice |

O índice devolvido por `resolve` (e atribuído automaticamente por `define`) é sempre relativo à sua própria categoria: a primeira `field` é índice 0, a segunda `field` é índice 1, e uma `static` declarada na mesma classe começa a sua própria contagem em 0. É exatamente esse índice — categoria + posição — que vira o segmento e o deslocamento de um comando `push`/`pop` da VM.

## Um exemplo completo

Considere a classe `Point`, que representa um ponto no plano e conta quantos pontos já foram criados:

```java
class Point {
    field int x, y;
    static int pointCount;

    constructor Point new(int ax, int ay) {
        let x = ax;
        let y = ay;
        let pointCount = pointCount + 1;
        return this;
    }

    method int getx() { return x; }
    method int gety() { return y; }

    method int distance(Point other) {
        var int dx, dy;
        let dx = x - other.getx();
        let dy = y - other.gety();
        return Math.sqrt((dx * dx) + (dy * dy));
    }
}
```

Ao terminar de ler as declarações do início da classe, o compilador tem a **tabela de classe** preenchida. Ela vale para todas as subroutines de `Point`:

| Nome | Tipo | Categoria | Índice | Acesso na VM |
|---|---|---|---|---|
| `x` | `int` | `field` | 0 | `this 0` |
| `y` | `int` | `field` | 1 | `this 1` |
| `pointCount` | `int` | `static` | 0 | `static 0` |

Repare que `pointCount` recebe índice 0, embora seja a terceira variável declarada: `static` e `field` são contadas separadamente.

A **tabela de subroutine**, por sua vez, é recriada a cada subroutine. Ao compilar `distance`, ela fica assim:

| Nome | Tipo | Categoria | Índice | Acesso na VM |
|---|---|---|---|---|
| `this` | `Point` | `arg` | 0 | `argument 0` |
| `other` | `Point` | `arg` | 1 | `argument 1` |
| `dx` | `int` | `var` | 0 | `local 0` |
| `dy` | `int` | `var` | 1 | `local 1` |

A entrada `this` não aparece no código-fonte: o compilador a insere porque `distance` é um `method`, que recebe o objeto como argumento implícito (veremos como, adiante neste capítulo). Por isso o primeiro parâmetro declarado, `other`, fica com índice 1. Compare com a tabela do construtor `new`, compilado antes de `distance` com a tabela de subroutine vazia:

| Nome | Tipo | Categoria | Índice | Acesso na VM |
|---|---|---|---|---|
| `ax` | `int` | `arg` | 0 | `argument 0` |
| `ay` | `int` | `arg` | 1 | `argument 1` |

Um construtor não recebe `this` como argumento — é ele quem cria o objeto —, então `ax` começa em 0.

Com as duas tabelas, o comando `let dx = x - other.getx();` de `distance` pode ser traduzido: a busca encontra `x` na tabela de classe (`this 0`), e `other` e `dx` na tabela de subroutine (`argument 1` e `local 0`):

```
push this 0           // x
push argument 1       // other (o objeto sobre o qual getx opera)
call Point.getx 1
sub
pop local 0           // dx
```

Da mesma forma, `let pointCount = pointCount + 1;` no construtor vira `push static 0`, `push constant 1`, `add`, `pop static 0`.

!!! note "Comparação com a API do Nand2Tetris"
    O livro do Nand2Tetris propõe uma API com três consultas separadas — `kindOf(name)`, `typeOf(name)` e `indexOf(name)`. Aqui elas foram reunidas em `resolve(name)`, que devolve as três informações de uma vez em um `Symbol` (ou `null`, o equivalente ao `NONE` de `kindOf`). `startSubroutine`, `define` e `varCount` têm o mesmo papel nas duas versões.

```java
public class SymbolTable {
    public enum Kind { STATIC, FIELD, ARG, VAR }

    public record Symbol(String name, String type, Kind kind, int index) {}

    private Map<String, Symbol> classScope = new HashMap<>();
    private Map<String, Symbol> subroutineScope = new HashMap<>();
    private Map<Kind, Integer> counts = new HashMap<>();

    public SymbolTable() {
        for (Kind k : Kind.values()) counts.put(k, 0);
    }

    public void startSubroutine() {
        subroutineScope.clear();
        counts.put(Kind.ARG, 0);
        counts.put(Kind.VAR, 0);
    }

    private Map<String, Symbol> scopeFor(Kind kind) {
        return (kind == Kind.STATIC || kind == Kind.FIELD) ? classScope : subroutineScope;
    }

    public void define(String name, String type, Kind kind) {
        var scope = scopeFor(kind);
        int index = varCount(kind);
        scope.put(name, new Symbol(name, type, kind, index));
        counts.put(kind, index + 1);
    }

    public int varCount(Kind kind) {
        return counts.get(kind);
    }

    public Symbol resolve(String name) {
        Symbol s = subroutineScope.get(name);
        return s != null ? s : classScope.get(name);
    }
}
```

Repare em `resolve`: a busca acontece **primeiro** no escopo de subroutine e só depois no de classe — exatamente a regra de sombreamento (*shadowing*) que se espera de escopos aninhados: uma variável local com o mesmo nome de um campo da classe deve "ganhar" dentro daquela subroutine.

## Integrando ao parser

A tabela se popula em exatamente três pontos da gramática — todos já reconhecidos pelo `CompilationEngine` da Parte 1, agora ganhando uma linha de `symbolTable.define(...)` cada:

**Variáveis de classe** (`classVarDec`, dentro de `parseClass`):

```java
void parseClassVarDec() {
    expectPeek(FIELD, STATIC);
    Kind kind = currentTokenIs(FIELD) ? Kind.FIELD : Kind.STATIC;

    expectPeek(INT, CHAR, BOOLEAN, IDENT);
    String type = currentToken.lexeme;

    expectPeek(IDENT);
    symbolTable.define(currentToken.lexeme, type, kind);
    while (peekTokenIs(COMMA)) {
        expectPeek(COMMA);
        expectPeek(IDENT);
        symbolTable.define(currentToken.lexeme, type, kind);
    }
    expectPeek(SEMICOLON);
}
```

**Parâmetros** (`parameterList`, dentro de `parseSubroutineDec`) — sempre `Kind.ARG`:

```java
void parseParameterList() {
    if (!peekTokenIs(RPAREN)) {
        expectPeek(INT, CHAR, BOOLEAN, IDENT);
        String type = currentToken.lexeme;
        expectPeek(IDENT);
        symbolTable.define(currentToken.lexeme, type, Kind.ARG);
        while (peekTokenIs(COMMA)) {
            expectPeek(COMMA);
            expectPeek(INT, CHAR, BOOLEAN, IDENT);
            type = currentToken.lexeme;
            expectPeek(IDENT);
            symbolTable.define(currentToken.lexeme, type, Kind.ARG);
        }
    }
}
```

**Variáveis locais** (`varDec`, dentro de `parseSubroutineBody`) — sempre `Kind.VAR`, seguindo exatamente o mesmo padrão de `classVarDec`, trocando apenas a categoria.

E, no início de cada `subroutineDec`, dois ajustes indispensáveis:

```java
void parseSubroutineDec() {
    symbolTable.startSubroutine();

    expectPeek(CONSTRUCTOR, FUNCTION, METHOD);
    TokenType subroutineType = currentToken.type;

    if (subroutineType == METHOD) {
        symbolTable.define("this", className, Kind.ARG);
    }
    // ...
}
```

O truque de `if (subroutineType == METHOD) symbolTable.define("this", className, Kind.ARG)` merece destaque: pelo mapeamento padrão (capítulo 7), um método é sempre chamado com o objeto receptor como **primeiro argumento implícito** (o compilador insere isso na chamada, como veremos no capítulo 10) — logo, registrá-lo como `arg` de índice 0 *antes* de processar a lista de parâmetros declarados garante que o primeiro parâmetro *de fato* declarado pelo programador já saia com índice 1, exatamente onde ele estará em tempo de execução.

## Testando a tabela antes de gerar código

Um erro na tabela de símbolos (um índice que não zera em `startSubroutine`, um `this` esquecido) só aparece, nos capítulos seguintes, como código VM sutilmente errado — e é difícil rastreá-lo de volta. Vale, por isso, testar a tabela **sozinha**, antes de gerar uma única linha de VM, aproveitando a saída XML que o parser da Parte 1 já produz: basta enriquecer cada `<identifier>` com o que a tabela sabe sobre ele.

Para cada identificador, imprima a **categoria** (`var`, `argument`, `static`, `field`, ou `class`/`subroutine` para os nomes que não estão na tabela), o **índice** (para as quatro categorias de variável) e se aquele ponto do código é uma **declaração** ou um **uso**. Para a classe `Point` do exemplo acima, um trecho da saída ficaria assim:

```xml
<keyword> field </keyword>
<keyword> int </keyword>
<identifier category="field" index="0" usage="declared"> x </identifier>
<symbol> , </symbol>
<identifier category="field" index="1" usage="declared"> y </identifier>
...
<keyword> let </keyword>
<identifier category="field" index="0" usage="used"> x </identifier>
<symbol> = </symbol>
<expression>
  <term>
    <identifier category="argument" index="0" usage="used"> ax </identifier>
  </term>
</expression>
```

Não há arquivo de referência para comparar (essa saída não faz parte do Nand2Tetris), mas a conferência é visual e rápida: rode o parser sobre `Point.jack` e confira, linha a linha, se cada índice bate com a tabela montada à mão na seção "Um exemplo completo". Quando bater, a tabela está pronta — e os atributos extras podem ser removidos, ou deixados atrás de uma opção de depuração.

## O que a tabela ainda não faz

Esta tabela responde "que categoria, que tipo, que índice" — o suficiente para gerar `push`/`pop` corretos nos capítulos 9 e 10. Ela **não** faz verificação de tipos (não impede que você some um `int` com um `boolean`), nem detecta redeclaração além do básico, nem resolve sobrecarga (que nem existe em Jack). Esse é o escopo deliberadamente mínimo de análise semântica deste curso — o suficiente para gerar código correto, sem se tornar um verificador de tipos completo.

## O que vem a seguir

Com a tabela de símbolos pronta, temos as três peças: a estrutura do programa (o parser da Parte 1), o alvo (a VM Hack do capítulo 7) e a ponte entre os nomes de Jack e os segmentos da VM. Os capítulos 9 e 10 juntam as três e começam a gerar código de verdade, agora que sabemos exatamente onde cada identificador vive.
