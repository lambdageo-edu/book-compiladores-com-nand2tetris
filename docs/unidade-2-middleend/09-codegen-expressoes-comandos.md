# 9. Geração de código: expressões e comandos

Chegou a hora de evoluir o `CompilationEngine` — que na Parte 1 apenas *reconhecia* a estrutura de um programa Jack, emitindo uma árvore XML — para efetivamente **gerar código VM**. A boa notícia é que a maior parte do trabalho já está feita: a gramática, o parser recursivo descendente e a tabela de símbolos já existem. Falta "só" substituir `printNonTerminal` por chamadas que emitem comandos VM reais, nos pontos certos — exatamente a técnica de **tradução dirigida por sintaxe** formalizada no capítulo 1.

## O desafio: um programa, cinco problemas

Antes de escrever qualquer linha do compilador, vale olhar para o problema de frente. A função abaixo calcula a raiz quadrada inteira de `x` por busca binária: mantém um intervalo `[low, high)` que sempre contém a resposta e o divide ao meio até sobrar um único número.

```
class Main {
    function int raiz(int x) {
        var int low, high, mid;
        let low = 0;
        let high = x + 1;
        while ((high - low) > 1) {
            let mid = (high + low) / 2;
            if ((mid * mid) > x) {
                let high = mid;
            } else {
                let low = mid;
            }
        }
        return low;
    }
}
```

Traduzi-la para a VM Hack, que só conhece `push`, `pop`, aritmética, `label`, `goto`, `if-goto`, `function`, `call` e `return`, exige resolver cinco problemas diferentes — e é essa a divisão dos dois capítulos de geração de código:

| Problema | Na função acima | Onde resolvemos |
|---|---|---|
| **Expressões** | `(high + low) / 2`, `(mid * mid) > x` | este capítulo |
| **Controle de fluxo** | o `while` e o `if`/`else` | este capítulo |
| **Variáveis** | `x`, `low`, `high`, `mid` | capítulo 10 |
| **Objetos** | (não há aqui) construtores, métodos, campos | capítulo 10 |
| **Arrays** | (não há aqui) `a[i]` | capítulo 10 |

Para cada construção, vamos seguir sempre a mesma ordem: primeiro **o que** gerar — um molde mostrando o código Jack e o código VM lado a lado —, e só depois **como** gerar, no `CompilationEngine`. Ao final do capítulo, voltaremos a esta função e a veremos inteiramente traduzida.

## O VMWriter

Assim como tivemos um `Scanner` dedicado a produzir tokens, vamos ter uma classe dedicada a produzir código VM: o `VMWriter`. Sua única responsabilidade é formatar comandos — ele não sabe nada sobre a gramática de Jack, apenas escreve o que lhe é pedido:

```java
public class VMWriter {
    public enum Segment { CONST, ARG, LOCAL, STATIC, THIS, THAT, POINTER, TEMP }
    public enum Command { ADD, SUB, NEG, EQ, GT, LT, AND, OR, NOT }

    private StringBuilder output = new StringBuilder();

    public void writePush(Segment seg, int index) {
        output.append("push %s %d\n".formatted(seg.name().toLowerCase(), index));
    }
    public void writePop(Segment seg, int index) {
        output.append("pop %s %d\n".formatted(seg.name().toLowerCase(), index));
    }
    public void writeArithmetic(Command cmd) {
        output.append(cmd.name().toLowerCase() + "\n");
    }
    public void writeLabel(String label) { output.append("label " + label + "\n"); }
    public void writeGoto(String label)  { output.append("goto " + label + "\n"); }
    public void writeIf(String label)    { output.append("if-goto " + label + "\n"); }
    public void writeCall(String name, int nArgs) {
        output.append("call %s %d\n".formatted(name, nArgs));
    }
    public void writeFunction(String name, int nLocals) {
        output.append("function %s %d\n".formatted(name, nLocals));
    }
    public void writeReturn() { output.append("return\n"); }

    public String output() { return output.toString(); }
}
```

O `Parser` ganha uma instância de `VMWriter` (além do `SymbolTable` do capítulo 8), e um método `VMOutput()` espelhando o `XMLOutput()` que já existia. A partir daqui, cada seção da gramática de expressões e comandos ganha, além do reconhecimento sintático, a emissão de código correspondente — construída e testada incrementalmente, como sempre.

## Expressões: da árvore à pilha

Uma expressão pode ser escrita de três formas, conforme a posição do operador:

| Notação | `a * (b + c)` escrita como | Para quem é natural |
|---|---|---|
| prefixa | `* a + b c` (ou `*(a, +(b, c))`) | funções: `f(x, y)` |
| infixa | `a * (b + c)` | pessoas (e Jack) |
| pós-fixa | `a b c + *` | máquinas de pilha |

A VM é uma máquina de pilha, então gerar código para uma expressão é, no fundo, **converter a forma infixa na pós-fixa**: cada operando vira um `push`, e cada operador vem **depois** dos seus operandos, consumindo-os do topo da pilha. Quando o código termina de executar, o valor da expressão está no topo da pilha.

Uma forma de ver a conversão é pela árvore da expressão. Para `x + g(2, y, -z) * 5` — lembrando que Jack avalia da esquerda para a direita, sem precedência (capítulo 3), o que dá `(x + g(2, y, -z)) * 5` —, a árvore é:

```
              *
            /   \
           +     5
          / \
         x   g
           / | \
          2  y  -
                |
                z
```

Percorrer a árvore em **pós-ordem** (primeiro os filhos, da esquerda para a direita, depois o nó) produz exatamente o código de pilha:

```
push x
push constant 2
push y
push z
neg            // o "-" unário
call g 3
add
push constant 5
call Math.multiply 2
```

O percurso cabe numa função recursiva de poucas linhas, que vale ter em mente durante todo o capítulo (aqui em pseudocódigo):

```
gerar(exp):
    se exp é um número n:            emita "push constant n"
    se exp é uma variável v:         emita "push v"     (o segmento certo vem no capítulo 10)
    se exp é "exp1 op exp2":         gerar(exp1); gerar(exp2); emita op
    se exp é "op exp1" (unário):     gerar(exp1); emita op
    se exp é "f(exp1, ..., expN)":   gerar(exp1); ...; gerar(expN); emita "call f N"
```

O nosso compilador **não** constrói essa árvore. Ele faz a mesma coisa num único passo: o parser recursivo da Parte 1 já visita a expressão na ordem certa, e a pilha de chamadas de `parseExpression`/`parseTerm` faz o papel da árvore. Basta emitir o comando VM do nó **na hora em que o método termina de compilar os filhos**. Compare a saída do parser da Parte 1 com a que vamos gerar agora, para `let x = a + b - c;`:

=== "Parte 1: só reconhecer"

    ```xml
    <letStatement>
      <keyword> let </keyword>
      <identifier> x </identifier>
      <symbol> = </symbol>
      <expression>
        <term> <identifier> a </identifier> </term>
        <symbol> + </symbol>
        <term> <identifier> b </identifier> </term>
        <symbol> - </symbol>
        <term> <identifier> c </identifier> </term>
      </expression>
      <symbol> ; </symbol>
    </letStatement>
    ```

=== "Parte 2: reconhecer e gerar"

    ```
    push a
    push b
    add
    push c
    sub
    pop x
    ```

(Aqui, e até o capítulo 10, escrevemos as variáveis pelo nome — `push a`, `pop x` —, como pseudocódigo; constantes já aparecem na forma real, `push constant n`. Trocar cada nome pelo segmento certo, como `push local 0`, é o problema das variáveis, que a tabela de símbolos resolve no capítulo 10.)

As próximas seções implementam a função `gerar` caso a caso, dentro dos métodos que já temos: literais e variáveis em `parseTerm`, operadores em `parseExpression`.

## Literais

Comecemos pelo caso mais simples: um `term` que é uma constante inteira já reconhecida vira, imediatamente, um `push constant`:

```java
case NUMBER:
    expectPeek(NUMBER);
    vmWriter.writePush(Segment.CONST, Integer.parseInt(currentToken.lexeme));
    break;
```

```java
@Test
void testInt() {
    var parser = new Parser("10".getBytes());
    parser.parseExpression();
    assertEquals("push constant 10\n", parser.VMOutput());
}
```

Os literais `true`/`false`/`null`/`this` seguem a mesma receita, mas cada um mapeia para uma sequência VM diferente — vale memorizar essas quatro traduções, porque vão aparecer o tempo todo:

| Literal Jack | Código VM | Por quê |
|---|---|---|
| `false` | `push constant 0` | falso é 0 |
| `null` | `push constant 0` | `null` também é representado como 0 |
| `true` | `push constant 0` seguido de `not` | não existe literal `-1`; `not 0` produz `-1` (todos os bits em 1) — a especificação sugere `push constant 1` seguido de `neg`, que dá o mesmo resultado |
| `this` | `push pointer 0` | `this` é sempre o conteúdo do registrador `pointer 0` (capítulo 7) |

```java
case FALSE: case NULL: case TRUE:
    expectPeek(FALSE, NULL, TRUE);
    vmWriter.writePush(Segment.CONST, 0);
    if (currentToken.type == TRUE) vmWriter.writeArithmetic(Command.NOT);
    break;
case THIS:
    expectPeek(THIS);
    vmWriter.writePush(Segment.POINTER, 0);
    break;
```

Uma string literal exige mais trabalho, porque a VM não tem um tipo string nativo — ela é construída, caractere a caractere, através da classe `String` do sistema operacional Jack:

```java
case STRING:
    expectPeek(STRING);
    String s = currentToken.lexeme;
    vmWriter.writePush(Segment.CONST, s.length());
    vmWriter.writeCall("String.new", 1);
    for (int i = 0; i < s.length(); i++) {
        vmWriter.writePush(Segment.CONST, s.charAt(i));
        vmWriter.writeCall("String.appendChar", 2);
    }
    break;
```

Isso já revela um padrão que vai se repetir: uma construção de alto nível (aqui, uma string literal) muitas vezes não vira um único comando VM, mas uma **sequência** de `push`/`call` que invoca a biblioteca padrão.

## Operadores binários e unários

`expression → term (op term)*` (capítulo 3/5) já era reconhecida; agora, a cada `op` reconhecido, emitimos o comando correspondente **depois** de compilar ambos os termos — a ordem pós-fixa que a VM exige:

```java
void parseExpression() {
    parseTerm();
    while (isOperator(peekToken.lexeme)) {
        TokenType op = peekToken.type;
        expectPeek(op);
        parseTerm();
        compileOperator(op);
    }
}

void compileOperator(TokenType op) {
    switch (op) {
        case ASTERISK -> vmWriter.writeCall("Math.multiply", 2);
        case SLASH    -> vmWriter.writeCall("Math.divide", 2);
        case PLUS  -> vmWriter.writeArithmetic(Command.ADD);
        case MINUS -> vmWriter.writeArithmetic(Command.SUB);
        case LT -> vmWriter.writeArithmetic(Command.LT);
        case GT -> vmWriter.writeArithmetic(Command.GT);
        case EQ -> vmWriter.writeArithmetic(Command.EQ);
        case AND -> vmWriter.writeArithmetic(Command.AND);
        case OR  -> vmWriter.writeArithmetic(Command.OR);
    }
}
```

Esse laço emite cada operador assim que o termo à sua direita foi compilado — ou seja, avalia da esquerda para a direita, sem precedência. É a convenção do compilador de referência do Nand2Tetris (capítulo 3): `a + b * c` gera

```
push a
push b
add
push c
call Math.multiply 2
```

isto é, `(a + b) * c`. A figura 11.4 da 1ª edição de *The Elements of Computing Systems* (seção 11.1.2) ilustra a geração de código para `x + g(2,y,-z) * 5` com a multiplicação feita antes da soma, mas aquele exemplo é sobre o algoritmo de travessia pós-ordem de uma árvore já construída; ele não é a regra de Jack. Para obter `a + (b * c)`, o programador precisa escrever os parênteses — e aí a recursão em `parseTerm` (que trata `'(' expression ')'`) compila `b * c` antes do `add`.

`*` e `/` são o único caso especial: como a VM não tem comandos nativos para eles (capítulo 7), viram chamadas a `Math.multiply`/`Math.divide` — o compilador "esconde" essa diferença do programador Jack, que escreve `a * b` sem saber que isso é, por baixo, uma chamada de função.

Operadores **unários** (`-x`, `~x`) seguem o mesmo princípio pós-fixo de um binário: compilam o operando primeiro e emitem a operação depois — mas com apenas um operando na pilha. Dentro de `parseTerm`, o operador é consumido, guardado, e só emitido depois do termo:

```java
case MINUS: case NOT:
    expectPeek(MINUS, NOT);
    TokenType unop = currentToken.type;   // guarda o operador
    parseTerm();                          // compila o operando
    vmWriter.writeArithmetic(unop == MINUS ? Command.NEG : Command.NOT);
    break;
```

Repare que o mesmo `-` vira `sub` quando é binário (em `parseExpression`) e `neg` quando é unário (em `parseTerm`): quem decide é a posição na gramática, não o símbolo.

Uma expressão entre parênteses não emite nada de próprio — os parênteses só agrupam. Basta compilar a expressão interna, e o valor dela fica no topo da pilha:

```java
case LPAREN:
    expectPeek(LPAREN);
    parseExpression();
    expectPeek(RPAREN);
    break;
```

## O comando `return`

```java
void parseReturn() {
    expectPeek(RETURN);
    if (!peekTokenIs(SEMICOLON)) {
        parseExpression();
    } else {
        vmWriter.writePush(Segment.CONST, 0);  // void: empilha um valor "nulo" por convenção
    }
    expectPeek(SEMICOLON);
    vmWriter.writeReturn();
}
```

Isso implementa diretamente a convenção do capítulo 7: toda subroutine, `void` ou não, precisa deixar exatamente um valor na pilha antes do `return`.

## Controle de fluxo: `if` e `while`

Linguagens de alto nível têm estruturas de controle ricas; a VM só tem `goto` e `if-goto` (capítulo 7). Cabe ao compilador fazer esse mapeamento — e o ponto delicado é garantir que os **rótulos sejam únicos** dentro de cada subroutine (dois `if`s na mesma função não podem gerar o mesmo `IF_TRUE0`). A solução é um contador de instância, zerado a cada `parseSubroutineDec` (a versão completa desse método está no capítulo 10):

```java
private int ifLabelNum = 0;
private int whileLabelNum = 0;

// no início de parseSubroutineDec:
ifLabelNum = 0;
whileLabelNum = 0;
```

Zerar a cada subroutine é suficiente porque rótulos VM têm escopo de função (capítulo 7): o `IF_TRUE0` de `Main.main` e o de `Main.foo` não colidem.

Comecemos pelos moldes, na forma mais enxuta, que é a que costuma aparecer nos livros-texto (e nos slides do Nand2Tetris). Os rótulos `L1`, `L2` são inventados pelo compilador:

=== "if/else"

    ```
    // if (cond) { s1 } else { s2 }
        código VM de cond
        not
        if-goto L1          // condição falsa: pula para o else
        código VM de s1
        goto L2
    label L1
        código VM de s2
    label L2
    ```

=== "while"

    ```
    // while (cond) { s }
    label L1
        código VM de cond
        not
        if-goto L2          // condição falsa: sai do laço
        código VM de s
        goto L1
    label L2
    ```

Em ambos, o código de cada parte (`cond`, `s1`, `s2`, `s`) é gerado pelos métodos que já existem, recursivamente — um `if` dentro de um `while` simplesmente aparece dentro do "código VM de s". O compilador só precisa encaixar as peças e inventar rótulos que não colidam.

**`if`/`else`**, no nosso compilador, usa uma variação desse molde: o seguinte esqueleto (o algoritmo aqui é o mesmo usado pelo compilador de referência do Nand2Tetris — não o único possível, mas o que facilita comparar sua saída com a oficial):

```
código VM da condição
if-goto IF_TRUE0
goto IF_FALSE0
label IF_TRUE0
    código VM do bloco then
    goto IF_END0        // só existe se houver else
label IF_FALSE0
    código VM do bloco else       // só existe se houver else
label IF_END0                     // só existe se houver else
```

```java
void parseIf() {
    String labelTrue = "IF_TRUE" + ifLabelNum;
    String labelFalse = "IF_FALSE" + ifLabelNum;
    String labelEnd = "IF_END" + ifLabelNum;
    ifLabelNum++;

    expectPeek(IF); expectPeek(LPAREN);
    parseExpression();
    expectPeek(RPAREN);

    vmWriter.writeIf(labelTrue);
    vmWriter.writeGoto(labelFalse);
    vmWriter.writeLabel(labelTrue);

    expectPeek(LBRACE); parseStatements(); expectPeek(RBRACE);
    if (peekTokenIs(ELSE)) vmWriter.writeGoto(labelEnd);

    vmWriter.writeLabel(labelFalse);
    if (peekTokenIs(ELSE)) {
        expectPeek(ELSE);
        expectPeek(LBRACE); parseStatements(); expectPeek(RBRACE);
        vmWriter.writeLabel(labelEnd);
    }
}
```

**`while`** é mais simples, porque não há ramificação — só um teste no início do laço:

```
label WHILE_EXP0
    código VM da condição
    not
    if-goto WHILE_END0
    código VM do corpo
    goto WHILE_EXP0
label WHILE_END0
```

O `not` antes do `if-goto` é o detalhe que vale internalizar: a VM só tem "salte se verdadeiro" — para "saia do laço quando a condição for **falsa**", invertemos a condição antes de testar.

```java
void parseWhile() {
    String labelExp = "WHILE_EXP" + whileLabelNum;
    String labelEnd = "WHILE_END" + whileLabelNum;
    whileLabelNum++;

    vmWriter.writeLabel(labelExp);
    expectPeek(WHILE); expectPeek(LPAREN);
    parseExpression();
    vmWriter.writeArithmetic(Command.NOT);
    vmWriter.writeIf(labelEnd);
    expectPeek(RPAREN);

    expectPeek(LBRACE); parseStatements(); expectPeek(RBRACE);
    vmWriter.writeGoto(labelExp);
    vmWriter.writeLabel(labelEnd);
}
```

## O exemplo inicial, traduzido

Com expressões e controle de fluxo, já conseguimos traduzir quase toda a função `raiz` do início do capítulo — ainda com as variáveis pelo nome:

```
function Main.raiz ?          // quantas variáveis locais? (capítulo 10)
    push constant 0
    pop low                   // let low = 0;
    push x
    push constant 1
    add
    pop high                  // let high = x + 1;
label WHILE_EXP0
    push high
    push low
    sub
    push constant 1
    gt                        // (high - low) > 1
    not
    if-goto WHILE_END0
    push high
    push low
    add
    push constant 2
    call Math.divide 2
    pop mid                   // let mid = (high + low) / 2;
    push mid
    push mid
    call Math.multiply 2
    push x
    gt                        // (mid * mid) > x
    if-goto IF_TRUE0
    goto IF_FALSE0
label IF_TRUE0
    push mid
    pop high                  // let high = mid;
    goto IF_END0
label IF_FALSE0
    push mid
    pop low                   // let low = mid;
label IF_END0
    goto WHILE_EXP0
label WHILE_END0
    push low
    return                    // return low;
```

Dos cinco problemas, dois estão resolvidos. Os nomes `x`, `low`, `high` e `mid` — e o `?` do cabeçalho — são o que falta: é com eles que o capítulo 10 começa.

!!! tip "Verifique você mesmo"
    Esse código (com as variáveis já trocadas pelos segmentos, como no capítulo 10) dá a resposta certa para todo `x` de 0 a 361. Acima disso, `mid * mid` passa de 32767 e estoura os 16 bits da máquina — um lembrete de que a VM Hack só conhece inteiros de 16 bits.

## O que levamos daqui

Neste capítulo, expressões e a maior parte dos comandos de Jack já geram código VM real — sempre seguindo o mesmo princípio: reconhecer a estrutura (o parser da Parte 1 já faz isso) e, no ponto certo da produção, emitir o comando VM correspondente, sem nunca materializar uma árvore sintática intermediária. O que falta — variáveis (usando a tabela de símbolos do capítulo 8 para resolver `push`/`pop` corretos), subroutines completas (com prólogo de função e alocação de objetos) e arrays — é o assunto do capítulo 10, que fecha o Project 11 por completo.
