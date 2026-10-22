# 2. Aquecimento: um simples tradutor

Antes de atacar a linguagem Jack por completo, vamos construir algo bem menor: um tradutor de expressões aritméticas simples para uma notação pós-fixada (tipo a usada pela VM Hack que veremos na Parte 2). O objetivo não é o tradutor em si — é praticar, em um problema pequeno, exatamente as técnicas que usaremos nos capítulos 4 e 5 para construir o tokenizador e o parser de Jack: análise léxica, análise sintática descendente recursiva e tradução dirigida pela sintaxe. Vamos construir esse tradutor com você, passo a passo, testando a cada etapa — é assim que todos os capítulos incrementais deste livro funcionam.

## Passo 1 — Só o parser, caractere a caractere

Nosso primeiro objetivo: traduzir expressões como `5 + 8 - 9` (dígitos únicos, sem espaços) de notação infixa para pós-fixa. Uma gramática ingênua para expressões,

```
expr -> expr + expr | expr - expr | digit
digit -> 0 | ... | 9
```

é **ambígua**: `5 + 8 - 9` admite duas árvores de derivação diferentes, dependendo de qual `expr` se expande primeiro. Reescrevemos para associar à esquerda:

```
expr -> expr + digit | expr - digit | digit
```

Isso resolve a ambiguidade, mas introduz **recursão à esquerda** — fatal para um analisador preditivo, que entraria em loop infinito tentando expandir `expr` antes de consumir qualquer entrada. Aplicando a transformação do capítulo 1 (`A → Aα | β` vira `A → βR`, `R → αR | ε`, com `A = expr`, `β = digit`, `α = + digit` ou `- digit`):

```
expr -> digit oper
oper -> + digit oper | - digit oper | ε
digit -> 0 | ... | 9
```

Essa gramática já é adequada para um parser descendente recursivo — um método Java por não-terminal. Como ainda não temos um analisador léxico, o `Parser` vai ler diretamente do array de bytes de entrada:

```java
public class Parser {
    private byte[] input;
    private int current;

    public Parser(byte[] input) {
        this.input = input;
    }

    public void parse() {
        expr();
    }

    private char peek() {
        if (current < input.length) return (char) input[current];
        return '\0';
    }

    private void match(char c) {
        if (c == peek()) {
            current++;
        } else {
            throw new Error("syntax error");
        }
    }

    void expr() {
        digit();
        oper();
    }

    void digit() {
        if (Character.isDigit(peek())) {
            System.out.println("push " + peek());
            match(peek());
        } else {
            throw new Error("syntax error");
        }
    }

    void oper() {
        if (peek() == '+') {
            match('+');
            digit();
            System.out.println("add");
            oper();
        } else if (peek() == '-') {
            match('-');
            digit();
            System.out.println("sub");
            oper();
        }
        // terceiro caso da produção: oper -> ε — não faz nada, apenas retorna
    }
}
```

Repare que cada não-terminal virou exatamente um método, e cada terminal virou uma chamada a `match`. As chamadas a `System.out.println` intercaladas entre elas são exatamente as ações semânticas do **esquema de tradução dirigido por sintaxe** do capítulo 1 — copiadas para dentro de cada método, na mesma posição em que apareceriam se escritas entre chaves na produção (`digit → dígito { print(dígito) }`, `oper → + digit { print('+') } oper`). É esse acoplamento entre "reconhecer a estrutura" e "emitir a tradução" — sem nenhuma árvore sintática explícita no meio do caminho — que vamos repetir em escala maior nos capítulos 9 e 10.

Rodando com `"8+5-7+9"`, o tradutor imprime `push 8`, `push 5`, `add`, `push 7`, `sub`, `push 9`, `add` — a expressão em notação pós-fixa, um comando por linha. Como exercício, fica a inclusão de `*` e `/`; e você vai notar que `45+89-876` já quebra, porque nosso "dígito" só aceita um caractere — é exatamente o problema que resolvemos no próximo passo.

## Passo 2 — Por que precisamos de um analisador léxico

Para reconhecer números com mais de um dígito (`number -> [0-9]+`), reconhecer identificadores, ou ignorar espaços em branco, precisamos de um componente dedicado a agrupar caracteres em palavras: o **analisador léxico** (scanner). Segundo Cooper, sua tarefa é transformar um fluxo de caracteres em um fluxo de palavras classificadas em categorias sintáticas — e ele é o único estágio do compilador que toca em cada caractere individual do programa-fonte. O scanner é subordinado ao parser: sempre que o parser precisa de um novo token, o scanner o produz sob demanda (nunca tokeniza tudo de uma vez — se houver um erro léxico ou sintático no meio do caminho, o restante pode nem precisar ser lido).

Scanners reais seguem tipicamente uma de três abordagens: **controlados por tabela** (um esqueleto genérico + tabelas que codificam a linguagem), **codificados diretamente** (o autômato vira uma cadeia de `if`/`switch`, como vimos no capítulo 1), ou **ad-hoc** — a abordagem informal (sem simular estados explicitamente) usada por boa parte dos compiladores de código aberto reais (o scanner do Go, do Rust, do TypeScript). Vamos adotar a abordagem ad-hoc: é a mais direta de escrever à mão e a que usaremos também no `JackTokenizer` do capítulo 4.

## Passo 3 — Extraindo o Scanner

Primeiro, uma refatoração pura: movemos a responsabilidade de ler caracteres do `Parser` para uma nova classe `Scanner`, mantendo por enquanto o "token" como um único caractere:

```java
public class Scanner {
    private byte[] input;
    private int current;

    public Scanner(byte[] input) {
        this.input = input;
    }

    private char peek() {
        if (current < input.length) return (char) input[current];
        return '\0';
    }

    private void advance() {
        if (peek() != '\0') current++;
    }

    public char nextToken() {
        char ch = peek();
        if (Character.isDigit(ch)) {
            advance();
            return ch;
        }
        switch (ch) {
            case '+':
            case '-':
                advance();
                return ch;
        }
        return '\0';
    }
}
```

O `Parser` agora delega ao `Scanner` e guarda apenas o **token corrente**:

```java
public class Parser {
    private Scanner scan;
    private char currentToken;

    public Parser(byte[] input) {
        scan = new Scanner(input);
        currentToken = scan.nextToken();
    }

    private void nextToken() {
        currentToken = scan.nextToken();
    }

    private void match(char t) {
        if (currentToken == t) {
            nextToken();
        } else {
            throw new Error("syntax error");
        }
    }

    void digit() {
        if (Character.isDigit(currentToken)) {
            System.out.println("push " + currentToken);
            match(currentToken);
        } else {
            throw new Error("syntax error");
        }
    }
    // oper() e expr() seguem o mesmo padrão, só trocando peek() por currentToken
}
```

Rodando de novo com `"8+5-7+9"`, o resultado é idêntico ao do passo 1 — a refatoração não mudou o comportamento, só a responsabilidade de cada classe. Esse é o padrão que vamos repetir sempre: refatorar com uma rede de segurança (o teste de antes e depois), não misturar "mudar comportamento" com "reorganizar responsabilidade".

## Passo 4 — Tokens de verdade: tipo + lexema

Um token não é só um caractere: é um **tipo** (categoria sintática) e um **lexema** (o texto reconhecido). Introduzimos:

```java
public enum TokenType { PLUS, MINUS, NUMBER, EOF }

public class Token {
    final TokenType type;
    final String lexeme;

    public Token(TokenType type, String lexeme) {
        this.type = type;
        this.lexeme = lexeme;
    }

    public String toString() {
        return "<" + type + ">" + lexeme + "</" + type + ">";
    }
}
```

E o `Scanner` passa a reconhecer números de múltiplos dígitos, aplicando exatamente a receita "diagrama de transição → código" do capítulo 1 — uma função dedicada consome dígitos enquanto houver:

```java
private Token number() {
    int start = current;
    while (Character.isDigit(peek())) advance();
    String n = new String(input, start, current - start);
    return new Token(TokenType.NUMBER, n);
}

public Token nextToken() {
    char ch = peek();
    if (Character.isDigit(ch)) return number();
    switch (ch) {
        case '+': advance(); return new Token(TokenType.PLUS, "+");
        case '-': advance(); return new Token(TokenType.MINUS, "-");
        case '\0': return new Token(TokenType.EOF, "EOF");
        default: throw new Error("lexical error at " + ch);
    }
}
```

Testando com `"289-85+0+69"`, o scanner agora devolve `<NUMBER>289</NUMBER>`, `<MINUS>-</MINUS>`, `<NUMBER>85</NUMBER>`, e assim por diante — cada número, não importa quantos dígitos, vira um único token. (Repare que `number()` aceita zeros à esquerda: `"007"` vira um único token `NUMBER` com lexema `007`. A expressão regular de inteiro sem sinal do capítulo 1, `0 | [1...9] [0...9]*`, não aceitaria esse lexema — para o nosso tradutor, e também para Jack, que só pede "um número decimal", essa folga não causa problema.)

O `Parser` é atualizado para comparar `currentToken.type` (não mais o caractere bruto), e a gramática ganha o não-terminal `number`:

```
expr -> number oper
oper -> + number oper | - number oper | ε
number -> [0-9]+
```

## Passo 5 — Espaços em branco

Ao testar `"89 +508 -7+99"` (com espaços), o scanner lança erro léxico no primeiro espaço. Espaços em branco são, como vimos no capítulo 1, um token que tipicamente **não é devolvido** ao parser — o scanner apenas os consome e segue em frente:

```java
private void skipWhitespace() {
    char ch = peek();
    while (ch == ' ' || ch == '\r' || ch == '\t' || ch == '\n') {
        advance();
        ch = peek();
    }
}
```

Chamado no início de `nextToken()`, isso resolve o problema — e de quebra nos dá tolerância a formatação livre, uma propriedade que vamos querer manter em Jack.

## Passo 6 — Identificadores e a primeira palavra reservada

Adicionamos `IDENT` ao `TokenType` e duas funções auxiliares (`isAlpha`, `isAlphaNumeric`) que definem o alfabeto de um identificador — a expressão regular de identificadores do capítulo 1, agora aceitando também `_` na primeira posição, como Jack exige — `([A-Za-z_])([A-Za-z0-9_])*` —, traduzida para código:

```java
private Token identifier() {
    int start = current;
    while (isAlphaNumeric(peek())) advance();
    String id = new String(input, start, current - start);
    return new Token(TokenType.IDENT, id);
}
```

A gramática ganha um não-terminal `term` que escolhe entre número e identificador:

```
expr -> term oper
oper -> + term oper | - term oper | ε
term -> number | identifier
```

Agora, para reconhecer `let a = 42 + 5;`, precisamos que `let` seja tratado como **palavra reservada**, não como identificador comum — mas léxica e sintaticamente `let` é indistinguível de qualquer outro identificador. A solução, como discutido no capítulo 1, é resolver isso via **tabela**: uma coleção de palavras-chave é consultada depois que o lexema já foi reconhecido como identificador; se o lexema estiver na tabela, o token é reclassificado:

```java
private static final Map<String, TokenType> keywords = new HashMap<>();
static { keywords.put("let", TokenType.LET); }

private Token identifier() {
    int start = current;
    while (isAlphaNumeric(peek())) advance();
    String id = new String(input, start, current - start);
    TokenType type = keywords.getOrDefault(id, TokenType.IDENT);
    return new Token(type, id);
}
```

Com `=` e `;` também reconhecidos como tokens de um único caractere, o scanner já tokeniza `let a = 42 + 5;` corretamente. A gramática ganha o comando de atribuição:

```
letStatement -> 'let' identifier '=' expression ';'
```

que se traduz diretamente em:

```java
void letStatement() {
    match(TokenType.LET);
    String id = currentToken.lexeme;
    match(TokenType.IDENT);
    match(TokenType.EQ);
    expr();
    System.out.println("pop " + id);
    match(TokenType.SEMICOLON);
}
```

A ordem de emissão é reveladora: primeiro traduzimos a expressão à direita do `=` (que empilha seu resultado), e só **depois** emitimos o `pop` — a atribuição, em uma máquina de pilha, é sempre "calcule o valor, depois desempilhe para a variável".

## Passo 7 — Comandos e o programa como uma lista de comandos

Adicionamos um segundo comando, `print`, seguindo exatamente o mesmo padrão de `let`:

```
printStatement -> 'print' expression ';'
statement -> printStatement | letStatement
statements -> statement*
```

```java
void statement() {
    if (currentToken.type == TokenType.PRINT) printStatement();
    else if (currentToken.type == TokenType.LET) letStatement();
    else throw new Error("syntax error");
}

void statements() {
    while (currentToken.type != TokenType.EOF) {
        statement();
    }
}
```

e `parse()` passa a chamar `statements()` em vez de `expr()` diretamente — nosso símbolo sentencial agora é "um programa é uma sequência de comandos". Um programa com três linhas (`let a = ...; let b = ...; print a + b + 6;`) já produz uma sequência completa de instruções em notação pós-fixa, pronta para ser executada por uma máquina de pilha.

## Passo 8 — Um pequeno interpretador para a saída

Para fechar o ciclo — ver o programa de fato *rodando*, não só traduzido — um `Interpretador` minúsculo consome as linhas de saída do parser (`push`, `pop`, `add`, `sub`, `print`) e as executa sobre uma pilha e um mapa de variáveis:

```java
switch (command.type) {
    case ADD: { int b = stack.pop(), a = stack.pop(); stack.push(a + b); break; }
    case SUB: { int b = stack.pop(), a = stack.pop(); stack.push(a - b); break; }
    case PUSH: {
        Integer v = variables.get(command.arg);
        stack.push(v != null ? v : Integer.parseInt(command.arg));
        break;
    }
    case POP: variables.put(command.arg, stack.pop()); break;
    case PRINT: System.out.println(stack.pop()); break;
}
```

Esse pequeno interpretador não é um capítulo à parte do nosso livro — é, na verdade, uma prévia da **VM Hack** que construiremos na Parte 2: uma máquina de pilha, comandos aritméticos, `push`/`pop` sobre variáveis nomeadas. O paralelo não é coincidência: é exatamente o tipo de representação intermediária que compiladores reais usam para separar "traduzir a sintaxe" de "executar (ou gerar código para) o resultado".

## O que levamos daqui

Neste tradutor de aquecimento, cobrimos o ciclo completo que vamos repetir, em escala maior, nos capítulos 4 e 5: eliminar ambiguidade e recursão à esquerda na gramática, construir um scanner ad-hoc que classifica lexemas em tokens tipados, e construir um parser descendente recursivo — um método por não-terminal — que traduz a estrutura reconhecida diretamente em código de saída, sem passar por uma árvore sintática intermediária. A diferença, dali em diante, é de tamanho e de rigor: a linguagem Jack tem uma gramática bem maior, exige uma especificação léxica completa e formal, e o "código de saída" será a VM Hack de verdade, não um punhado de instruções ad-hoc.

!!! note "Nota histórica: geradores automáticos e ANTLR"
    Compiladores de produção raramente escrevem scanners e parsers inteiramente à mão — ferramentas como Flex (scanners) e ANTLR (scanners + parsers, a partir de uma gramática declarativa) automatizam boa parte desse trabalho. Uma versão anterior deste material explorou o uso do ANTLR para gerar o mesmo tradutor a partir de uma gramática `.g4`; vale a pena estudar por curiosidade, mas neste livro escrevemos os analisadores à mão de propósito — é a única forma de realmente entender o que esses geradores fazem por baixo dos panos.
