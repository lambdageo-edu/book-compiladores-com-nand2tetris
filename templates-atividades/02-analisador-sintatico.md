# Atividade Prática: Analisador Sintático da Linguagem Jack

**Disciplina:** Compiladores | **Unidade 1 (fechamento)** | **Semestre:** {{SEMESTRE}} | **Entrega:** {{DATA_ENTREGA}}

---

## 📋 Descrição

Implementar um analisador sintático (parser) para a linguagem **Jack**, capaz de consumir a
saída do analisador léxico desenvolvido na entrega anterior, verificar a conformidade com a
gramática da linguagem e gerar um `.xml` representando a árvore sintática.

> **Importante:** esta atividade **encerra a Unidade 1**. O parser deve ser integrado ao
> scanner já implementado, **reutilizando o mesmo repositório**. A implementação deve seguir
> a abordagem de *recursive descent parsing* apresentada em aula.

📖 **Leitura de apoio:** capítulos 3 (gramática de Jack), 5 (analisador sintático) e
Apêndice A (gramática completa).

---

## ✅ Requisitos

### 1. Implementação da gramática Jack

Suporte completo às regras da linguagem:

- Estrutura de classe: `class`, `classVarDec`, `subroutineDec`, `parameterList`, `subroutineBody`, `varDec`
- Comandos: `let`, `if`, `while`, `do`, `return`
- Expressões: `expression`, `term`, `op`, `unaryOp`, `keywordConstant`, `subroutineCall`, `expressionList`

Método por não-terminal (*recursive descent*), traduzindo cada produção diretamente em código.

> 💡 **Note que Jack não tem precedência de operadores.** A avaliação é sempre da esquerda
> para a direita, e é isso que permite `expression → term (op term)*` — um único nível, sem
> a cadeia `expression → term → factor` que uma linguagem com precedência exigiria. Não
> tente implementar tabela de precedência: seria incorreto em relação à especificação.

### 2. Consumo de tokens

- Reutilizar o analisador léxico da entrega anterior
- Implementar auxiliares equivalentes a `advance()`, `peek()`, `match(tipo)`, `expect(tipo)`
- Manter **um token de antecipação** (*lookahead*): `term` (e `subroutineCall`) é o único ponto
  em que é preciso olhar o token seguinte ao identificador para desambiguar variável simples ×
  acesso a array × chamada de subroutine — o único trecho LL(2) da gramática, que se torna
  LL(1) ao consumir o identificador antes de decidir
- Detecção de tokens inesperados com mensagem clara indicando a **linha**

### 3. Saída XML

Cada não-terminal aparece como tag, envolvendo seus filhos:

```xml
<class>
  <keyword> class </keyword>
  <identifier> Square </identifier>
  <symbol> { </symbol>
  ...
</class>
```

Mesmas regras de escape da entrega anterior. O nome do arquivo de saída é livre — documente
no `README.md`.

### 4. Validação

A saída deve ser **estruturalmente equivalente** aos arquivos de árvore sintática oficiais
do Project 10 (`Main.xml`, `Square.xml`, `SquareGame.xml` — sem o sufixo `T`, que identifica
os arquivos de *tokens* da entrega anterior). Diferenças apenas de espaçamento/indentação
são aceitáveis; a hierarquia de tags deve ser idêntica.

> 💡 O pacote de testes inclui `ExpressionLessSquare/`, em que expressões complexas são
> substituídas por placeholders simples. Use-o para validar a "casca" da gramática (classes,
> subroutines, comandos) **antes** de enfrentar a gramática de expressões.

---

## 🧪 Estratégias de teste

| Abordagem | Quando usar |
| --- | --- |
| `diff` textual direto | Testes rápidos e manuais |
| `diff -w` (ignora espaços) | Quando a indentação varia |
| Framework de testes (JUnit, pytest…) ⭐ | **Recomendado para a entrega** |
| Normalização prévia do XML | Para evitar falsos negativos por indentação |

**Exemplo em Python (pytest):**

```python
def normalize_xml(content):
    return '\n'.join(line.strip() for line in content.splitlines() if line.strip())

def test_parser_output():
    esperado = normalize_xml(open('expected/Square.xml').read())
    gerado = normalize_xml(open('output/Square.xml').read())
    assert esperado == gerado
```

**Exemplo em Java (JUnit):**

```java
private String normalizeXml(String content) {
    return content.lines()
                  .map(String::strip)
                  .filter(l -> !l.isEmpty())
                  .collect(Collectors.joining("\n"));
}

@Test
public void testSquareParser() throws IOException {
    String esperado = normalizeXml(Files.readString(Path.of("expected/Square.xml")));
    String gerado = normalizeXml(Files.readString(Path.of("output/Square.xml")));
    assertEquals(esperado, gerado);
}
```

---

## 📦 Entrega

1. **Repositório** (o mesmo criado para o scanner)
   - Scanner + parser **integrados**
   - Histórico com pelo menos **{{MIN_COMMITS}} commits coerentes**
   - `README.md` com: integrantes, linguagem, build/execução, exemplo de uso, status da
     validação contra os arquivos oficiais, nome do arquivo XML de saída e breve relato dos
     desafios da unidade
2. **Vídeo de demonstração** *(opcional — remover se a oferta não exigir)*
   - Duração: até 5 minutos
   - Conteúdo: apresentação dos integrantes; execução do pipeline scanner + parser sobre um
     `.jack`; comparação da saída com o arquivo oficial; explicação de uma decisão técnica
   - Hospedagem: YouTube (não listado) ou Drive com link público
3. **Na plataforma {{PLATAFORMA_ENTREGA}}**
   - `.zip` com o código + link do repositório (+ link do vídeo, se exigido)

---

## ⚠️ Observações

- **Mesma linguagem do scanner** é fortemente recomendada, para facilitar a integração
- **Mesmo repositório:** não crie um novo
- **Trabalho em grupo:** {{TAMANHO_GRUPO}}
- Este repositório continua sendo usado na Unidade 2 (gerador de código VM)
- **Dica:** implemente e teste na ordem do capítulo 5 — primeiro `term` simples, depois
  `expression`, depois cada comando isoladamente, e só então `parseStatements`,
  `parseSubroutineDec` e `parseClass`

---

## 🔗 Materiais de apoio

- Capítulos 3 e 5 do livro + Apêndice A (gramática completa de Jack)
- Ferramentas e arquivos de teste oficiais: <https://www.nand2tetris.org/software>
- Página da turma: {{LINK_TURMA}}

---

## 🎯 Critérios de avaliação *(sugestão — calibrar por oferta)*

| Item | Peso |
| --- | --- |
| Funcionalidade (gramática Jack completa e correta) | 35% |
| Integração scanner + parser | 25% |
| Validação XML (hierarquia equivalente à oficial) | 20% |
| Qualidade do código e organização modular | 10% |
| Histórico de commits | 5% |
| Documentação (README + relato da unidade) | 5% |

**Penalidades:** código copiado = nota zero; atraso = -10% ao dia (máx. 3 dias);
parser que não integra com o scanner = -20%.

---

> 🎉 Esta entrega finaliza a Unidade 1. Na próxima, avançamos para a tabela de símbolos e a
> geração de código VM.
