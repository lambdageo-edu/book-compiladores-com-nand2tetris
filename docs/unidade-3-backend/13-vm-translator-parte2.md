# 13. VM Translator — Parte 2: controle de fluxo e funções

A Parte 1 (capítulo 11) traduziu aritmética e acesso à memória — comandos que não atravessam fronteiras de função. Este capítulo fecha o VM Translator (Project 8) com a parte mais sofisticada do tradutor: **rótulos com escopo de função** e o protocolo completo de **chamada de função** (`call`/`function`/`return`), que é o que torna possível compilar recursão corretamente. Ao final, teremos uma cadeia de tradução completa: Jack → VM → Assembly Hack.

## Rótulos com escopo de função

Um `label LOOP` dentro de uma função não pode colidir com um `label LOOP` de outra — mas a VM (capítulo 7) não exige nomes globalmente únicos, só únicos **dentro da função**. A solução do VM Translator é qualificar cada rótulo com o nome da função corrente antes de traduzi-lo para Assembly:

```
label LOOP     (dentro de Main.fibonacci)   →   (Main.fibonacci$LOOP)
goto LOOP                                    →   @Main.fibonacci$LOOP
                                                  0;JMP
if-goto LOOP                                 →   @SP
                                                  AM=M-1
                                                  D=M
                                                  @Main.fibonacci$LOOP
                                                  D;JNE
```

O tradutor só precisa lembrar qual é a função "corrente" (atualizada a cada comando `function` traduzido) para montar esse prefixo — nenhuma estrutura de dados nova é necessária.

## O registro de ativação (stack frame)

Toda linguagem que suporta chamadas de função — e, em particular, chamadas **recursivas** — precisa de uma forma de isolar o estado de cada invocação. Essa unidade de isolamento é o **registro de ativação** (*activation record*, ou *stack frame*): um bloco de memória, alocado a cada chamada, que guarda tudo que aquela invocação específica precisa — seus argumentos, suas variáveis locais, e informação suficiente para restaurar o contexto do chamador quando a função retornar.

Dois princípios sustentam esse mecanismo:

1. **Isolamento de escopo**: cada invocação tem seu próprio "espaço de nomes" — o tradutor nunca precisa de nomes globais para distinguir a variável local `x` da função `A` da variável local `x` da função `B`; basta que cada uma resolva `local 0` relativo ao seu **próprio** `LCL`.
2. **Suporte a recursão**: a mesma função pode estar ativa **múltiplas vezes simultaneamente** — `fib(5)` chama `fib(4)`, que chama `fib(3)`, e assim por diante. Cada uma dessas ativações ocupa um frame distinto e independente na pilha.

Na VM Hack, o registro de ativação é uma sequência determinística de valores na pilha:

```
┌─────────────────────────────┐
│  variáveis locais            │  ← LCL aponta aqui
├─────────────────────────────┤
│  THAT salvo (do chamador)    │
│  THIS salvo (do chamador)    │
│  ARG salvo (do chamador)     │
│  LCL salvo (do chamador)     │
│  endereço de retorno          │
├─────────────────────────────┤
│  argumentos (nArgs valores)   │  ← ARG aponta aqui
├─────────────────────────────┤
│  frames de chamadas anteriores│
└─────────────────────────────┘
                                  ↑ SP
```

Vale notar uma simplificação importante da VM Hack em relação a linguagens com procedimentos aninhados (Pascal, Scheme): o frame guarda apenas um **control link** — a cadeia de retorno, que permite restaurar o chamador — mas nenhum **access link** (que permitiria acessar variáveis de um escopo léxico envolvente). Como Jack não tem subroutines aninhadas, essa complexidade simplesmente não existe aqui, e o acesso a variáveis é sempre direto, via `LCL`/`ARG` da própria função.

## `call`: cinco operações atômicas

Traduzir `call functionName nArgs` envolve, em sequência: salvar o endereço de retorno, salvar os quatro registradores do chamador, reposicionar `ARG` e `LCL` para o novo contexto, e saltar para a função.

**1. Salvar o endereço de retorno** — um rótulo único é gerado para cada chamada traduzida, e seu endereço é empilhado:

```
@RETURN_LABEL_42
D=A
@SP
A=M
M=D
@SP
M=M+1
```

**2. Salvar `LCL`, `ARG`, `THIS`, `THAT`** — quatro blocos idênticos ao acima, trocando apenas o registrador de origem.

**3. Reposicionar `ARG`** — o novo `ARG` deve apontar para onde os argumentos (já empilhados pelo chamador, antes do `call`) começam: `ARG = SP - 5 - nArgs` (o `5` é o tamanho fixo dos metadados que acabamos de empilhar):

```
@5
D=A
@nArgs
D=D+A
@SP
D=M-D
@ARG
M=D
```

**4. Reposicionar `LCL`** — simplesmente `LCL = SP` (as variáveis locais da nova função começarão exatamente onde a pilha está agora):

```
@SP
D=M
@LCL
M=D
```

**5. Saltar para a função:**

```
@functionName
0;JMP
(RETURN_LABEL_42)
```

O rótulo `RETURN_LABEL_42` é declarado logo **depois** do salto — é para lá que o `return` da função chamada vai desviar quando terminar.

Custo total: cerca de 34 instruções — um overhead fixo, independente da complexidade da função chamada.

## `function`: inicializando variáveis locais

O comando `function nome nLocais` marca o ponto de entrada e tem uma única responsabilidade: garantir que as `nLocais` variáveis locais comecem zeradas (é a semântica que Jack promete: toda `var` não explicitamente inicializada vale 0). Isso é um laço simples de "empilhar zero, `nLocais` vezes":

```
(nome)
    @nLocais
    D=A
    @R13
    M=D                    // R13 = contador
(nome$LOOP_INIT)
    @R13
    D=M
    @nome$END_INIT
    D;JEQ                   // contador == 0 → fim
    @0
    D=A
    @SP
    A=M
    M=D                     // push 0
    @SP
    M=M+1
    @R13
    M=M-1
    @nome$LOOP_INIT
    0;JMP
(nome$END_INIT)
```

## `return`: desmontando o frame em ordem reversa

`return` é a operação inversa de `call`: extrair o valor de retorno, restaurar `SP`/`LCL`/`ARG`/`THIS`/`THAT` do chamador, e saltar de volta. A ordem importa — os registradores foram salvos numa sequência (`RetAddr`, `LCL`, `ARG`, `THIS`, `THAT`), e por isso precisam ser restaurados na ordem **reversa** (`THAT`, `THIS`, `ARG`, `LCL`), para que o deslocamento a partir do topo do frame (`endFrame`) continue válido a cada passo.

```
// 1. Guardar o topo do frame atual (LCL será sobrescrito em breve)
@LCL
D=M
@R14
M=D              // R14 = endFrame

// 2. Extrair o endereço de retorno: RAM[endFrame - 5]
@5
D=A
@R14
A=M-D
D=M
@R15
M=D              // R15 = retAddr

// 3. Colocar o valor de retorno onde o chamador vai procurá-lo: *ARG
@SP
AM=M-1
D=M
@ARG
A=M
M=D              // *ARG = resultado

// 4. Restaurar SP: logo após o valor de retorno
@ARG
D=M+1
@SP
M=D

// 5. Restaurar THAT, THIS, ARG, LCL — nessa ordem, todos relativos a R14 (endFrame)
@1
D=A
@R14
A=M-D
D=M
@THAT
M=D
// (o mesmo padrão, com deslocamentos 2, 3, 4, para THIS, ARG, LCL)

// 6. Saltar de volta
@R15
A=M
0;JMP
```

Por que salvar `retAddr` em `R15` e `endFrame` em `R14`? Porque `LCL` é justamente um dos registradores que estamos prestes a sobrescrever — sem guardar seu valor original em algum lugar seguro, perderíamos a referência para localizar os demais metadados salvos.

## Recursão: o teste definitivo

O protocolo acima, aplicado sem alteração a cada chamada — inclusive quando a função chama a si mesma —, é o que sustenta a recursão:

```
function Main.fibonacci 0
    push argument 0
    push constant 2
    lt
    if-goto BASE
    push argument 0
    push constant 1
    sub
    call Main.fibonacci 1
    push argument 0
    push constant 2
    sub
    call Main.fibonacci 1
    add
    return
label BASE
    push argument 0
    return
```

Cada chamada recursiva empilha um novo frame completo — `fib(3)` chamando `fib(2)` chamando `fib(1)` — sem que nenhuma delas jamais veja ou corrompa o frame das demais. A pilha cresce a cada `call` e encolhe a cada `return`; é essa disciplina de empilhamento estrito que garante a correção, sem qualquer análise especial para o caso recursivo.

## Bootstrap: dando a partida

Antes de qualquer código VM rodar, a máquina precisa ser inicializada: `SP` apontando para o início da pilha (RAM 256), e uma chamada inicial a `Sys.init` (a função que, por convenção, chama `Main.main`):

```
@256
D=A
@SP
M=D
```

seguido de um `call Sys.init 0` — usando exatamente o mesmo mecanismo de `call` descrito acima. Esse código de bootstrap é emitido uma única vez, no início do arquivo `.asm` final, antes da tradução de qualquer arquivo `.vm`.

## Comparação com convenções de chamada reais

Vale posicionar esse protocolo ao lado de convenções usadas em processadores reais, para enxergar o que é essencial e o que é escolha de projeto:

| Responsabilidade | cdecl (C, x86) | stdcall (Windows) | VM Hack |
|---|---|---|---|
| Empilhar argumentos | chamador | chamador | chamador |
| Descartar argumentos após a chamada | chamador | callee (via `RET n`) | callee (implícito, via `ARG`) |
| Salvar registradores | parcialmente ambos | parcialmente ambos | callee salva; caller nada faz |
| Valor de retorno | registrador (`EAX`) | registrador (`EAX`) | pilha (posição `ARG`) |
| Overhead por chamada | variável (poucos registradores movidos) | variável | fixo (~34 + ~38 instruções) |

A VM Hack sacrifica desempenho (um overhead fixo e relativamente alto por chamada) por simplicidade extrema de implementação — não há alocação de registradores, não há análise de quais registradores precisam ser preservados: **tudo** é sempre salvo e restaurado, do mesmo jeito, sempre. É uma escolha de projeto coerente com o resto deste curso: corretude e clareza antes de desempenho.

## O que vem a seguir

Com o VM Translator completo (Partes 1 e 2), qualquer programa Jack — via o compilador da Unidade 2 — já pode ser traduzido, em duas etapas, até Assembly Hack. Falta uma última peça: transformar esse Assembly em código de máquina de verdade. Os capítulos 13 a 15 fecham essa lacuna: uma consolidação teórica do caminho até aqui, a especificação completa da linguagem Assembly Hack, e finalmente a construção do Assembler.
