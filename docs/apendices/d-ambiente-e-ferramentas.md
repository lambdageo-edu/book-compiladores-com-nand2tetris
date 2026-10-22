# Apêndice D — Ambiente e ferramentas

Este apêndice cobre o que você precisa instalar para acompanhar o livro e testar seu código contra os materiais oficiais do Nand2Tetris.

## Software oficial do Nand2Tetris

O projeto disponibiliza um pacote de ferramentas gratuito — os emuladores e os arquivos de teste que usamos em todo o livro para validar cada etapa por comparação exata.

1. Baixe o **Nand2Tetris Software Suite** em [nand2tetris.org/software](https://www.nand2tetris.org/software).
2. Descompacte em uma pasta de sua preferência. A estrutura relevante para este curso:
   ```
   nand2tetris/
     tools/              — os emuladores (scripts .sh / .bat)
     projects/
       06/                — Assembler: programas .asm de teste (o .hack esperado é gerado pelo Assembler de tools/)
       07/  08/            — VM Translator: programas .vm de teste + scripts .tst e saídas esperadas .cmp
       09/                — exemplos de programas Jack
       10/                — Analisador léxico/sintático: .jack + .xml de referência
       11/                — Compilador completo: programas .jack de teste (sem .vm esperado; compare com o JackCompiler de tools/)
   ```
3. As ferramentas relevantes, em `tools/`:
   - **CPU Emulator** (`CPUEmulator.sh`/`.bat`): executa arquivos `.hack` (ou `.asm` diretamente) e permite inspecionar registradores e RAM — use para validar o Assembler (capítulo 15).
   - **VM Emulator** (`VMEmulator.sh`/`.bat`): executa arquivos `.vm` diretamente — use para validar a saída do compilador Jack (capítulos 9-10) sem precisar primeiro passar pelo VM Translator.
   - **Assembler** de referência: útil para comparar sua implementação (capítulo 15) contra a oficial em casos de dúvida.

Alternativa sem instalação: a [IDE web oficial](https://nand2tetris.github.io/web-ide/compiler) roda o compilador Jack e os emuladores direto no navegador — suficiente para experimentar a linguagem (capítulo 3) sem baixar nada.

## Ambiente Java (para o código deste livro)

O código incremental do livro é escrito em Java simples (sem dependências externas além de uma biblioteca de testes). Você precisa de:

- **JDK 17 ou superior** — qualquer distribuição (Temurin, Corretto, ou a da sua preferência).
- Um editor ou IDE de sua escolha — o código deste livro não depende de nenhuma IDE específica; qualquer uma com suporte a Java (VS Code + extensão Java, IntelliJ, Eclipse) funciona.
- Para os testes de unidade usados nos capítulos incrementais (comparação de saída XML/VM contra strings esperadas): **JUnit 5**, adicionado como dependência do projeto (via Maven ou Gradle).

## Comparando saídas

Boa parte da validação neste livro é comparação exata de texto — a saída do seu programa contra um arquivo de referência. Duas ferramentas simples ajudam bastante quando um teste falha e você precisa achar a diferença:

```bash
diff meu_output.xml referencia.xml        # mostra as linhas diferentes
diff -y meu_output.xml referencia.xml     # visão lado a lado
```

Preste atenção a duas armadilhas comuns de comparação exata: finais de linha (`\r\n` do Windows vs. `\n` do Linux/Mac) e espaços à direita — normalize ambos antes de comparar, se necessário.

## Sua linguagem de implementação

Lembre-se: o código deste livro é em Java apenas como demonstração (capítulo 0). Suas atividades práticas podem ser em qualquer linguagem — Python, JavaScript, C, Go, o que preferir. As únicas exigências reais do seu ambiente são: conseguir ler arquivos de texto, escrever arquivos de texto, e (para os capítulos de VM Translator e Assembler) fazer aritmética simples e manipulação de strings.
