# Atividade Prática: Gerador de Código Intermediário (VM)

**Disciplina:** Compiladores | **Unidade 2** | **Semestre:** {{SEMESTRE}} | **Entrega:** {{DATA_ENTREGA}}

---

## 📋 Descrição

Implementar o **gerador de código intermediário** para a linguagem **Jack**, traduzindo a
estrutura sintática reconhecida nas entregas anteriores para código da máquina virtual
(`.vm`), compatível com o VM Emulator oficial.

> ⚠️ **Requisito crítico:** o compilador **deve aceitar diretórios** como entrada. Ao receber
> um caminho de pasta, deve compilar **todos os arquivos `.jack`** contidos nela, gerando um
> `.vm` por classe. Sem isso, programas como `Square` e `Pong` (múltiplas classes) não podem
> ser compilados.

> **Importante:** esta atividade dá continuidade ao compilador. O gerador deve ser integrado
> ao parser já implementado, **reutilizando o mesmo repositório**.

📖 **Leitura de apoio:** capítulos 7 (especificação da VM Hack), 8 (tabela de símbolos),
9 (geração de código: expressões e comandos) e 10 (subrotinas e objetos).

---

## ✅ Requisitos

### 1. Leitura de diretórios e compilação em lote

O compilador deve aceitar:

- Um arquivo único: `compilador Main.jack`
- **Um diretório**: `compilador ./projects/11/Square/`

Ao receber um diretório: listar todos os `.jack`, compilar cada um individualmente e gerar
um `.vm` por classe (`Square.jack` → `Square.vm`).

### 2. Geração de código VM para as construções de Jack

- Expressões aritméticas, lógicas e relacionais: `+` `-` `*` `/` `&` `|` `<` `>` `=` (binários)
  e `-` `~` (unários)
- Lembre-se de que `*` e `/` **não são comandos VM**: viram chamadas a `Math.multiply` e
  `Math.divide`
- Comandos de controle: `if`, `while`, `do`, `return`
- Acesso e atribuição de variáveis: `static`, `field`, `arg` e `local`
- Chamadas de função, método e construtor (incluindo o tratamento de `this` e a alocação via
  `Memory.alloc`)
- Arrays: leitura e atribuição indexada (`arr[i]`), via `pointer 1` / segmento `that`
- Geração de **rótulos únicos** por subroutine para desvios e laços

### 3. Integração com a tabela de símbolos

- Resolver escopo e tipo de cada identificador pela tabela de símbolos
- Mapear corretamente categoria → segmento: `static`→`static`, `field`→`this`,
  `arg`→`argument`, `var`→`local`

### 4. Saída em arquivos `.vm`

Comandos em **minúsculas**, um por linha, com argumentos quando necessário:

```
push constant 0
pop local 1
call Math.abs 1
```

---

## 🧪 Estratégia de testes

### 🔹 Camada 1 — testes unitários

Testar isoladamente cada método do `VMWriter` (ou equivalente):

- `writePush(CONST, 5)` → `push constant 5`
- `writeArithmetic(ADD)` → `add`
- `writeCall("Math.abs", 1)` → `call Math.abs 1`

### 🔹 Camada 2 — testes de comportamento (Project 11)

Compile e execute os programas de referência **nesta ordem progressiva**:

| Programa | Arquivos | Foco de validação |
| --- | --- | --- |
| **Seven** | `Seven/Main.jack` | Expressões simples, `do`, `return` |
| **ConvertToBin** | `ConvertToBin/Main.jack` | Funções, laços, operações bit a bit |
| **Square** | `Square/*.jack` (3 arquivos) | Objetos, métodos, construtores, múltiplas classes |
| **Average** | `Average/Main.jack` | Arrays, strings, entrada/saída |
| **Pong** | `Pong/*.jack` (4 arquivos) | Projeto completo: física, I/O, várias classes |
| **ComplexArrays** 🏆 | `ComplexArrays/Main.jack` | Endereçamento de arrays aninhados (o caso mais hostil) |

> 💡 `ComplexArrays` é o teste que quebra implementações que erram o uso de `temp 0` na
> atribuição a array. Deixe-o por último, mas **não pule**: é ele que valida o caso
> `let a[a[i]] = a[b[a[b[j]]]]`.

### 🔹 Fluxo de validação

```bash
# 1. Compilar um diretório inteiro
compilador ./projects/11/Seven/

# 2. Conferir a geração do .vm
ls ./projects/11/Seven/Main.vm

# 3. Carregar no VM Emulator e executar

# 4. Validar o resultado esperado (Seven imprime 7)
```

---

## 📦 Entrega

1. **Repositório** (o mesmo das entregas anteriores)
   - Scanner + parser + tabela de símbolos + gerador de código **integrados**
   - Suporte a entrada por **arquivo e por diretório**
   - Histórico com pelo menos **{{MIN_COMMITS}} commits coerentes**
   - `README.md` com: integrantes, linguagem, instruções de build/execução (com exemplos de
     arquivo **e** de diretório), status da validação (quais programas do Project 11 foram
     testados e com que resultado) e relato dos desafios
2. **Vídeo de demonstração** *(opcional — remover se a oferta não exigir)*
   - Duração: até 10 minutos
   - Conteúdo: integrantes; visão geral do código; **compilação ao vivo de um diretório**;
     execução dos `.vm` gerados no VM Emulator; explicação de uma decisão técnica
3. **Na plataforma {{PLATAFORMA_ENTREGA}}**
   - `.zip` ou link do repositório (+ link do vídeo, se exigido)

---

## ⚠️ Observações

- **Mesmo repositório:** continue no repositório das entregas anteriores
- **Trabalho em grupo:** {{TAMANHO_GRUPO}}
- O compilador **precisa compilar pastas**, não apenas arquivos individuais
- **Teste incrementalmente:** valide `Seven` antes de tentar `Pong`
- Quando a saída não bate byte a byte com a referência oficial, ainda é possível validar
  **comportamentalmente**: execute ambas no VM Emulator e compare o resultado observável

---

## 🔗 Materiais de apoio

- Capítulos 7, 8, 9 e 10 do livro + Apêndice B (especificação da VM)
- Ferramentas e arquivos de teste oficiais: <https://www.nand2tetris.org/software>
- Página da turma: {{LINK_TURMA}}

---

## 🎯 Critérios de avaliação *(sugestão — calibrar por oferta)*

| Dimensão | O que será observado |
| --- | --- |
| **Funcionalidade** | Gera código VM correto para as construções exigidas? |
| **Compilação em lote** | Lê diretórios e compila múltiplos `.jack` automaticamente? |
| **Integração** | Scanner, parser, tabela de símbolos e gerador funcionam em conjunto? |
| **Validação prática** | Os programas do Project 11 compilam e executam corretamente? |
| **Qualidade técnica** | Código modular, legível, com testes e tratamento de erros? |
| **Documentação** | README claro, com instruções de uso e relato da implementação? |
| **Versionamento** | Commits refletem evolução gradual e organizada? |

**Penalidades:** código copiado = nota zero; atraso = -10% ao dia (máx. 3 dias);
compilador sem suporte a entrada por diretório = redução significativa.

---

> 🎉 Esta é a etapa que transforma análise sintática em execução real. Siga a progressão de
> testes e valide cada programa antes de avançar.
