# Apêndice A — Gramática da linguagem Jack

Referência completa, para consulta rápida durante os capítulos 4 e 5. A notação segue a convenção do Nand2Tetris: `'x'` símbolo terminal literal; `x` (sem aspas) não-terminal; `()` agrupamento; `x|y` alternativa; `x?` zero ou uma ocorrência; `x*` zero ou mais ocorrências.

## Elementos léxicos

| Terminal | Regra |
|---|---|
| `keyword` | `class` \| `constructor` \| `function` \| `method` \| `field` \| `static` \| `var` \| `int` \| `char` \| `boolean` \| `void` \| `true` \| `false` \| `null` \| `this` \| `let` \| `do` \| `if` \| `else` \| `while` \| `return` |
| `symbol` | `{` `}` `(` `)` `[` `]` `.` `,` `;` `+` `-` `*` `/` `&` `|` `<` `>` `=` `~` |
| `integerConstant` | Um número decimal |
| `stringConstant` | Sequência de caracteres Unicode, sem aspas duplas nem quebra de linha |
| `identifier` | Sequência de letras, dígitos e `_`, não iniciando com dígito |

## Estrutura de classe

| Não-terminal | Regra |
|---|---|
| `class` | `'class'` className `'{'` classVarDec\* subroutineDec\* `'}'` |
| `classVarDec` | (`'static'`\|`'field'`) type varName (`','` varName)* `';'` |
| `type` | `'int'` \| `'char'` \| `'boolean'` \| className |
| `subroutineDec` | (`'constructor'`\|`'function'`\|`'method'`) (`'void'`\|type) subroutineName `'('` parameterList `')'` subroutineBody |
| `parameterList` | ((type varName) (`','` type varName)*)? |
| `subroutineBody` | `'{'` varDec* statements `'}'` |
| `varDec` | `'var'` type varName (`','` varName)* `';'` |
| `className` | identifier |
| `subroutineName` | identifier |
| `varName` | identifier |

## Comandos (statements)

| Não-terminal | Regra |
|---|---|
| `statements` | statement* |
| `statement` | letStatement \| ifStatement \| whileStatement \| doStatement \| returnStatement |
| `letStatement` | `'let'` varName (`'['` expression `']'`)? `'='` expression `';'` |
| `ifStatement` | `'if'` `'('` expression `')'` `'{'` statements `'}'` (`'else'` `'{'` statements `'}'`)? |
| `whileStatement` | `'while'` `'('` expression `')'` `'{'` statements `'}'` |
| `doStatement` | `'do'` subroutineCall `';'` |
| `returnStatement` | `'return'` expression? `';'` |

## Expressões

| Não-terminal | Regra |
|---|---|
| `expression` | term (op term)* |
| `term` | integerConstant \| stringConstant \| keywordConstant \| varName \| varName `'['` expression `']'` \| subroutineCall \| `'('` expression `')'` \| unaryOp term |
| `subroutineCall` | subroutineName `'('` expressionList `')'` \| (className\|varName) `'.'` subroutineName `'('` expressionList `')'` |
| `expressionList` | (expression (`','` expression)*)? |
| `op` | `+` \| `-` \| `*` \| `/` \| `&` \| `|` \| `<` \| `>` \| `=` |
| `unaryOp` | `-` \| `~` |
| `keywordConstant` | `true` \| `false` \| `null` \| `this` |

## Notas de implementação

- `term` é o único não-terminal que exige mais de um token de *lookahead* para decidir a produção: ao ver um `identifier`, o parser precisa olhar o token seguinte (`[`, `(`, `.`, ou nenhum dos três) para diferenciar variável simples, acesso a array, e chamada de subroutine (qualificada ou não). Ver capítulo 5.
- A ausência de precedência de operadores em Jack (capítulo 3) é o que permite `expression` ser `term (op term)*` — um único nível, sem a cadeia `expression → term → factor` que uma gramática com precedência exigiria.
- Esta gramática já é LL(1): não há recursão à esquerda nem produções alternativas com o mesmo símbolo inicial dentro do mesmo não-terminal, exceto o *lookahead* extra de `term` descrito acima.
