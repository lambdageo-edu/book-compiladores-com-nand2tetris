# 4. Analisador léxico

Com a linguagem Jack conhecida (capítulo 3) e a técnica de análise léxica dominada no tradutor de aquecimento (capítulo 2), vamos construir o analisador léxico de verdade: o `JackTokenizer` (ou, seguindo a nomenclatura que usaremos, a evolução da classe `Scanner` do capítulo 2). Partimos exatamente de onde paramos — um scanner ad-hoc que já reconhece números, identificadores, uma palavra-chave e alguns símbolos — e vamos, seção por seção, estendê-lo até cobrir as cinco categorias léxicas completas de Jack: `keyword`, `symbol`, `integerConstant`, `stringConstant` e `identifier`.

## Ponto de partida

Ao final do capítulo 2, nosso `Scanner` tinha essa forma (resumida):

```java
class Scanner {
    private byte[] input;
    private int current;

    private static final Map<String, TokenType> keywords = new HashMap<>();
    static {
        keywords.put("let", TokenType.LET);
        keywords.put("print", TokenType.PRINT);
    }

    public Token nextToken() {
        skipWhitespace();
        char ch = peek();
        if (isAlpha(ch)) return identifier();
        if (Character.isDigit(ch)) return number();
        switch (ch) {
            case '+': advance(); return new Token(TokenType.PLUS, "+");
            case '-': advance(); return new Token(TokenType.MINUS, "-");
            case '=': advance(); return new Token(TokenType.EQ, "=");
            case ';': advance(); return new Token(TokenType.SEMICOLON, ";");
            case '\0': return new Token(TokenType.EOF, "EOF");
            default: throw new Error("lexical error at " + ch);
        }
    }
}
```

Reconhecer inteiros e identificadores já está pronto — a lógica não muda para Jack, exceto por um detalhe: uma `integerConstant` de Jack vai de 0 a 32767 (o maior inteiro positivo de 16 bits; `-5` é o operador unário aplicado a `5`), então `number()` deve rejeitar, com erro léxico, um lexema acima desse valor. O que falta é: o conjunto completo de palavras reservadas, o conjunto completo de símbolos, strings entre aspas, comentários (de linha e de bloco) e contagem de linha (para mensagens de erro úteis).

## Palavras reservadas completas

Jack tem 21 palavras-chave. Adicionamos cada uma ao `TokenType` e à tabela `keywords` — mecanicamente, sem lógica nova:

```java
public enum TokenType {
    // symbols
    LPAREN, RPAREN, LBRACE, RBRACE, LBRACKET, RBRACKET,
    COMMA, SEMICOLON, DOT, PLUS, MINUS, ASTERISK, SLASH,
    AND, OR, NOT, LT, GT, EQ,

    // literals
    NUMBER, IDENT, STRING,

    // keywords
    CLASS, CONSTRUCTOR, FUNCTION, METHOD, FIELD, STATIC, VAR,
    INT, CHAR, BOOLEAN, VOID, TRUE, FALSE, NULL, THIS,
    LET, DO, IF, ELSE, WHILE, RETURN,

    EOF
}
```

```java
static {
    keywords.put("class", TokenType.CLASS);
    keywords.put("constructor", TokenType.CONSTRUCTOR);
    keywords.put("function", TokenType.FUNCTION);
    keywords.put("method", TokenType.METHOD);
    keywords.put("field", TokenType.FIELD);
    keywords.put("static", TokenType.STATIC);
    keywords.put("var", TokenType.VAR);
    keywords.put("int", TokenType.INT);
    keywords.put("char", TokenType.CHAR);
    keywords.put("boolean", TokenType.BOOLEAN);
    keywords.put("void", TokenType.VOID);
    keywords.put("true", TokenType.TRUE);
    keywords.put("false", TokenType.FALSE);
    keywords.put("null", TokenType.NULL);
    keywords.put("this", TokenType.THIS);
    keywords.put("let", TokenType.LET);
    keywords.put("do", TokenType.DO);
    keywords.put("if", TokenType.IF);
    keywords.put("else", TokenType.ELSE);
    keywords.put("while", TokenType.WHILE);
    keywords.put("return", TokenType.RETURN);
}
```

Testando com `"45 variavel while if"`, o scanner já devolve `<NUMBER>45</NUMBER>`, `<IDENT>variavel</IDENT>`, `<WHILE>while</WHILE>`, `<IF>if</IF>` — a mesma técnica do capítulo 2 (reconhece como identificador, depois reclassifica se estiver na tabela), só que agora com o vocabulário completo de Jack.

## Todos os símbolos

Jack usa 19 símbolos de um único caractere: `{ } ( ) [ ] . , ; + - * / & | < > = ~`. Cada um vira um `case` no `switch` de `nextToken`:

```java
switch (ch) {
    case '+': advance(); return new Token(TokenType.PLUS, "+");
    case '-': advance(); return new Token(TokenType.MINUS, "-");
    case '*': advance(); return new Token(TokenType.ASTERISK, "*");
    case '.': advance(); return new Token(TokenType.DOT, ".");
    case '&': advance(); return new Token(TokenType.AND, "&");
    case '|': advance(); return new Token(TokenType.OR, "|");
    case '~': advance(); return new Token(TokenType.NOT, "~");
    case '>': advance(); return new Token(TokenType.GT, ">");
    case '<': advance(); return new Token(TokenType.LT, "<");
    case '=': advance(); return new Token(TokenType.EQ, "=");
    case '(': advance(); return new Token(TokenType.LPAREN, "(");
    case ')': advance(); return new Token(TokenType.RPAREN, ")");
    case '{': advance(); return new Token(TokenType.LBRACE, "{");
    case '}': advance(); return new Token(TokenType.RBRACE, "}");
    case '[': advance(); return new Token(TokenType.LBRACKET, "[");
    case ']': advance(); return new Token(TokenType.RBRACKET, "]");
    case ';': advance(); return new Token(TokenType.SEMICOLON, ";");
    case ',': advance(); return new Token(TokenType.COMMA, ",");
    // '/' é tratado à parte — ver seção de comentários
}
```

## Reconhecendo strings

Uma `stringConstant` em Jack é uma sequência de caracteres Unicode entre aspas duplas, sem suporte a escape. A função segue o mesmo padrão de `number()` e `identifier()`: consumir enquanto a condição de parada não for satisfeita, então fatiar a substring:

```java
private Token string() {
    advance();              // consome a aspa de abertura
    int start = current;
    while (peek() != '"' && peek() != '\0') {
        advance();
    }
    String s = new String(input, start, current - start, StandardCharsets.UTF_8);
    advance();               // consome a aspa de fechamento
    return new Token(TokenType.STRING, s);
}
```

registrada em `nextToken` com `case '"': return string();`.

## Ignorando comentários

Jack aceita comentários de linha (`// ...`) e de bloco (`/* ... */`, incluindo a variante de documentação `/** ... */`, tratada da mesma forma). O detalhe interessante aqui é que o caractere `/` sozinho é ambíguo: pode ser o operador de divisão, o início de um comentário de linha, ou o início de um comentário de bloco — só sabemos qual ao olhar **um caractere à frente** do `/`. Isso motiva uma nova função auxiliar, `peekNext()`, que olha adiante sem consumir:

```java
private char peekNext() {
    int next = current + 1;
    return next < input.length ? (char) input[next] : '\0';
}
```

Com ela, o `case '/'` decide entre os três casos:

```java
case '/':
    if (peekNext() == '/') {
        skipLineComments();
        return nextToken();     // após pular, o próximo token de verdade vem daqui
    } else if (peekNext() == '*') {
        skipBlockComments();
        return nextToken();
    } else {
        advance();
        return new Token(TokenType.SLASH, "/");
    }
```

```java
private void skipLineComments() {
    while (peek() != '\n' && peek() != '\0') advance();
}

private void skipBlockComments() {
    advance(); advance();   // consome '/' e '*'
    while (true) {
        if (peek() == '\0') throw new Error("comentário de bloco não fechado");
        if (peek() == '*' && peekNext() == '/') {
            advance(); advance();
            return;
        }
        advance();
    }
}
```

Repare o padrão: `nextToken()` chama a si mesmo recursivamente depois de pular um comentário. É uma forma direta de expressar "esse token não conta, tente de novo" sem duplicar a lógica de reconhecimento.

## Contando linhas

Mensagens de erro úteis (para o capítulo 5, sobre erros sintáticos) precisam saber **em que linha** cada token começou. Adicionamos um contador `line`, incrementado em todo lugar que consumimos uma quebra de linha (`skipWhitespace`, `skipLineComments`, `skipBlockComments`), e passamos esse número para cada `Token` criado:

```java
private int line = 1;

public class Token {
    final TokenType type;
    final String lexeme;
    final int line;

    public Token(TokenType type, String lexeme, int line) {
        this.type = type;
        this.lexeme = lexeme;
        this.line = line;
    }
}
```

## Validando contra a saída de referência

O projeto Nand2Tetris fornece, para cada programa Jack de exemplo, um arquivo XML de referência com a sequência de tokens esperada — a forma mais direta de validar nosso tokenizador sem escrever manualmente centenas de casos de teste. O formato é simples: cada token vira uma tag com o nome da categoria léxica (`keyword`, `symbol`, `identifier`, `integerConstant`, `stringConstant`), com o lexema entre a abertura e o fechamento:

```
<tokens>
<keyword> if </keyword>
<symbol> ( </symbol>
<identifier> x </identifier>
<symbol> &lt; </symbol>
<integerConstant> 0 </integerConstant>
<symbol> ) </symbol>
...
</tokens>
```

Três detalhes de formatação importam para bater exatamente com a referência:

1. Todos os 19 símbolos (`+`, `-`, `.`, `<`, etc.) são agrupados sob a única tag `symbol` — a distinção fina entre `PLUS`, `MINUS`, `LT` etc. existe no nosso `TokenType`, mas não aparece no XML.
2. Os caracteres `<`, `>`, `"` e `&` são escapados como `&lt;`, `&gt;`, `&quot;` e `&amp;`, para não conflitar com a sintaxe do próprio XML.
3. Strings são impressas **sem** as aspas duplas que as delimitam.

Um `toString()` dedicado ao formato de teste cuida disso:

```java
public String toString() {
    String categoria = switch (type) {
        case NUMBER -> "integerConstant";
        case IDENT -> "identifier";
        case STRING -> "stringConstant";
        default -> isSymbol(type) ? "symbol" : "keyword";
    };
    String valor = lexeme;
    if (categoria.equals("symbol")) {
        valor = switch (valor) {
            case ">" -> "&gt;";
            case "<" -> "&lt;";
            case "\"" -> "&quot;";
            case "&" -> "&amp;";
            default -> valor;
        };
    }
    return "<" + categoria + "> " + valor + " </" + categoria + ">";
}
```

Com essa formatação, gerar a saída de teste é só percorrer todos os tokens:

```java
System.out.println("<tokens>");
for (Token tk = scan.nextToken(); tk.type != TokenType.EOF; tk = scan.nextToken()) {
    System.out.println(tk);
}
System.out.println("</tokens>");
```

Rodando contra um arquivo Jack real como `Main.jack` (do jogo Square do próprio Nand2Tetris) e comparando byte a byte com o `MainT.xml` de referência (os arquivos de teste "só tokens" terminam em `T.xml`, para diferenciar dos arquivos de árvore sintática completa que usaremos no capítulo 5), temos uma forma objetiva de saber se o tokenizador está correto — sem depender de inspeção visual.

## O que vem a seguir

Com um `JackTokenizer` completo e validado, o próximo capítulo constrói o analisador sintático: um método por não-terminal da gramática do capítulo 3, cada um consumindo tokens deste tokenizador e emitindo (inicialmente) uma árvore sintática em XML — o mesmo tipo de validação por comparação que usamos aqui, agora para a estrutura completa do programa.

!!! note "Atividade"
    Como exercício de fixação, tente escrever a especificação léxica de Jack inteira como uma única expressão regular por categoria de token (símbolos, inteiros, identificadores, strings) na linguagem que você escolher para o seu projeto. Depois de feito, reflita: quais as desvantagens de usar expressões regulares (ao invés de um scanner ad-hoc escrito à mão) para ler um arquivo inteiro e devolver a lista de tokens reconhecidos?
