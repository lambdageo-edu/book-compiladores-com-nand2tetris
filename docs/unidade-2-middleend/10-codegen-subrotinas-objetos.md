# 10. Geração de código: subrotinas e objetos

Este capítulo fecha o compilador Jack → VM: variáveis (usando a tabela de símbolos do capítulo 8), a declaração completa de `function`/`method`/`constructor` (com o prólogo que aloca variáveis locais e, para construtores, memória de objeto), chamadas de subroutine, e o mapeamento de arrays via aritmética de ponteiros. Ao final, o **Project 11** do Nand2Tetris estará completo: qualquer programa Jack válido poderá ser compilado para VM.

## O quadro geral: três níveis de memória

Os três problemas que restam do capítulo 9 — variáveis, objetos e arrays — são, no fundo, um só: **onde cada dado mora**. E há três respostas, uma por nível:

- o programa **Jack** fala em nomes: `low`, o campo `x` de um `Point`, o elemento `a[i]`;
- o programa **VM** fala em segmentos virtuais: `local 0`, `this 0`, `that 0`;
- o programa de **máquina** fala em endereços da RAM: `RAM[300]`, `RAM[6001]`.

Cada tradutor cobre uma das duas pontes. O **compilador** (este capítulo) só precisa decidir o segmento e o índice de cada nome; nunca vê um endereço de RAM. O **VM Translator** (capítulos 12 e 13) é quem posiciona os segmentos na RAM, em tempo de execução:

| Em Jack | Na VM (decidido pelo compilador) | Na RAM, em tempo de execução |
|---|---|---|
| variável local (`var`) | `local i` | no quadro da subroutine, na pilha, a partir do endereço guardado em `LCL` (`RAM[1]`) |
| parâmetro | `argument i` | na pilha, a partir do endereço em `ARG` (`RAM[2]`) |
| campo (`field`) | `this i` | no bloco do objeto, no heap, a partir do endereço em `THIS` (`RAM[3]`) |
| variável de classe (`static`) | `static i` | numa célula fixa, a partir de `RAM[16]` |
| elemento de array | `that i` | no bloco do array, no heap, a partir do endereço em `THAT` (`RAM[4]`) |

Repare em quem cria cada coisa. As variáveis locais e os parâmetros nascem e morrem com cada chamada, pelo protocolo `call`/`return`; o compilador só numera. Objetos e arrays nascem quando o programa chama `Memory.alloc` (direta ou indiretamente), em tempo de execução; o compilador só gera essa chamada e, depois, acessa o bloco pelos segmentos `this` e `that`. É por isso que o compilador consegue ser tão simples: ele raciocina sobre **segmentos**, e a memória concreta fica para depois.

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

### Fechando o exemplo do capítulo 9

Com isso, a função `raiz` do capítulo 9 fica completa. A tabela de símbolos da subroutine registra `x` como `argument 0` e `low`, `high`, `mid` como `local 0`, `1` e `2` — e cada nome do pseudocódigo vira o seu segmento:

```
function Main.raiz 3          // 3 variáveis locais (próxima seção)
push constant 0
pop local 0
push argument 0
push constant 1
add
pop local 1
label WHILE_EXP0
push local 1
push local 0
sub
push constant 1
gt
not
if-goto WHILE_END0
push local 1
push local 0
add
push constant 2
call Math.divide 2
pop local 2
push local 2
push local 2
call Math.multiply 2
push argument 0
gt
if-goto IF_TRUE0
goto IF_FALSE0
label IF_TRUE0
push local 2
pop local 1
goto IF_END0
label IF_FALSE0
push local 2
pop local 0
label IF_END0
goto WHILE_EXP0
label WHILE_END0
push local 0
return
```

O `3` do cabeçalho é o número de variáveis locais, que a tabela de símbolos também sabe informar — é o assunto da próxima seção.

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

## Objetos: o que o compilador precisa arranjar

Para os objetos, vamos acompanhar uma única classe do começo ao fim do capítulo:

```
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

e um trecho de código que a usa, em outra classe:

```
var Point p1, p2;
var int d;
let p1 = Point.new(2, 3);
let p2 = Point.new(5, 7);
let d = p1.distance(p2);
```

Depois das duas construções, a memória fica mais ou menos assim (os endereços do heap são os que `Memory.alloc` devolver; 6000 é só um exemplo):

```
  pilha (quadro de quem chama)             heap
  local 0 (p1) │ 6000 │ ──────────►  6000 │ 2 │  x de p1
  local 1 (p2) │ 6002 │ ─────┐       6001 │ 3 │  y de p1
  local 2 (d)  │   0  │      └────►  6002 │ 5 │  x de p2
                                     6003 │ 7 │  y de p2

  static 0 (Point.pointCount) = 2
```

Três observações orientam o resto do capítulo:

1. **Uma variável de objeto guarda só um endereço.** `p1` é uma variável local comum (`local 0`), que guarda o número 6000. Os campos `x` e `y` moram no heap, num bloco que o construtor alocou.
2. **Construir é chamar uma função que devolve esse endereço.** Para quem chama, `Point.new(2, 3)` é uma chamada como outra qualquer; o trabalho de alocar o bloco e preencher os campos fica todo do lado do construtor.
3. **Um método é uma função que recebe o objeto como primeiro argumento.** A VM não conhece objetos, então o compilador traduz a chamada orientada a objetos para uma chamada procedural, passando o objeto explicitamente:

    | Em Jack (orientado a objetos) | Na VM (procedural) |
    |---|---|
    | `p1.distance(p2)` | `Point.distance(p1, p2)` |
    | `p2.getx()` | `Point.getx(p2)` |
    | `obj.m(a, b)` | `Classe.m(obj, a, b)` |

    Dentro do método, a primeira instrução aponta `this` para esse argumento (`pointer 0 ← argument 0`). A partir daí, `this 0` é o `x` **daquele** objeto: durante `p1.distance(p2)`, `this 0` é `RAM[6000]`; durante `p2.getx()`, é `RAM[6002]`. O mesmo código serve para todos os objetos da classe.

As próximas seções implementam cada lado: o construtor, o método e a chamada.

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

(A versão da classe `Point` acima, com o contador `pointCount`, tem ainda `push static 0`, `push constant 1`, `add`, `pop static 0` antes do `return`: um `static` é só mais um segmento.)

Do lado de quem chama, não há nada de especial: `let p1 = Point.new(2, 3);` é uma chamada de função cujo resultado — o endereço do objeto novo — é guardado em `p1`:

```
push constant 2
push constant 3
call Point.new 2
pop local 0               // p1 = endereço do objeto novo
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

E, do lado de quem é chamado, o método `distance` compila para:

```
function Point.distance 2     // dx e dy
push argument 0
pop pointer 0                 // this = o objeto receptor (p1)
push this 0                   // x
push argument 1               // other
call Point.getx 1
sub
pop local 0                   // dx = x - other.getx()
push this 1                   // y
push argument 1
call Point.gety 1
sub
pop local 1                   // dy = y - other.gety()
push local 0
push local 0
call Math.multiply 2
push local 1
push local 1
call Math.multiply 2
add
call Math.sqrt 1
return
```

Repare num detalhe: `Point.getx`, chamado no meio, também começa com `pop pointer 0`, apontando `this` para `other`. Mesmo assim, o `push this 1` logo depois continua lendo o `y` de `p1`. Quem garante isso é o protocolo de chamada da VM: o `call` salva o `THIS` de quem chama, e o `return` o restaura (capítulo 13). O compilador pode contar com isso sem gerar nada.

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

## Resumo: o contrato entre quem chama e quem é chamado

Cada tipo de subroutine é um acordo entre dois trechos de código compilados **separadamente** — muitas vezes, em arquivos diferentes. O compilador precisa cumprir os dois lados:

| Situação | Quem chama gera | Quem é chamado gera |
|---|---|---|
| função: `Classe.f(a, b)` | `push a`, `push b`, `call Classe.f 2` | nada de especial |
| construtor: `Classe.new(a, b)` | o mesmo que uma função; o valor devolvido é o endereço do objeto novo | no início: `push constant n` (n = nº de campos), `call Memory.alloc 1`, `pop pointer 0`; no fim: `push pointer 0`, `return` |
| método de outro objeto: `obj.m(a)` | `push obj`, `push a`, `call Classe.m 2` | no início: `push argument 0`, `pop pointer 0` |
| método da própria classe: `m(a)` | `push pointer 0`, `push a`, `call Classe.m 2` | (o mesmo de cima) |
| subroutine `void` | no `do`, depois do `call`: `pop temp 0` | antes de `return`: `push constant 0` |

Se um dos lados descumprir o acordo, o erro aparece longe de onde nasceu: um método que esquece o `pop pointer 0` lê os campos de outro objeto; um `do` que esquece o `pop temp 0` deixa lixo na pilha, que corrompe um cálculo vários comandos depois.

## Arrays: aritmética de ponteiros

Um array Jack é, para o compilador, apenas um ponteiro para o endereço-base de um bloco contíguo no heap — `arr[i]` é açúcar sintático para "o valor no endereço `arr + i`". A VM não tem um comando de "acesso indexado" dedicado; ela oferece **endereçamento indireto** através do par `pointer 1` / segmento `that`: gravar um endereço em `pointer 1` faz `that 0` passar a se referir àquela célula de memória.

Para **ler** ou **escrever** uma célula, basta apontar `that` para o endereço `arr + i`. Por exemplo, `let arr[2] = 17;` vira:

```
push local 0          // arr: o endereço-base
push constant 2       // o índice
add                   // arr + 2
pop pointer 1         // that aponta para arr[2]
push constant 17
pop that 0            // arr[2] = 17
```

Generalizando para `let arr[expr1] = expr2`, a primeira ideia é trocar o `2` pelo código de `expr1` e o `17` pelo de `expr2`. Mas essa tentativa ingênua quebra quando o lado direito também acessa um array. Veja o que ela gera para `let a[i] = b[j]`:

```
push a
push i
add
pop pointer 1         // that aponta para a[i]...
push b
push j
add
pop pointer 1         // ...e agora para b[j]: o endereço de a[i] se perdeu!
push that 0           // lê b[j]
pop that 0            // grava em b[j], e não em a[i]
```

Como o compilador gera código da esquerda para a direita, na hora em que compila o lado direito ele já gastou o `pointer 1` com o lado esquerdo — e qualquer acesso a array dentro de `expr2` o sobrescreve. A solução é adiar o `pop pointer 1` do lado esquerdo: o endereço de `arr[expr1]` fica **na pilha** enquanto o lado direito é calculado, e o valor calculado espera em `temp 0` enquanto `pointer 1` é ajustado:

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

??? note "Gabarito (supondo `var Array a, b; var int i, j;`, ou seja, `local 0` a `local 3`)"
    ```
    // lado esquerdo: endereço de a[a[i]]
    push local 0        // a
    push local 0        // a
    push local 2        // i
    add
    pop pointer 1
    push that 0         // a[i]
    add                 // endereço de a[a[i]] fica na pilha
    // lado direito: a[b[a[b[j]]]], de dentro para fora
    push local 0        // a
    push local 1        // b
    push local 0        // a
    push local 1        // b
    push local 3        // j
    add
    pop pointer 1
    push that 0         // b[j]
    add
    pop pointer 1
    push that 0         // a[b[j]]
    add
    pop pointer 1
    push that 0         // b[a[b[j]]]
    add
    pop pointer 1
    push that 0         // a[b[a[b[j]]]]
    // a atribuição
    pop temp 0
    pop pointer 1       // that aponta para a[a[i]]
    push temp 0
    pop that 0
    ```

    Repare que o endereço do lado esquerdo fica **na pilha**, e não em `pointer 1`, durante toda a avaliação do lado direito — por isso as quatro leituras do lado direito podem usar `pointer 1` à vontade. Simulando essa sequência com arrays e índices aleatórios, o resultado bate sempre com a atribuição esperada.

## Teste rápido

??? question "1. Em `let d = p1.distance(p2);`, por que o compilador gera `call Point.distance 2`, se o programador escreveu um argumento só?"
    Porque `distance` é um método, e o objeto receptor (`p1`) é passado como primeiro argumento implícito: `p1.distance(p2)` vira `Point.distance(p1, p2)`. Por isso o compilador empilha `p1` antes de `p2` e soma 1 ao número de argumentos.

??? question "2. Supondo `p1` em `local 0`, que código VM o comando `do p1.print();` gera (sendo `print` um método `void`)?"
    ```
    push local 0          // o receptor
    call Point.print 1
    pop temp 0            // descarta o 0 que todo void devolve
    ```

??? question "3. Uma classe tem `field int x, y, z;` e `static int contador;`. Que valor o construtor passa para `Memory.alloc`?"
    `3`. Só os campos (`field`) ficam no bloco do objeto; a variável `static` é única para a classe toda e mora no segmento `static`, fora do heap.

??? question "4. Dentro de um método, `let x = 0;` (sendo `x` o primeiro campo) e, numa função, `let x = 0;` (sendo `x` a primeira variável local) geram o mesmo código?"
    Não. No método, `x` é um campo: `push constant 0`, `pop this 0`. Na função, é uma variável local: `push constant 0`, `pop local 0`. Quem decide é a categoria guardada na tabela de símbolos, não o nome.

## Fechando o Project 11

Com variáveis, funções, construtores, métodos, chamadas e arrays todos gerando código VM, o compilador Jack → VM está completo — qualquer arquivo `.jack` válido, de qualquer complexidade, agora compila para um `.vm` executável no VM Emulator do Nand2Tetris. É um bom momento para rodar contra os programas de teste do Project 11, nesta ordem — `Seven` (expressão com constantes, `do` e `return`), `ConvertToBin` (comandos procedurais e funções), `Square` (construtores, métodos e campos), `Average` (arrays e strings), `Pong` (aplicação orientada a objetos completa, com variáveis `static`) e `ComplexArrays` (índices de array aninhados) — e comparar a saída com a do `JackCompiler` de referência (pasta `tools/` do Nand2Tetris; o Project 11 não traz arquivos `.vm` esperados). Quando ela não bate byte a byte, normalmente ainda é possível validar comportamentalmente, executando ambos no VM Emulator e comparando o resultado observável. Para executar, carregue no emulador o diretório inteiro do programa: o VM Emulator já traz uma implementação embutida das oito classes do sistema operacional Jack; em qualquer outro ambiente, os oito arquivos `.vm` do sistema operacional (capítulo 7) precisam estar no mesmo diretório dos `.vm` gerados.

## O que vem a seguir

O compilador está pronto, mas ele só produz código VM — que ainda não roda em hardware nenhum. As Partes 3 e 4 fecham essa lacuna: o capítulo 11 especifica o Assembly Hack, a linguagem-alvo do próximo tradutor; os capítulos 12 e 13 constroem o VM Translator (primeiro aritmética e acesso à memória, depois controle de fluxo e o protocolo `call`/`return`); o capítulo 14 consolida o caminho percorrido; e o capítulo 15 constrói o Assembler.
