# Atividade Prática: Implementação do Assembler Hack

**Disciplina:** Compiladores | **Unidade 3** | **Semestre:** {{SEMESTRE}} | **Entrega:** {{DATA_ENTREGA}}

> **Objetivo:** desenvolver um **montador** que traduz programas em Assembly Hack (`.asm`)
> para código de máquina binário (`.hack`), executável pelo CPU Emulator.

📖 **Leitura de apoio:** capítulos 11 (especificação do Assembly Hack) e 15 (construindo o
Assembler) + Apêndice C.

---

## 👥 Equipe e linguagem

- ✅ **Mudança de dupla permitida** em relação aos projetos anteriores
- ✅ **Mudança de linguagem permitida** (Python, Java, Go, C++, Rust, Haskell, etc.)
- 📝 No `README.md`, registre: integrantes, linguagem e versão, instruções de build e
  execução, e (opcional) justificativa da escolha

---

## 🗂️ Configuração do repositório

1. Crie um **novo repositório** chamado `assembler`
2. Separação de módulos: `parser/`, `code/`, `symbol_table/`, `main/`
3. Inclua um `.gitignore` adequado à linguagem
4. **Histórico de commits é critério de avaliação:**

```
feat(parser): implementa leitura e filtragem de comentarios
feat(symbol_table): adiciona tabela com simbolos predefinidos
fix(code): corrige encoding dos bits dest da instrucao C
docs(readme): adiciona exemplos de uso e testes
```

---

## 🧠 Visão geral da arquitetura Hack

### Formato das instruções

| Tipo | Formato binário | Descrição |
| --- | --- | --- |
| **A-instruction** | `0vvvvvvvvvvvvvvv` | `@valor` — carrega valor/endereço em `A` |
| **C-instruction** | `111accccccdddjjj` | `dest=comp;jump` |

**Posição dos campos na C-instruction (16 bits, do bit 15 ao 0):**

| Bits | Campo | Tamanho |
| --- | --- | --- |
| 15–13 | Prefixo fixo `111` | 3 bits |
| 12–6 | `comp` (inclui o bit `a`) | 7 bits |
| 5–3 | `dest` | 3 bits |
| 2–0 | `jump` | 3 bits |

### Símbolos predefinidos

| Símbolo | Endereço | Função |
| --- | --- | --- |
| `R0`–`R15` | 0–15 | Registradores virtuais |
| `SP`, `LCL`, `ARG`, `THIS`, `THAT` | 0–4 | Ponteiros usados pela VM |
| `SCREEN` | 16384 | Base do mapa de memória de vídeo |
| `KBD` | 24576 | Registrador do teclado |

---

## ⚙️ Componentes a implementar

### 1️⃣ Parser — analisador de arquivos `.asm`

| Método | Descrição |
| --- | --- |
| `newParser(arquivo)` | Abre o `.asm` e prepara a leitura |
| `hasMoreInstructions()` | Indica se há instruções pendentes |
| `advance()` | Avança para a próxima instrução válida |
| `instructionType()` | `A_INSTRUCTION`, `C_INSTRUCTION`, `LABEL` |
| `symbol()` | Símbolo/valor de uma A-instruction ou label |
| `dest()`, `comp()`, `jump()` | Partes de uma C-instruction |

**Exemplo simplificado (Python):**

```python
class Parser:
    def __init__(self, filename):
        with open(filename) as f:
            linhas = [line.split('//')[0].strip() for line in f]
        self.lines = [l for l in linhas if l]
        self.index = 0

    def has_more_instructions(self):
        return self.index < len(self.lines)

    def advance(self):
        line = self.lines[self.index]
        self.index += 1
        return line

    def instruction_type(self, line):
        if line.startswith('@'):
            return 'A_INSTRUCTION'
        if line.startswith('(') and line.endswith(')'):
            return 'LABEL'
        return 'C_INSTRUCTION'
```

### 2️⃣ SymbolTable — tabela de símbolos

- **Primeira passagem:** registrar rótulos (`(LOOP)`, `(END)`) com o endereço **ROM** da
  próxima instrução real — rótulos **não incrementam** o contador
- **Segunda passagem:** alocar variáveis novas a partir de `RAM[16]`
- Inicializar com os símbolos predefinidos da tabela acima

```python
class SymbolTable:
    def __init__(self):
        self.symbols = {f'R{i}': i for i in range(16)}
        self.symbols.update({
            'SP': 0, 'LCL': 1, 'ARG': 2, 'THIS': 3, 'THAT': 4,
            'SCREEN': 16384, 'KBD': 24576,
        })
        self.next_address = 16

    def add_entry(self, symbol, address):
        self.symbols[symbol] = address

    def add_variable(self, symbol):
        if symbol not in self.symbols:
            self.symbols[symbol] = self.next_address
            self.next_address += 1
        return self.symbols[symbol]

    def get_address(self, symbol):
        return self.symbols.get(symbol)
```

### 3️⃣ Code — gerador de código binário

```python
def encode_a_instruction(symbol, table):
    value = int(symbol) if symbol.isdigit() else table.get_address(symbol)
    return '0' + format(value, '015b')

def encode_c_instruction(dest, comp, jump):
    return '111' + COMP_TABLE[comp] + DEST_TABLE[dest] + JUMP_TABLE[jump]
```

### 4️⃣ Main — orquestrador em duas passagens

```python
def main(input_file):
    output_file = input_file.replace('.asm', '.hack')
    symbols = SymbolTable()

    # === Primeira passagem: só rótulos ===
    parser = Parser(input_file)
    rom_address = 0
    while parser.has_more_instructions():
        line = parser.advance()
        if parser.instruction_type(line) == 'LABEL':
            symbols.add_entry(line[1:-1], rom_address)   # NÃO incrementa
        else:
            rom_address += 1

    # === Segunda passagem: gerar binário ===
    parser = Parser(input_file)
    with open(output_file, 'w') as out:
        while parser.has_more_instructions():
            line = parser.advance()
            tipo = parser.instruction_type(line)
            if tipo == 'LABEL':
                continue                                  # não gera código
            elif tipo == 'A_INSTRUCTION':
                symbol = line[1:]
                if not symbol.isdigit():
                    symbols.add_variable(symbol)
                out.write(encode_a_instruction(symbol, symbols) + '\n')
            else:
                dest, comp, jump = parse_c_instruction(line)
                out.write(encode_c_instruction(dest, comp, jump) + '\n')
```

---

## 🔍 Tabelas de encoding

### Campo `comp` (7 bits: `a` + `c1..c6`)

| `comp` | Binário | `comp` (variante com `M`) | Binário |
| --- | --- | --- | --- |
| `0` | `0101010` | — | — |
| `1` | `0111111` | — | — |
| `-1` | `0111010` | — | — |
| `D` | `0001100` | — | — |
| `A` | `0110000` | `M` | `1110000` |
| `!D` | `0001101` | — | — |
| `!A` | `0110001` | `!M` | `1110001` |
| `-D` | `0001111` | — | — |
| `-A` | `0110011` | `-M` | `1110011` |
| `D+1` | `0011111` | — | — |
| `A+1` | `0110111` | `M+1` | `1110111` |
| `D-1` | `0001110` | — | — |
| `A-1` | `0110010` | `M-1` | `1110010` |
| `D+A` | `0000010` | `D+M` | `1000010` |
| `D-A` | `0010011` | `D-M` | `1010011` |
| `A-D` | `0000111` | `M-D` | `1000111` |
| `D&A` | `0000000` | `D&M` | `1000000` |
| `D\|A` | `0010101` | `D\|M` | `1010101` |

> 💡 Toda expressão que usa `A` tem uma gêmea que usa `M`: a diferença é **apenas** o bit
> `a` (o primeiro dos 7). Implemente uma tabela só e derive a variante `M`.

### Campo `dest` (3 bits)

| `dest` | Binário | Armazena em |
| --- | --- | --- |
| (nenhum) | `000` | — |
| `M` | `001` | `RAM[A]` |
| `D` | `010` | `D` |
| `MD` | `011` | `RAM[A]` e `D` |
| `A` | `100` | `A` |
| `AM` | `101` | `A` e `RAM[A]` |
| `AD` | `110` | `A` e `D` |
| `AMD` | `111` | `A`, `D` e `RAM[A]` |

### Campo `jump` (3 bits)

| `jump` | Binário | Condição |
| --- | --- | --- |
| (nenhum) | `000` | nunca salta |
| `JGT` | `001` | `comp > 0` |
| `JEQ` | `010` | `comp == 0` |
| `JGE` | `011` | `comp >= 0` |
| `JLT` | `100` | `comp < 0` |
| `JNE` | `101` | `comp != 0` |
| `JLE` | `110` | `comp <= 0` |
| `JMP` | `111` | sempre |

---

## 🧪 Testando sua implementação

Arquivos do **Project 06**, no pacote oficial:

```
projects/06/
├── add/Add.asm            # R0 + R1 → R2 (sem símbolos)
├── max/Max.asm, MaxL.asm  # máximo entre R0 e R1 (com e sem símbolos)
├── rect/Rect.asm, RectL.asm
└── pong/Pong.asm, PongL.asm
```

> 💡 Os arquivos terminados em `L` são as versões **sem símbolos** (já resolvidos). Comece
> por eles: validam o encoding puro, sem a tabela de símbolos. Só depois passe para as
> versões simbólicas, que exigem as duas passagens.

**Fluxo:** traduza `.asm` → gere `.hack` → compare com o `.hack` de referência via `diff`
(comparação exata) e/ou execute no CPU Emulator.

---

## 📦 Entregáveis

- [ ] Repositório `assembler` criado e acessível
- [ ] Código-fonte organizado por módulos
- [ ] `README.md` completo (integrantes, linguagem, build/execução, exemplos)
- [ ] Mínimo de **{{MIN_COMMITS}} commits** bem descritos
- [ ] Tradução funcional: `Add.asm` e `Max.asm` (obrigatórios), `Rect.asm` (recomendado),
      `Pong.asm` (desafio)

---

## 🎯 Critérios de avaliação *(sugestão — calibrar por oferta)*

| Critério | Peso | Detalhes |
| --- | --- | --- |
| **Correção funcional** | 45% | Os `.hack` gerados batem com a referência / executam corretamente |
| **Tratamento de símbolos** | 20% | Rótulos e variáveis resolvidos corretamente em duas passagens |
| **Qualidade do código** | 20% | Legibilidade, modularidade, padrões da linguagem |
| **Documentação e versionamento** | 15% | README claro, commits descritivos, histórico coerente |

---

## 📚 Recursos adicionais

- Capítulos 11 e 15 do livro + Apêndice C
- *The Elements of Computing Systems*, cap. 6 — <https://www.nand2tetris.org/book>
- Ferramentas e arquivos de teste oficiais: <https://www.nand2tetris.org/software>
- Página da turma: {{LINK_TURMA}}

---

> 💡 **Dicas de implementação:**
> 1. Comece pelas versões `L` (sem símbolos) — validam o encoding isoladamente
> 2. **Duas passagens são essenciais**: não tente resolver símbolos em uma leitura só
> 3. Rótulos **não ocupam** espaço em ROM — não incremente o contador ao registrá-los
> 4. Use tabelas/dicionários para o encoding em vez de `if`s encadeados
> 5. Teste incrementalmente: `Add` → `MaxL` → `Max` → `Rect` → `Pong`
