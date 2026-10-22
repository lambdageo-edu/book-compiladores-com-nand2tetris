# 10. Geração de código: subrotinas e objetos

Este capítulo fecha o compilador Jack → VM: variáveis (usando a tabela de símbolos do capítulo 8), a declaração completa de `function`/`method`/`constructor` (com o prólogo que aloca variáveis locais e, para construtores, memória de objeto), chamadas de subroutine, e o mapeamento de arrays via aritmética de ponteiros. Ao final, o **Project 11** do Nand2Tetris estará completo: qualquer programa Jack válido poderá ser compilado para VM.

## De categoria para segmento

A tabela de símbolos (capítulo 8) guarda a *categoria* de uma variável (`STATIC`, `FIELD`, `ARG`, `VAR`); a VM (capítulo 7) trabalha com *segmentos* (`static`, `this`, `argument`, `local`). Uma função pura faz essa ponte — e é usada em todo lugar que o compilador precisa gerar `push`/`pop` para uma variável:

```java
private Segment kind2Segment(Kind kind) {
    return switch (kind) {
        case STATIC -> Segment.STATIC;
        case FIELD  -> Segment.THIS;   // campos vivem no segmento `this`
        case VAR    -> Segment.LOCAL;
        case ARG    -> Segment.ARG;
    };
}
```

O único mapeamento não óbvio é `FIELD → THIS`: campos de um objeto são acessados através do segmento `this`, que aponta (via `pointer 0`) para o objeto correntemente "ativo" — voltaremos a isso na seção de métodos.

## Acessando variáveis em expressões

Quando `parseTerm` encontra um identificador que não é seguido de `(` nem `.` (ou seja, não é uma chamada de subroutine — distinção que revisitamos adiante), ele é uma variável simples: basta resolver na tabela de símbolos e emitir o `push` correspondente.

```java
case IDENT:
    expectPeek(IDENT);
    Symbol sym = symbolTable.resolve(currentToken.lexeme);
    if (peekTokenIs(LPAREN) || peekTokenIs(DOT)) {
        parseSubroutineCall();
    } else if (peekTokenIs(LBRACKET)) {
        // acesso a array — ver seção dedicada abaixo
    } else {
        vmWriter.writePush(kind2Segment(sym.kind()), sym.index());
    }
    break;
```

Do lado esquerdo de uma atribuição (`let x = ...`), o mesmo símbolo resolve para um `pop` em vez de um `push` — calcule primeiro o valor da direita, desempilhe-o para a variável depois:

```java
void parseLet() {
    expectPeek(LET);
    expectPeek(IDENT);
    Symbol symbol = symbolTable.resolve(currentToken.lexeme);
    boolean isArray = false;

    if (peekTokenIs(LBRACKET)) {
        // ver seção de arrays
        isArray = true;
    }

    expectPeek(EQ);
    parseExpression();

    if (!isArray) {
        vmWriter.writePop(kind2Segment(symbol.kind()), symbol.index());
    }
    expectPeek(SEMICOLON);
}
```

## Declarando funções

O nome de uma função VM é sempre `Classe.nomeDaSubroutine` — então o `Parser` precisa lembrar o nome da classe corrente (capturado uma única vez, em `parseClass`):

```java
void parseClass() {
    expectPeek(CLASS);
    expectPeek(IDENT);
    className = currentToken.lexeme;
    // ...
}
```

O comando `function` só pode ser emitido depois que **todas** as variáveis locais já foram registradas na tabela de símbolos (capítulo 8) — porque `nLocais` é exatamente `symbolTable.varCount(Kind.VAR)`. Por isso a emissão acontece no corpo da subroutine, não na sua declaração. `parseSubroutineDec` reúne tudo o que o corpo vai precisar — o nome completo da função e o tipo de subroutine — e passa adiante:

```java
void parseSubroutineDec() {
    symbolTable.startSubroutine();       // capítulo 8
    ifLabelNum = 0;                      // capítulo 9
    whileLabelNum = 0;

    expectPeek(CONSTRUCTOR, FUNCTION, METHOD);
    TokenType subroutineType = currentToken.type;
    if (subroutineType == METHOD) {
        symbolTable.define("this", className, Kind.ARG);
    }

    expectPeek(VOID, INT, CHAR, BOOLEAN, IDENT);   // tipo de retorno
    expectPeek(IDENT);
    String functionName = className + "." + currentToken.lexeme;

    expectPeek(LPAREN);
    parseParameterList();
    expectPeek(RPAREN);

    parseSubroutineBody(functionName, subroutineType);
}

void parseSubroutineBody(String functionName, TokenType subroutineType) {
    expectPeek(LBRACE);
    while (peekTokenIs(VAR)) parseVarDec();

    vmWriter.writeFunction(functionName, symbolTable.varCount(Kind.VAR));
    // prólogo de construtor ou de método — ver as duas próximas seções

    parseStatements();
    expectPeek(RBRACE);
}
```

Com isso, uma função simples já compila de ponta a ponta:

```java
@Test
void testSimpleFunctions() {
    var input = """
        class Main {
            function int soma(int x, int y) { return 30; }
            function void main() { var int d; return; }
        }
        """;
    var parser = new Parser(input.getBytes());
    parser.parse();
    assertEquals("""
        function Main.soma 0
        push constant 30
        return
        function Main.main 1
        push constant 0
        return
        """, parser.VMOutput());
}
```

## Construtores: alocando o objeto

Um `constructor` precisa, antes de mais nada, alocar espaço no heap para o novo objeto — e apontar `this` (via `pointer 0`) para esse espaço recém-alocado. O tamanho a alocar é exatamente o número de `field`s da classe, já conhecido pela tabela de símbolos. Este trecho entra em `parseSubroutineBody`, no lugar do comentário de prólogo, logo depois do `writeFunction` — a alocação precisa ser a primeira coisa que a função faz:

```java
if (subroutineType == CONSTRUCTOR) {
    vmWriter.writePush(Segment.CONST, symbolTable.varCount(Kind.FIELD));
    vmWriter.writeCall("Memory.alloc", 1);
    vmWriter.writePop(Segment.POINTER, 0);   // this = endereço retornado por Memory.alloc
}
```

A partir daí, o restante do construtor é só comandos normais — `let x = ax;` vira `push argument 0` seguido de `pop this 0`, exatamente como qualquer atribuição a um campo. O `return this;` no final de um construtor Jack é apenas `push pointer 0` seguido de `return` — devolver o endereço do objeto que acabou de ser inicializado.

```
constructor Point new(int ax, int ay) {
    let x = ax; let y = ay; return this;
}
```

compila para:

```
function Point.new 0
push constant 2          // 2 campos: x e y
call Memory.alloc 1
pop pointer 0             // this = endereço alocado
push argument 0
pop this 0                // x = ax
push argument 1
pop this 1                // y = ay
push pointer 0
return                    // return this
```

## Métodos: recebendo `this`

Um método Jack (`p.getX()`) é, na VM, uma função comum que recebe o objeto receptor como **primeiro argumento**. O capítulo 8 já garantiu que, ao compilar um método, `"this"` foi registrado como `arg` de índice 0 — falta apenas, no prólogo do método (o mesmo ponto de `parseSubroutineBody`, logo depois do `writeFunction`), **materializar** esse argumento no registrador `pointer 0`, para que o restante do corpo possa usar `this 0`, `this 1` etc. normalmente:

```java
if (subroutineType == METHOD) {
    vmWriter.writePush(Segment.ARG, 0);
    vmWriter.writePop(Segment.POINTER, 0);  // this = argument 0
}
```

Depois dessa única linha extra no prólogo, o corpo de um método é indistinguível, em termos de geração de código, do corpo de qualquer outra função.

## Chamadas: distinguindo função, método e método de outro objeto

A parte mais delicada da geração de código de chamadas é decidir, a partir de um identificador seguido de `(` ou de `.`, qual das três formas se aplica — e a tabela de símbolos é exatamente a ferramenta que resolve essa ambiguidade:

```java
void parseSubroutineCall() {
    String ident = currentToken.lexeme;
    Symbol symbol = symbolTable.resolve(ident);
    int nArgs = 0;
    String functionName;

    if (peekTokenIs(LPAREN)) {
        // método da própria classe: chamado sem qualificador -> this é implícito
        expectPeek(LPAREN);
        vmWriter.writePush(Segment.POINTER, 0);
        nArgs = parseExpressionList() + 1;
        expectPeek(RPAREN);
        functionName = className + "." + ident;
    } else {
        expectPeek(DOT);
        expectPeek(IDENT);
        if (symbol != null) {
            // "ident" é uma VARIÁVEL -> é um método de outro objeto
            functionName = symbol.type() + "." + currentToken.lexeme;
            vmWriter.writePush(kind2Segment(symbol.kind()), symbol.index());
            nArgs = 1;  // o próprio objeto, como primeiro argumento
        } else {
            // "ident" não está na tabela -> é o nome de uma CLASSE -> função
            functionName = ident + "." + currentToken.lexeme;
        }
        expectPeek(LPAREN);
        nArgs += parseExpressionList();
        expectPeek(RPAREN);
    }
    vmWriter.writeCall(functionName, nArgs);
}
```

A regra de ouro está no `if (symbol != null)`: se o identificador antes do `.` resolve para algo na tabela de símbolos, ele é uma **variável** (logo, `variavel.metodo(...)` é uma chamada de método sobre outro objeto, e esse objeto precisa ser empilhado como argumento extra); se não resolve, é o nome de uma **classe**, e `Classe.funcao(...)` é uma chamada de função comum, sem receptor implícito.

```
let d = p1.distance(p2);
```

com `p1` resolvido como uma variável do tipo `Point`, compila para:

```
push local 0        // empilha p1 (o receptor)
push local 1        // empilha p2 (o argumento explícito)
call Point.distance 2
pop local 2
```

### O comando `do` e o valor descartado

Pelo mapeamento padrão (capítulo 7), toda função VM devolve um valor — inclusive as que correspondem a subroutines `void`, que devolvem `0`. Uma chamada usada como `term` de uma expressão aproveita esse valor; já o comando `do` chama a subroutine só pelo efeito, e o valor que sobra no topo da pilha precisa ser descartado. Por isso `parseDo` emite um `pop temp 0` logo depois da chamada:

```java
void parseDo() {
    expectPeek(DO);
    expectPeek(IDENT);
    parseSubroutineCall();
    expectPeek(SEMICOLON);
    vmWriter.writePop(Segment.TEMP, 0);   // descarta o valor devolvido
}
```

O programa `Seven` do Project 11 mostra os dois lados desse contrato em poucas linhas:

```
// do Output.printInt(1 + (2 * 3));  return;
function Main.main 0
push constant 1
push constant 2
push constant 3
call Math.multiply 2
add
call Output.printInt 1
pop temp 0            // quem chama: descarta o valor devolvido
push constant 0       // quem é chamado (main é void): devolve 0
return
```

## Arrays: aritmética de ponteiros

Um array Jack é, para o compilador, apenas um ponteiro para o endereço-base de um bloco contíguo no heap — `arr[i]` é açúcar sintático para "o valor no endereço `arr + i`". A VM não tem um comando de "acesso indexado" dedicado; ela oferece **endereçamento indireto** através do par `pointer 1` / segmento `that`: gravar um endereço em `pointer 1` faz `that 0` passar a se referir àquela célula de memória.

Uma primeira tentativa ingênua — calcular o endereço, apontar `that` para lá, e então avaliar o lado direito — quebra em casos como `let a[i] = b[j]`, porque avaliar `b[j]` (o lado direito) também precisa usar `pointer 1`/`that`, sobrescrevendo o endereço que tínhamos acabado de calcular para `a[i]`. A solução é guardar o valor do lado direito em `temp 0` **antes** de mexer em `pointer 1`:

```
// let arr[expr1] = expr2
código VM de arr (push do segmento/índice de arr)
código VM de expr1
add                    // topo da pilha = endereço de arr[expr1]
código VM de expr2     // calcula o valor a atribuir
pop temp 0             // guarda o valor calculado, ANTES de mexer em pointer 1
pop pointer 1           // agora sim: that passa a apontar para arr[expr1]
push temp 0
pop that 0              // arr[expr1] = valor
```

```java
// dentro de parseLet, ao ver LBRACKET:
expectPeek(LBRACKET);
vmWriter.writePush(kind2Segment(symbol.kind()), symbol.index());
parseExpression();
vmWriter.writeArithmetic(Command.ADD);
expectPeek(RBRACKET);
isArray = true;
// ... depois de compilar a expressão do lado direito do '=':
if (isArray) {
    vmWriter.writePop(Segment.TEMP, 0);
    vmWriter.writePop(Segment.POINTER, 1);
    vmWriter.writePush(Segment.TEMP, 0);
    vmWriter.writePop(Segment.THAT, 0);
}
```

**Ler** um elemento de array (dentro de `parseTerm`, quando o identificador é seguido de `[`) é mais simples, porque não há um segundo acesso a `that` em conflito — apenas calcule o endereço, aponte `that` para lá, e empilhe o valor:

```java
vmWriter.writePush(kind2Segment(sym.kind()), sym.index());
parseExpression();
vmWriter.writeArithmetic(Command.ADD);
expectPeek(RBRACKET);
vmWriter.writePop(Segment.POINTER, 1);
vmWriter.writePush(Segment.THAT, 0);
```

Como exercício de verificação, vale compilar mentalmente (ou de fato, testando) uma expressão deliberadamente hostil como `let a[a[i]] = a[b[a[b[j]]]]` — se o `temp 0` intermediário estiver no lugar certo, mesmo esse emaranhado de acessos aninhados produz código correto.

## Fechando o Project 11

Com variáveis, funções, construtores, métodos, chamadas e arrays todos gerando código VM, o compilador Jack → VM está completo — qualquer arquivo `.jack` válido, de qualquer complexidade, agora compila para um `.vm` executável no VM Emulator do Nand2Tetris. É um bom momento para rodar contra os programas de teste do Project 11, nesta ordem — `Seven` (expressão com constantes, `do` e `return`), `ConvertToBin` (comandos procedurais e funções), `Square` (construtores, métodos e campos), `Average` (arrays e strings), `Pong` (aplicação orientada a objetos completa, com variáveis `static`) e `ComplexArrays` (índices de array aninhados) — e comparar a saída com a do `JackCompiler` de referência (pasta `tools/` do Nand2Tetris; o Project 11 não traz arquivos `.vm` esperados). Quando ela não bate byte a byte, normalmente ainda é possível validar comportamentalmente, executando ambos no VM Emulator e comparando o resultado observável. Para executar, carregue no emulador o diretório inteiro do programa: o VM Emulator já traz uma implementação embutida das oito classes do sistema operacional Jack; em qualquer outro ambiente, os oito arquivos `.vm` do sistema operacional (capítulo 7) precisam estar no mesmo diretório dos `.vm` gerados.

## O que vem a seguir

O compilador está pronto, mas ele só produz código VM — que ainda não roda em hardware nenhum. As Partes 3 e 4 fecham essa lacuna: o capítulo 11 especifica o Assembly Hack, a linguagem-alvo do próximo tradutor; os capítulos 12 e 13 constroem o VM Translator (primeiro aritmética e acesso à memória, depois controle de fluxo e o protocolo `call`/`return`); o capítulo 14 consolida o caminho percorrido; e o capítulo 15 constrói o Assembler.
