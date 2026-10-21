# 9. Geração de código: expressões e comandos

Chegou a hora de evoluir o `CompilationEngine` — que na Unidade 1 apenas *reconhecia* a estrutura de um programa Jack, emitindo uma árvore XML — para efetivamente **gerar código VM**. A boa notícia é que a maior parte do trabalho já está feita: a gramática, o parser recursivo descendente e a tabela de símbolos já existem. Falta "só" substituir `printNonTerminal` por chamadas que emitem comandos VM reais, nos pontos certos — exatamente a técnica de **tradução dirigida por sintaxe** formalizada no capítulo 1.

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

isto é, `(a + b) * c`. A figura 11.4 do livro *The Elements of Computing Systems* ilustra a geração de código para `x + g(2,y,-z) * 5` com a multiplicação feita antes da soma, mas aquele exemplo é sobre o algoritmo de travessia pós-ordem de uma árvore já construída; ele não é a regra de Jack. Para obter `a + (b * c)`, o programador precisa escrever os parênteses — e aí a recursão em `parseTerm` (que trata `'(' expression ')'`) compila `b * c` antes do `add`.

`*` e `/` são o único caso especial: como a VM não tem comandos nativos para eles (capítulo 7), viram chamadas a `Math.multiply`/`Math.divide` — o compilador "esconde" essa diferença do programador Jack, que escreve `a * b` sem saber que isso é, por baixo, uma chamada de função.

Operadores **unários** (`-x`, `~x`) seguem o padrão inverso de um binário: compilam o operando primeiro, emitem a operação depois — mas com apenas um operando na pilha:

```java
case MINUS: case NOT:
    TokenType unop = currentToken.type; // já consumido via expectPeek antes de chamar parseTerm
    parseTerm();
    vmWriter.writeArithmetic(unop == MINUS ? Command.NEG : Command.NOT);
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

Linguagens de alto nível têm estruturas de controle ricas; a VM só tem `goto` e `if-goto` (capítulo 7). Cabe ao compilador fazer esse mapeamento — e o ponto delicado é garantir que os **rótulos sejam únicos** dentro de cada subroutine (dois `if`s na mesma função não podem gerar o mesmo `IF_TRUE0`). A solução é um contador de instância, zerado a cada `parseSubroutineDec`:

```java
private int ifLabelNum = 0;
private int whileLabelNum = 0;
```

**`if`/`else`** mapeia para o seguinte esqueleto (o algoritmo aqui é o mesmo usado pelo compilador de referência do Nand2Tetris — não o único possível, mas o que facilita comparar sua saída com a oficial):

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

## O que levamos daqui

Neste capítulo, expressões e a maior parte dos comandos de Jack já geram código VM real — sempre seguindo o mesmo princípio: reconhecer a estrutura (o parser da Unidade 1 já faz isso) e, no ponto certo da produção, emitir o comando VM correspondente, sem nunca materializar uma árvore sintática intermediária. O que falta — variáveis (usando a tabela de símbolos do capítulo 8 para resolver `push`/`pop` corretos), subroutines completas (com prólogo de função e alocação de objetos) e arrays — é o assunto do capítulo 10, que fecha o Project 11 por completo.
