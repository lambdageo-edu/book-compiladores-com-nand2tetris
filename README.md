# Compiladores — do zero ao Hack

Livro-disciplina de **Compiladores** que usa o projeto [Nand to Tetris](https://www.nand2tetris.org/)
como fio condutor prático: o leitor implementa, do começo ao fim, um compilador para a
linguagem Jack, um tradutor de VM e um montador — levando um programa Jack completo até o
nível de código de máquina.

📖 **Leia online:** <https://lambdageo.github.io/nand2tetris-compilador/>

## Organização

O livro segue a ordem clássica de uma disciplina de Compiladores (front-end → middle-end →
back-end), que é deliberadamente diferente da ordem *bottom-up* do Nand2Tetris original:

| Parte | Conteúdo | Projeto Nand2Tetris |
| --- | --- | --- |
| 1 — Front-end | Linguagens, gramáticas, analisador léxico e sintático | Project 10 |
| 2 — O Compilador | IR, especificação da VM Hack, tabela de símbolos, geração de código VM | Project 11 |
| 3 — O Tradutor VM | Especificação da VM Hack, tradução VM → Assembly | Projects 7 e 8 |
| 4 — O Montador | Assembly Hack, montagem em duas passagens | Project 6 |

Os apêndices reúnem a gramática completa de Jack, as especificações da VM e do Assembly
Hack, o guia de ambiente/ferramentas e as referências bibliográficas.

## Estrutura do repositório

```
docs/                    conteúdo do livro (fonte do site MkDocs)
templates-atividades/    enunciados de atividades práticas, parametrizados por semestre
.github/workflows/       publicação automática no GitHub Pages
mkdocs.yml               configuração do site
requirements.txt         dependências Python
```

Os enunciados em `templates-atividades/` ficam **fora** do build de propósito: são logística
de turma (datas, plataforma de entrega, pesos de avaliação), que muda a cada oferta, enquanto
o livro é conteúdo técnico estável. Veja o README daquela pasta para o uso dos placeholders.

## Build local

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

mkdocs serve                     # servidor local em http://127.0.0.1:8000
mkdocs build --strict            # valida links e navegação
```

`mkdocs build --strict` é o portão de qualidade: qualquer link quebrado ou página fora do
`nav` falha o build. Rode antes de abrir um PR.

## Publicação

O workflow em `.github/workflows/` publica no GitHub Pages a cada push na branch principal.
Não é necessário rodar `mkdocs gh-deploy` manualmente.

## Licença e atribuição

O texto e o código deste livro são de autoria do Prof. Sergio Costa e estão licenciados sob
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.pt-BR). Veja `LICENSE`.

Este livro é um material **independente**, não afiliado aos autores do Nand2Tetris. A
plataforma Hack, a linguagem Jack e a máquina virtual descritas aqui são especificações do
projeto *From Nand to Tetris*, de Noam Nisan e Shimon Schocken, cujo software e arquivos de
teste estão disponíveis em <https://www.nand2tetris.org/software>. As especificações são
descritas com nossas próprias palavras; nenhum material dos autores originais é
redistribuído neste repositório.
