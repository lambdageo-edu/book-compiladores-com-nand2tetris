# Atividade Prática: Implementação do VMTranslator

**Disciplina:** Compiladores | **Unidade 3** | **Semestre:** {{SEMESTRE}} | **Entrega:** {{DATA_ENTREGA}}

> **Objetivo:** desenvolver o tradutor VM → Assembly Hack, cobrindo acesso a dados,
> operações aritméticas/lógicas, controle de fluxo e o protocolo completo de chamada de
> função.

📖 **Leitura de apoio:** capítulos 7 (especificação da VM), 11 (especificação do Assembly Hack),
12 (VM Translator parte 1) e 13 (VM Translator parte 2) + Apêndices B e C.

> 📌 **Este enunciado cobre as duas partes.** Se a sua oferta dividir em duas entregas,
> corte na marca `--- PARTE 2 ---` mais abaixo. A Parte 1 corresponde ao **Project 7** e ao
> capítulo 12; a Parte 2, ao **Project 8** e ao capítulo 13.

---

## 👥 Equipe e linguagem

- ✅ **Mudança de dupla permitida** em relação ao projeto anterior (compilador Jack)
- ✅ **Mudança de linguagem permitida** — sinta-se à vontade para explorar novas ferramentas
- 📝 No `README.md`, registre: integrantes, linguagem e versão, instruções de build e
  execução, e (opcional) justificativa da escolha da linguagem

---

## 🗂️ Configuração do repositório

1. Crie um **novo repositório** chamado `vmtranslator`
2. Estruture o projeto seguindo boas práticas da linguagem escolhida (separação de módulos:
   `parser/`, `codewriter/`, `main/`)
3. Inclua um `.gitignore` adequado à linguagem
4. **Histórico de commits é critério de avaliação** — commits atômicos e descritivos:

```
feat(parser): implementa tokenizacao e commandType
fix(codewriter): corrige calculo de endereco para o segmento 'that'
docs(readme): adiciona instrucoes de execucao
```

---

## ⚙️ PARTE 1 — Acesso a dados e operações aritméticas/lógicas

### 1️⃣ Parser — analisador de arquivos `.vm`

Lê, filtra comentários e tokeniza comandos VM.

| Método | Descrição |
| --- | --- |
| `newParser(arquivo)` | Abre o arquivo e prepara a leitura |
| `hasMoreCommands()` | Indica se há comandos pendentes |
| `advance()` | Avança para o próximo comando |
| `commandType()` | `C_ARITHMETIC`, `C_PUSH`, `C_POP` |
| `arg1()` | Primeiro argumento (ex: `local`, `add`) |
| `arg2()` | Índice (apenas para `push`/`pop`) |

**Exemplo simplificado (Python):**

```python
class Parser:
    def __init__(self, filename):
        with open(filename) as f:
            self.lines = [
                line.split('//')[0].split()
                for line in f
                if line.split('//')[0].strip()
            ]
        self.index = 0

    def has_more_commands(self):
        return self.index < len(self.lines)

    def advance(self):
        cmd = self.lines[self.index]
        self.index += 1
        return cmd
```

### 2️⃣ CodeWriter — gerador de Assembly Hack

| Método | Descrição |
| --- | --- |
| `newCodeWriter(arquivo)` | Abre o `.asm` para escrita |
| `writeArithmetic(cmd)` | `add`, `sub`, `neg`, `eq`, `gt`, `lt`, `and`, `or`, `not` |
| `writePush(segmento, índice)` | Empilha valores |
| `writePop(segmento, índice)` | Desempilha valores |
| `close()` | Finaliza e fecha o arquivo |

**Segmentos de memória:**

| Segmento | Base | Observação |
| --- | --- | --- |
| `constant` | — | Pseudo-segmento; o valor **é** o índice, não há leitura de memória |
| `local` | `LCL` (RAM[1]) | Endereço calculado: `RAM[LCL + i]` |
| `argument` | `ARG` (RAM[2]) | `RAM[ARG + i]` |
| `this` | `THIS` (RAM[3]) | `RAM[THIS + i]` |
| `that` | `THAT` (RAM[4]) | `RAM[THAT + i]` |
| `temp` | RAM[5]–RAM[12] | Endereço fixo: `RAM[5 + i]` |
| `pointer` | RAM[3]–RAM[4] | `pointer 0` = `THIS`, `pointer 1` = `THAT` |
| `static` | RAM[16]+ | Rótulo simbólico por arquivo (`NomeArquivo.i`) |

> 💡 `R13`–`R15` estão reservados como área de rascunho — você vai precisar deles no `pop`
> de segmento dinâmico e no `return`.

### 🧪 Testes da Parte 1 (Project 07)

```
projects/07/
├── StackArithmetic/
│   ├── SimpleAdd/     (SimpleAdd.vm, .tst, .cmp)
│   └── StackTest/
└── MemoryAccess/
    ├── BasicTest/
    ├── PointerTest/
    └── StaticTest/
```

**Fluxo:** traduza o `.vm` → abra o CPU Emulator → carregue o `.tst` → execute → compare a
saída com o `.cmp`. Sucesso é a mensagem `Comparison ended successfully.`

---

## --- PARTE 2 ---

## ⚙️ PARTE 2 — Controle de fluxo e funções

### 3️⃣ Comandos adicionais

| Comando | O que implementar |
| --- | --- |
| `label`, `goto`, `if-goto` | Rótulos **qualificados por função** (`funcao$rotulo`) para evitar colisão |
| `function nome nLocais` | Ponto de entrada + inicialização das `nLocais` variáveis em 0 |
| `call nome nArgs` | Salvar endereço de retorno + `LCL`/`ARG`/`THIS`/`THAT`; reposicionar `ARG` e `LCL`; saltar |
| `return` | Restaurar o frame do chamador na **ordem reversa** e saltar de volta |

### 4️⃣ Bootstrap

Emitido **uma única vez**, no início do `.asm` final: inicializar `SP = 256` e executar
`call Sys.init 0`.

### 5️⃣ Entrada por diretório

Ao receber uma pasta, traduzir **todos** os `.vm` para um **único** `.asm`, com o bootstrap
no topo.

### 🧪 Testes da Parte 2 (Project 08)

| Programa | Foco |
| --- | --- |
| `BasicLoop`, `FibonacciSeries` | `label`, `goto`, `if-goto` |
| `SimpleFunction` | `function` / `return` básicos |
| `NestedCall` | Preservação correta de todos os registradores do frame |
| `FibonacciElement`, `StaticsTest` | Múltiplos arquivos + bootstrap + recursão |

> 🏆 `NestedCall` é o teste que revela erros sutis no protocolo de `call`/`return` — se ele
> passa, seu tradutor provavelmente está correto.

---

## 📦 Entregáveis

- [ ] Repositório `vmtranslator` criado e acessível
- [ ] Código-fonte organizado por módulos
- [ ] `README.md` completo (integrantes, linguagem e versão, build/execução, exemplo de uso)
- [ ] Mínimo de **{{MIN_COMMITS}} commits significativos**
- [ ] Tradução funcional dos testes do Project 07 (Parte 1) e Project 08 (Parte 2)

---

## 🎯 Critérios de avaliação *(sugestão — calibrar por oferta)*

| Critério | Peso | Detalhes |
| --- | --- | --- |
| **Correção funcional** | 40% | O `.asm` gerado passa nos scripts `.tst`/`.cmp` oficiais |
| **Qualidade do código** | 25% | Legibilidade, coesão, uso de padrões da linguagem |
| **Documentação** | 15% | README claro, API bem definida |
| **Versionamento** | 10% | Commits atômicos, evolução lógica |
| **Testes e validação** | 10% | Capacidade de executar e passar nos scripts fornecidos |

---

## 📚 Recursos adicionais

- Capítulos 7, 11, 12 e 13 do livro + Apêndices B e C
- *The Elements of Computing Systems*, caps. 7 e 8 — <https://www.nand2tetris.org/book>
- Ferramentas e arquivos de teste oficiais: <https://www.nand2tetris.org/software>
- Página da turma: {{LINK_TURMA}}

---

> 💡 **Implementação incremental:**
> 1. Comece com `push constant` → valide com `SimpleAdd`
> 2. Adicione `add`/`sub` → teste novamente
> 3. Expanda para os demais segmentos (`local`, `argument`, `this`, `that`, `temp`, `pointer`, `static`)
> 4. Implemente as comparações (`eq`, `gt`, `lt`) — exigem rótulos únicos
> 5. Só então avance para controle de fluxo e funções (Parte 2)
>
> Teste **cada comando isoladamente** antes de integrar. Isso reduz drasticamente o tempo de
> depuração.
