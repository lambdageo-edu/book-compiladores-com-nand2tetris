# Apêndice E — Referências

## Livros-texto

- **Aho, A. V.; Lam, M. S.; Sethi, R.; Ullman, J. D.** *Compilers: Principles, Techniques, and Tools* (2ª ed.), Pearson, 2006. O "Dragon Book" — a referência clássica de teoria de compiladores; base para boa parte da terminologia usada neste livro (análise léxica/sintática, representações intermediárias, geração de código).
- **Cooper, K. D.; Torczon, L.** *Engineering a Compiler* (2ª ed.), Morgan Kaufmann, 2011. Fonte da caracterização de compiladores como "microcosmo da ciência da computação" (capítulo 0) e de boa parte da discussão sobre representações intermediárias lineares (capítulo 6) e ambiente de execução/*stack frames* (capítulo 13).
- **Nisan, N.; Schocken, S.** *The Elements of Computing Systems* (2ª ed.), MIT Press, 2021. O livro-base do projeto [Nand2Tetris](https://www.nand2tetris.org/) — especificação canônica da linguagem Jack, da VM Hack e da arquitetura Hack, usadas nas quatro partes deste livro.
- **Ricarte, I. L. M.** *Introdução à Compilação*. Referência para a formalização de autômatos finitos, expressões regulares e gramáticas livres de contexto usada no capítulo 1.
- **Sebesta, R. W.** *Concepts of Programming Languages*. Referência para a discussão de convenções de chamada (cdecl, stdcall) comparadas ao protocolo de função da VM Hack, no capítulo 13.

## Estilo e inspiração pedagógica

- **Nystrom, R.** *Crafting Interpreters* — [craftinginterpreters.com](https://craftinginterpreters.com/contents.html). A referência de estilo deste livro: código construído incrementalmente, capítulo a capítulo, junto com o leitor. O mini-tradutor do capítulo 2 e a construção do `JackTokenizer`/`CompilationEngine` (capítulos 4-5) seguem esse espírito.

## Software e ferramentas

- [Nand2Tetris — site oficial](https://www.nand2tetris.org/) — software, arquivos de teste, e os cursos em vídeo associados (Coursera, "From Nand to Tetris").
- [Nand2Tetris Web IDE](https://nand2tetris.github.io/web-ide/compiler) — compilador Jack e emuladores rodando no navegador, sem instalação.
- [AST Explorer](https://astexplorer.net/) — visualização interativa de árvores sintáticas abstratas em diversas linguagens; complementa a discussão de árvore de derivação vs. AST do capítulo 6.
- [Compiler Explorer](https://godbolt.org/) — visualização da representação intermediária/código gerado por compiladores reais; complementa a discussão de representações intermediárias do capítulo 6.

## Sobre este livro

Este material foi construído a partir de três tentativas anteriores do mesmo professor: uma versão em Java (estrutura sólida, mas sem o capítulo de Assembler), uma versão em Python (teoria rica sobre Assembler e VM Translator, mas estilo mais denso), e um conjunto de slides teóricos (a fonte mais completa para os fundamentos de expressões regulares, autômatos e gramáticas do capítulo 1). Este livro consolida as três em uma única narrativa incremental, no estilo de *Crafting Interpreters*, usando o Nand2Tetris como espinha dorsal prática.
