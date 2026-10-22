# 5. Analisador sintático

Com o `JackTokenizer` completo, chegamos à etapa central do front-end: o analisador sintático, ou `CompilationEngine`. Ele consome a sequência de tokens do capítulo anterior e reconhece a estrutura da gramática de Jack (capítulo 3) — por enquanto, apenas *reconhecendo*, sem gerar nenhum código ainda (isso fica para os capítulos 9 e 10, na Parte 2). A saída deste capítulo é uma árvore sintática em XML, que — assim como fizemos com os tokens — vamos validar por comparação byte a byte com arquivos de referência do próprio Nand2Tetris.

## Bottom-up ou top-down?

Existem duas famílias opostas de algoritmo para construir uma árvore sintática a partir de uma cadeia de tokens:

- **Parsers *bottom-up***: começam pelas folhas e fazem a árvore crescer em direção à raiz, identificando a cada passo uma substring da entrada que corresponde ao lado direito de alguma produção.
- **Parsers *top-down***: começam pela raiz (o não-terminal inicial) e fazem a árvore crescer em direção às folhas, expandindo a cada passo um não-terminal pendente com uma de suas produções.

Neste livro usamos **top-down**, na sua forma mais direta de implementar à mão: o **analisador preditivo recursivo**, que já introduzimos no capítulo 2. A eficiência (e a própria viabilidade) de um parser top-down depende de escolher a produção certa a cada passo, olhando apenas o token atual — sem *backtracking*. Uma gramática que permite essa escolha univocamente com um único símbolo de antecipação (*lookahead*) é chamada **LL(1)**: entrada lida da esquerda para a direita (*Left-to-right*), construindo a derivação mais à esquerda (*Leftmost*), com exatamente 1 símbolo de antecipação. Gramáticas LL(1) também são chamadas **preditivas**, e é exatamente esse o cuidado que tivemos, no capítulo 1, ao eliminar recursão à esquerda e fatorar a gramática: sem essas duas propriedades, um parser recursivo simplesmente não funciona.

A gramática de Jack do capítulo 3 é **quase** LL(1): não tem recursão à esquerda, e em quase todo não-terminal o primeiro token basta para escolher a produção (um comando que começa com `while` só pode ser um `whileStatement`, por exemplo). A única exceção está em `term` (e, pelo mesmo motivo, em `subroutineCall`): as alternativas `varName`, `varName '[' expression ']'` e as duas formas de `subroutineCall` começam todas com um `identifier`, e só o token **seguinte** — `[`, `(`, `.` ou nenhum deles — diz qual delas se aplica. Nesse ponto, e apenas nele, a gramática como está escrita é LL(2).

Não precisamos reescrever a gramática para lidar com isso. Basta aplicar, no código, a fatoração à esquerda do capítulo 1: o parser consome o `identifier` comum a todas as alternativas e só então decide, olhando o próximo token. Com essa fatoração implícita, cada decisão do parser é tomada com um único token de antecipação. A técnica de tradução é a mesma do capítulo 2: **um método por não-terminal**, mapeando cada produção diretamente em código.

## Tratamento de erros sintáticos

Diferente de erros léxicos (tipicamente erros de digitação — um símbolo que não existe na linguagem), erros sintáticos dizem respeito à *estrutura*: um `;` esquecido, um `let` sem `=`. Nossa estratégia de tratamento é deliberadamente simples: no primeiro erro encontrado, o parser para e emite uma mensagem no formato `Linha N: Esperado 'X' encontrado 'Y'`. Não faremos recuperação de erros nem tentaremos continuar a análise após o primeiro problema — isso é suficiente para os objetivos didáticos deste curso. (Também não implementaremos análise semântica neste analisador. A análise semântica deste livro se resume à tabela de símbolos do capítulo 8, que descobre o que cada identificador é, mas não verifica tipos: somar um `int` com um `boolean` não é detectado em nenhum ponto do compilador.)

## Estrutura básica: um token de antecipação

Boa parte da gramática de Jack pode ser decidida olhando apenas o token atual. Mas uma parte — notavelmente, distinguir uma variável simples de um acesso a array ou de uma chamada de subroutine dentro de `term` — exige olhar **um token à frente**. Por isso, de forma simétrica ao `peekNext()` do tokenizador, o parser mantém dois tokens em memória: o token atual (`currentToken`) e o próximo (`peekToken`):

```java
public class Parser {
    private static class ParseError extends RuntimeException {}

    private Scanner scan;
    private Token currentToken;
    private Token peekToken;
    private StringBuilder xmlOutput = new StringBuilder();

    public Parser(byte[] input) {
        scan = new Scanner(input);
        nextToken();   // primeira chamada: currentToken fica null, peekToken recebe o 1º token
        nextToken();   // segunda chamada: currentToken recebe o 1º, peekToken recebe o 2º
    }

    private void nextToken() {
        currentToken = peekToken;
        peekToken = scan.nextToken();
    }
}
```

Três funções auxiliares organizam toda a interação com os tokens — e vão aparecer em praticamente todo método deste capítulo:

```java
boolean peekTokenIs(TokenType type) {
    return peekToken.type == type;
}

private void expectPeek(TokenType type) {
    if (peekToken.type == type) {
        nextToken();
        xmlOutput.append(currentToken.toString()).append("\r\n");
    } else {
        throw error(peekToken, "Esperado " + type.name());
    }
}

private void expectPeek(TokenType... types) {
    for (TokenType type : types) {
        if (peekToken.type == type) {
            expectPeek(type);
            return;
        }
    }
    throw error(peekToken, "Esperado um dos tipos: " + Arrays.toString(types));
}
```

`expectPeek` é o equivalente, neste parser, do `match()` do capítulo 2 — mas com uma diferença de nome que reflete a diferença de estrutura: como mantemos `peekToken`, "casar" com um símbolo terminal significa verificar que o **próximo** token é do tipo esperado, avançar, e (novidade) registrar esse token na saída XML. Um método auxiliar `printNonTerminal(nome)` grava as tags de abertura/fechamento de cada não-terminal (`<term>`, `</term>`, etc.) — é o que torna a árvore sintática visível na saída. Completam o esqueleto o tratamento de erro, o ponto de entrada e o acesso à saída:

```java
private ParseError error(Token token, String message) {
    String encontrado = token.type == TokenType.EOF ? "fim do arquivo" : token.lexeme;
    System.err.println("Linha " + token.line + ": " + message + " encontrado '" + encontrado + "'");
    return new ParseError();
}

private void printNonTerminal(String nterminal) {
    xmlOutput.append("<").append(nterminal).append(">\r\n");   // "/term" gera </term>
}

public void parse() {
    parseClass();       // o símbolo inicial da gramática
}

public String XMLOutput() {
    return xmlOutput.toString();
}
```

`error` usa o número de linha que o tokenizador do capítulo 4 guarda em cada `Token`, produzindo mensagens como `Linha 12: Esperado SEMICOLON encontrado 'let'`. Ela *devolve* a exceção em vez de lançá-la, para que quem chama escreva `throw error(...)` — assim o compilador Java sabe que aquele ponto do código não continua.

## Term — começando pelo caso simples

A gramática de `term` (capítulo 3, [Apêndice A](../apendices/a-gramatica-jack.md)) tem oito alternativas. Vamos começar por apenas quatro, as mais diretas:

```
term -> integerConstant | stringConstant | keywordConstant | varName
```

```java
void parseTerm() {
    printNonTerminal("term");
    switch (peekToken.type) {
        case NUMBER:
            expectPeek(TokenType.NUMBER);
            break;
        case STRING:
            expectPeek(TokenType.STRING);
            break;
        case FALSE: case NULL: case TRUE:
            expectPeek(TokenType.FALSE, TokenType.NULL, TokenType.TRUE);
            break;
        case THIS:
            expectPeek(TokenType.THIS);
            break;
        case IDENT:
            expectPeek(TokenType.IDENT);
            break;
        default:
            throw error(peekToken, "term esperado");
    }
    printNonTerminal("/term");
}
```

Testando isoladamente com a entrada `"10;"`, chamando `parser.parseTerm()` diretamente (sem passar por `parse()`), a saída XML deve ser:

```xml
<term>
<integerConstant> 10 </integerConstant>
</term>
```

Esse hábito — testar cada não-terminal isoladamente, chamando o método de parse diretamente com uma entrada mínima — é o que torna viável construir um parser desse tamanho de forma incremental: cada método pode ser validado no mesmo instante em que é escrito, sem depender do resto da gramática já estar pronto.

## Expressão: term seguido de zero ou mais (op term)

```
expression -> term (op term)*
```

```java
static boolean isOperator(String op) {
    return !op.isEmpty() && "+-*/<>=&|".contains(op);
}

void parseExpression() {
    printNonTerminal("expression");
    parseTerm();
    while (isOperator(peekToken.lexeme)) {
        expectPeek(peekToken.type);
        parseTerm();
    }
    printNonTerminal("/expression");
}
```

Repare como a ausência de precedência de operadores em Jack (capítulo 3) se paga exatamente aqui: não precisamos de uma hierarquia `expression → term → factor → ...` para codificar níveis de precedência — um único laço `while` já captura toda a gramática de expressões. Testando com `"10+20"`:

```xml
<expression>
<term><integerConstant> 10 </integerConstant></term>
<symbol> + </symbol>
<term><integerConstant> 20 </integerConstant></term>
</expression>
```

## O comando `let`

```
letStatement -> 'let' varName ('[' expression ']')? '=' expression ';'
```

```java
void parseLet() {
    printNonTerminal("letStatement");
    expectPeek(TokenType.LET);
    expectPeek(TokenType.IDENT);

    if (peekTokenIs(TokenType.LBRACKET)) {
        expectPeek(TokenType.LBRACKET);
        parseExpression();
        expectPeek(TokenType.RBRACKET);
    }

    expectPeek(TokenType.EQ);
    parseExpression();
    expectPeek(TokenType.SEMICOLON);
    printNonTerminal("/letStatement");
}
```

O `if (peekTokenIs(LBRACKET))` é a tradução direta do `?` (zero ou uma vez) da gramática — o mesmo padrão que vamos repetir para toda parte opcional das demais regras (o `else` de `ifStatement`, o `expression?` de `returnStatement`, etc.).

## Chamada de subroutine e o comando `do`

Uma chamada de subroutine é o primeiro lugar onde o *lookahead* de um token realmente importa: ao ver um identificador, ainda não sabemos se é uma chamada direta (`hello()`) ou qualificada por classe/objeto (`Classe.metodo()` ou `objeto.metodo()`) — só o próximo token (`(` ou `.`) resolve a ambiguidade. Começamos pelo caso mais simples, deixando a forma qualificada como extensão natural do mesmo método:

```
subroutineCall -> subroutineName '(' expressionList ')'
                 | (className|varName) '.' subroutineName '(' expressionList ')'
```

```java
// versão inicial: apenas identifier '(' ')'
void parseSubroutineCall() {
    expectPeek(TokenType.IDENT);
    expectPeek(TokenType.LPAREN);
    expectPeek(TokenType.RPAREN);
}
```

E o comando `do`, que apenas envolve uma chamada de subroutine com a palavra-chave e o `;` final:

```
doStatement -> 'do' subroutineCall ';'
```

```java
void parseDo() {
    printNonTerminal("doStatement");
    expectPeek(TokenType.DO);
    parseSubroutineCall();
    expectPeek(TokenType.SEMICOLON);
    printNonTerminal("/doStatement");
}
```

Testando com `"do hello();"`:

```xml
<doStatement>
<keyword> do </keyword>
<identifier> hello </identifier>
<symbol> ( </symbol>
<symbol> ) </symbol>
<symbol> ; </symbol>
</doStatement>
```

## O identificador vem primeiro

A versão inicial de `parseSubroutineCall` consome o identificador por conta própria. Isso funciona no `do`, mas não dentro de `term`. Lá, ao ver um `identifier`, o parser ainda não sabe se está diante de uma variável (`x`), de um acesso a array (`a[i]`) ou de uma chamada (`f()`, `Classe.f()`, `obj.f()`) — e só descobre olhando o token **seguinte** ao identificador. É o trecho LL(2) da gramática, discutido no início do capítulo.

A solução é a fatoração à esquerda: `parseTerm` consome o identificador comum a todas as alternativas e só então decide, olhando o `peekToken`. Para isso, `parseSubroutineCall` passa a supor que o identificador inicial **já foi consumido** (é o `currentToken`):

```java
// pré-condição: o identificador inicial já é o currentToken
void parseSubroutineCall() {
    if (peekTokenIs(TokenType.LPAREN)) {           // subroutineName '(' expressionList ')'
        expectPeek(TokenType.LPAREN);
        parseExpressionList();
        expectPeek(TokenType.RPAREN);
    } else {                                       // (className | varName) '.' subroutineName '(' ... ')'
        expectPeek(TokenType.DOT);
        expectPeek(TokenType.IDENT);
        expectPeek(TokenType.LPAREN);
        parseExpressionList();
        expectPeek(TokenType.RPAREN);
    }
}
```

O caso `IDENT` de `parseTerm` fica:

```java
case IDENT:
    expectPeek(TokenType.IDENT);                  // consome o identificador comum
    if (peekTokenIs(TokenType.LPAREN) || peekTokenIs(TokenType.DOT)) {
        parseSubroutineCall();                    // chamada
    } else if (peekTokenIs(TokenType.LBRACKET)) {
        expectPeek(TokenType.LBRACKET);           // acesso a array
        parseExpression();
        expectPeek(TokenType.RBRACKET);
    }
    // nenhum dos dois: varName simples, nada mais a consumir
    break;
```

e `parseDo` passa a consumir o identificador antes de chamar `parseSubroutineCall`:

```java
void parseDo() {
    printNonTerminal("doStatement");
    expectPeek(TokenType.DO);
    expectPeek(TokenType.IDENT);
    parseSubroutineCall();
    expectPeek(TokenType.SEMICOLON);
    printNonTerminal("/doStatement");
}
```

A saída XML de `"do hello();"` não muda. É essa forma — identificador consumido antes de `parseSubroutineCall` — que os capítulos 9 e 10 vão estender com a geração de código.

## Quais não-terminais viram tags

Para que a saída bata com os arquivos de referência, nem todo não-terminal da gramática gera um par de tags. A especificação do Nand2Tetris define exatamente quais geram:

| Geram `<tag>` ... `</tag>` | Não geram tag |
|---|---|
| `class`, `classVarDec`, `subroutineDec`, `parameterList`, `subroutineBody`, `varDec` | `type`, `className`, `subroutineName`, `varName` — aparecem direto como `<keyword>` ou `<identifier>` |
| `statements`, `letStatement`, `ifStatement`, `whileStatement`, `doStatement`, `returnStatement` | `statement` — `parseStatements` chama `parseLet`, `parseIf` etc. diretamente; não existe `parseStatement` |
| `expression`, `term`, `expressionList` | `subroutineCall` — seus tokens aparecem dentro de `<doStatement>` ou de `<term>`, por isso `parseSubroutineCall` não chama `printNonTerminal` |

Dois detalhes costumam quebrar a comparação: `<parameterList>` e `<expressionList>` aparecem **mesmo quando a lista está vazia** (um par de tags sem nada dentro), e `parseStatements` emite `<statements>` mesmo para um bloco `{ }` vazio.

## Fechando o restante da gramática

A partir daqui, o padrão está estabelecido — cada não-terminal restante segue exatamente a mesma receita (um método, um `expectPeek` por terminal, uma chamada recursiva por não-terminal, um `if`/`while` por `?`/`*` da gramática). Fica como exercício guiado completar:

- `parseTerm` — os dois casos que faltam: expressão entre parênteses (`'(' expression ')'`) e operador unário (`unaryOp term`). Ambos se decidem pelo primeiro token (`(`, ou `-`/`~`), sem antecipação extra.
- `parseExpressionList`, `parseIf`, `parseWhile`, `parseReturn`, `parseStatements` — cada um uma tradução direta de sua produção na tabela do capítulo 3.
- `parseClassVarDec`, `parseSubroutineDec`, `parseParameterList`, `parseVarDec`, `parseSubroutineBody`, `parseClass` — a "casca" da classe, fechando o topo da gramática.

A API completa esperada do `CompilationEngine` (ou `Parser`) é:

| Rotina | Compila |
|---|---|
| `parseClass` | Uma classe completa |
| `parseClassVarDec` | Uma declaração `static` ou `field` |
| `parseSubroutineDec` | Um método, função ou construtor completo |
| `parseSubroutineBody` | O corpo de uma subroutine: declarações `var` e comandos, com as chaves |
| `parseParameterList` | Uma lista de parâmetros (possivelmente vazia), sem os parênteses |
| `parseVarDec` | Uma declaração `var` |
| `parseStatements` | Uma sequência de comandos, sem as chaves |
| `parseDo` / `parseLet` / `parseIf` / `parseWhile` / `parseReturn` | Cada tipo de comando |
| `parseExpression` / `parseTerm` / `parseExpressionList` | Expressões e suas partes |
| `parseSubroutineCall` | Uma chamada de subroutine (auxiliar, sem tag própria; supõe o identificador já consumido) |

## Validando contra a árvore sintática de referência

Assim como no capítulo 4, o Nand2Tetris fornece, para cada programa Jack de exemplo, um arquivo XML de referência — desta vez com a árvore sintática **completa** (não apenas os tokens). Testar o parser é, de novo, uma questão de comparação exata de texto:

```java
@Test
public void testParserWithSquare() throws IOException {
    var input = fromFile("Square/Square.jack");
    var expectedResult = fromFile("Square/Square.xml");

    var parser = new Parser(input.getBytes(StandardCharsets.UTF_8));
    parser.parse();
    var result = parser.XMLOutput();
    assertEquals(expectedResult, result);
}
```

O pacote de testes do Nand2Tetris inclui uma variante interessante para progressão incremental: os arquivos `ExpressionLessSquare/*.jack` substituem expressões complexas por espaços reservados simples, permitindo validar a "casca" da gramática (classes, subroutines, comandos) antes de enfrentar a gramática de expressões por completo.

!!! note "Dica de processo"
    Não tente compilar de uma vez um arquivo `.jack` inteiro contra a gramática completa. Construa o parser exatamente na ordem que fomos construindo aqui: primeiro `term` simples, depois `expression`, depois cada comando isoladamente (com testes próprios, como fizemos com `parseLet` e `parseDo`), só então subindo para `parseStatements`, `parseSubroutineDec` e finalmente `parseClass`. É a mesma disciplina de "um passo por vez, com teste a cada passo" que usamos desde o capítulo 2.

## O que vem a seguir

Com um parser completo (mesmo que ainda apenas reconhecendo a estrutura, sem gerar código), fechamos o front-end da Parte 1. Antes de gerar código de verdade, porém, a Parte 2 precisa definir *para onde* traduzir: o capítulo 6 discute representações intermediárias e o capítulo 7 especifica a VM Hack, nosso alvo. Só então faz sentido a última peça — a tabela de símbolos (capítulo 8), que diz **onde** cada identificador vive (é uma variável local? um campo do objeto? um parâmetro?) e **que tipo** ele tem — e que finalmente nos permite evoluir este reconhecedor em um compilador que produz código VM de verdade.
