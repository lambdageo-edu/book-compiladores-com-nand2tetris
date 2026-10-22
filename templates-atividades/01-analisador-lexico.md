# Atividade Prática: Analisador Léxico da Linguagem Jack

**Disciplina:** Compiladores | **Unidade 1** | **Semestre:** {{SEMESTRE}} | **Entrega:** {{DATA_ENTREGA}}

---

## 📋 Descrição

Implementar um analisador léxico (scanner) para a linguagem **Jack** (projeto nand2tetris),
capaz de ler arquivos fonte `.jack`, identificar seus tokens e gerar um arquivo de saída
`.xml` compatível com os arquivos de referência oficiais do curso.

> **Importante:** os capítulos do livro servem como guia conceitual. A implementação é
> livre e deve ser desenvolvida pelo grupo.

📖 **Leitura de apoio:** capítulos 1 (linguagens e gramáticas), 2 (tradutor de aquecimento),
3 (a linguagem Jack) e 4 (analisador léxico).

---

## ✅ Requisitos

### 1. Reconhecimento das cinco categorias léxicas

| Categoria | O que reconhecer |
| --- | --- |
| `keyword` | As 21 palavras reservadas de Jack |
| `symbol` | `{` `}` `(` `)` `[` `]` `.` `,` `;` `+` `-` `*` `/` `&` `\|` `<` `>` `=` `~` (19 símbolos) |
| `integerConstant` | Números inteiros decimais de 0 a 32767 |
| `stringConstant` | Cadeias entre aspas duplas (sem escapes) |
| `identifier` | Letras, dígitos e `_`, não iniciando com dígito |

> ⚠️ Atenção aos colchetes `[` e `]`: eles fazem parte do conjunto de símbolos (são usados
> em acesso a array) e é comum esquecê-los na primeira implementação.

### 2. Tratamento de ruído

- Ignorar espaços em branco, tabs e quebras de linha
- Ignorar comentários de linha (`//`), de bloco (`/* */`) e de documentação (`/** */`)
- Reconhecer que `/` é ambíguo: pode ser divisão ou início de comentário — resolve-se com
  um caractere de *lookahead*

### 3. Saída XML

- Documento envolto por `<tokens>` e `</tokens>`
- Tags com espaços internos: `<keyword> class </keyword>`
- Todos os 19 símbolos usam a mesma tag `symbol` (a distinção fina fica no seu `TokenType`,
  não na saída)
- Caracteres escapados: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`
- Strings impressas **sem** as aspas delimitadoras
- Sem token `EOF` na saída

### 4. Validação

A saída do seu scanner deve ser **idêntica** aos arquivos `*T.xml` oficiais do Project 10:

- `Main.jack` → `MainT.xml`
- `Square.jack` → `SquareT.xml`
- `SquareGame.jack` → `SquareGameT.xml`

---

## 📦 Entrega

1. **Repositório no GitHub/GitLab**
   - Código fonte do projeto
   - Histórico com pelo menos **{{MIN_COMMITS}} commits coerentes** (evolução gradual)
   - `README.md` com: integrantes, linguagem utilizada, instruções de build/execução e
     instruções para rodar os testes
2. **Na plataforma {{PLATAFORMA_ENTREGA}}**
   - Enviar `.zip` com o código do projeto
   - Inserir o link do repositório nos comentários da entrega

---

## ⚠️ Observações

- **Linguagem livre:** escolha da equipe (Python, Java, C, C++, Go, Rust, etc.)
- **Trabalho em grupo:** {{TAMANHO_GRUPO}}
- **Não copie código** de repositórios de referência — use apenas como base conceitual
- Este repositório **será reutilizado** nas próximas entregas (parser, gerador de código):
  organize-o com isso em mente
- Saída com formatação XML incorreta terá penalidade na avaliação

---

## 🔗 Materiais de apoio

- Capítulos 1–4 do livro da disciplina
- Ferramentas e arquivos de teste oficiais: <https://www.nand2tetris.org/software>
- IDE web oficial (sem instalação): <https://nand2tetris.github.io/web-ide/>
- Página da turma: {{LINK_TURMA}}

---

## 🎯 Critérios de avaliação *(sugestão — calibrar por oferta)*

| Item | Peso |
| --- | --- |
| Funcionalidade (tokens reconhecidos corretamente) | 40% |
| Validação XML (saída idêntica aos arquivos oficiais) | 30% |
| Qualidade do código e organização | 15% |
| Histórico de commits | 10% |
| Documentação (README claro) | 5% |

**Penalidades:** código copiado = nota zero; atraso = -10% ao dia (máx. 3 dias).

---

> ❓ Dúvidas? Utilize o fórum da disciplina ou os horários de atendimento.
