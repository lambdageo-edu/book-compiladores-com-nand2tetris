# 13. VM Translator — Parte 2: controle de fluxo e funções

A primeira metade do VM Translator (capítulo 12) traduziu aritmética e acesso à memória — comandos que não atravessam fronteiras de função. Este capítulo fecha o VM Translator (Project 8) com a parte mais sofisticada do tradutor: **desvios com escopo de função** e o protocolo completo de **chamada de função** (`call`/`function`/`return`), que é o que torna possível compilar recursão corretamente. Ao final, teremos uma cadeia de tradução completa: Jack → VM → Assembly Hack.

O caminho do capítulo segue a mesma ordem do anterior: primeiro entender **o que** cada comando promete (a abstração), depois **como** realizá-lo com os recursos do Hack (a implementação).

## Desvios: `label`, `goto` e `if-goto`

Um programa VM é executado em sequência, comando após comando, até que um desvio mande a execução para outro ponto. A VM tem dois tipos de desvio e uma declaração:

| Comando | Efeito |
|---|---|
| `label L` | declara o rótulo `L` (não executa nada) |
| `goto L` | desvio incondicional: continua a execução no comando logo após `label L` |
| `if-goto L` | `cond = pop`; se `cond` for diferente de zero (verdadeiro), desvia para `L`; caso contrário, segue em frente |

O ponto a reter sobre `if-goto` é que a condição **é desempilhada**: o código precisa empilhá-la logo antes do desvio, e ela não fica na pilha depois. Veja como um laço `while` de alto nível vira VM — a função abaixo calcula `x * y` somando `x` a si mesmo `y` vezes:

```
// Retorna x * y                         function mult 2      // 2 variáveis locais
int mult(int x, int y) {                     push constant 0    // sum = 0
    int sum = 0;                             pop local 0
    int n = 1;                               push constant 1    // n = 1
    while !(n > y) {                         pop local 1
        sum += x;                        label LOOP
        n++;                                 push local 1       // if (n > y) goto END
    }                                        push argument 1
    return sum;                              gt
}                                            if-goto END
                                             push local 0       // sum += x
                                             push argument 0
                                             add
                                             pop local 0
                                             push local 1       // n++
                                             push constant 1
                                             add
                                             pop local 1
                                             goto LOOP
                                         label END
                                             push local 0       // return sum
                                             return
```

O padrão é o mesmo do capítulo 9: a condição de saída (`n > y`) é calculada, `if-goto END` sai do laço quando ela é verdadeira, e um `goto LOOP` no fim do corpo volta ao teste.

### Traduzindo os desvios

O Assembly Hack já tem rótulos e saltos (capítulo 11), então a tradução é quase direta. O único cuidado é com os **nomes**: um `label LOOP` dentro de uma função não pode colidir com um `label LOOP` de outra — mas a VM (capítulo 7) não exige nomes globalmente únicos, só únicos **dentro da função**. A solução do VM Translator é qualificar cada rótulo com o nome da função corrente, no formato `nomeDaFunção$rótulo`:

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

O `if-goto` é o `pop` do capítulo 12 (`SP--; D = *SP`) seguido de um salto condicional `D;JNE`: qualquer valor diferente de zero desvia. O tradutor só precisa lembrar qual é a função "corrente" (atualizada a cada comando `function` traduzido) para montar o prefixo — nenhuma estrutura de dados nova é necessária. (Os dois primeiros testes do Project 8, `BasicLoop` e `FibonacciSeries`, têm rótulos fora de qualquer `function`; para eles, qualquer prefixo fixo serve.)

## Funções: a abstração

Na VM, aplicar um operador primitivo e chamar uma função têm a **mesma aparência**. Compare as duas operações na tradução de `sqrt(x - 17 + x * 5)`:

```
push local 0         // x
push constant 17
sub                  // operação primitiva
push local 0         // x
push constant 5
call Math.multiply 2 // operação "estendida"
add
call Math.sqrt 1
```

`sub` e `add` são fixos na VM; `Math.multiply` e `Math.sqrt` são funções que alguém escreveu — em Jack, no sistema operacional. Mas, do ponto de vista de quem chama, `call Math.multiply 2` se comporta exatamente como um `add`: consome os dois valores do topo da pilha e deixa no lugar um único resultado. A linguagem VM é **extensível**: cada função definida é, para o resto do programa, um novo comando.

Esse comportamento é um **contrato**, e é útil enxergá-lo pelos dois lados.

**O que o chamador (caller) vê:**

- Antes de chamar, empilho tantos argumentos quantos a função espera.
- Chamo com `call nomeDaFunção nArgs`.
- Quando a função retorna, os argumentos que empilhei **sumiram** da pilha, e no lugar deles há um **valor de retorno** (que sempre existe — funções `void` retornam um valor qualquer, descartado com `pop temp 0`).
- Quando a função retorna, todos os meus segmentos (`local`, `argument`, `this`, `that`, `pointer`) estão **exatamente como estavam** antes da chamada — exceto `temp`, que fica indefinido, e `static`, que a função pode ter alterado.

**O que a função chamada (callee) vê:**

- Ao começar, meu segmento `argument` já contém os valores passados pelo chamador.
- Meu segmento `local` já foi alocado e zerado.
- Meu segmento `static` é o do arquivo `.vm` ao qual pertenço.
- `this`, `that`, `pointer` e `temp` estão indefinidos ao entrar.
- Minha pilha de trabalho começa vazia.
- Antes de executar `return`, preciso empilhar um valor.

Por exemplo, se a pilha do chamador tem `17` e `212` no topo, depois de `call mult 2` ela tem só `3604` — o efeito líquido é que **os argumentos foram substituídos pelo valor da função**. Todo o resto do contrato (alocar e zerar `local`, ancorar `argument`, preservar e restaurar os segmentos do chamador, devolver o valor no lugar certo) não é responsabilidade nem do chamador nem da função: é o **VM Translator** que precisa gerar o código que faz isso acontecer. É o assunto do resto do capítulo.

## O registro de ativação (stack frame)

Um programa tem muitas funções, mas, a cada instante, só algumas estão em execução — as que formam a **cadeia de chamadas** (`main` chamou `foo`, que chamou `bar`, que chamou `sqrt`). Só a última está de fato rodando; as outras estão **esperando** que ela retorne. Cada uma delas tem um **estado** — sua pilha de trabalho e seus segmentos — que precisa ser criado quando a função começa, mantido enquanto ela espera, e reciclado quando ela retorna.

Toda linguagem que suporta chamadas de função — e, em particular, chamadas **recursivas** — precisa de uma forma de isolar o estado de cada invocação. Essa unidade de isolamento é o **registro de ativação** (*activation record*, ou *stack frame*): um bloco de memória, alocado a cada chamada, que guarda tudo que aquela invocação específica precisa — seus argumentos, suas variáveis locais, e informação suficiente para restaurar o contexto do chamador quando a função retornar.

Dois princípios sustentam esse mecanismo:

1. **Isolamento de escopo**: cada invocação tem seu próprio "espaço de nomes" — o tradutor nunca precisa de nomes globais para distinguir a variável local `x` da função `A` da variável local `x` da função `B`; basta que cada uma resolva `local 0` relativo ao seu **próprio** `LCL`.
2. **Suporte a recursão**: a mesma função pode estar ativa **múltiplas vezes simultaneamente** — `fib(5)` chama `fib(4)`, que chama `fib(3)`, e assim por diante. Cada uma dessas ativações ocupa um frame distinto e independente na pilha.

Como as chamadas seguem uma disciplina LIFO (a última função chamada é sempre a primeira a retornar), os frames cabem numa **única pilha global**: a pilha da VM, a partir de `RAM[256]`, contém um bloco para cada função da cadeia de chamadas, um em cima do outro, e a pilha de trabalho da função corrente fica no topo. Na VM Hack, cada bloco é uma sequência determinística de valores:

```
              ┌──────────────────────────────┐
     SP  →    │  (próxima posição livre)      │   endereços mais altos
              ├──────────────────────────────┤
              │  pilha de trabalho da função  │
              ├──────────────────────────────┤
              │  local nLocais-1              │
              │  ...                          │
     LCL →    │  local 0                      │
              ├──────────────────────────────┤
              │  THAT salvo (do chamador)     │   LCL - 1
              │  THIS salvo (do chamador)     │   LCL - 2
              │  ARG salvo (do chamador)      │   LCL - 3
              │  LCL salvo (do chamador)      │   LCL - 4
              │  endereço de retorno          │   LCL - 5
              ├──────────────────────────────┤
              │  argument nArgs-1             │
              │  ...                          │
     ARG →    │  argument 0                   │
              ├──────────────────────────────┤
              │  frames de chamadas anteriores│   endereços mais baixos
              └──────────────────────────────┘
```

Os argumentos foram empilhados pelo chamador antes do `call` (eles eram o topo da pilha de trabalho do chamador), os cinco valores salvos são empilhados pelo próprio `call`, e as variáveis locais pelo comando `function`. Os deslocamentos à direita — `LCL - 1` a `LCL - 5` — são exatamente os que o `return` vai usar para desmontar o frame.

Vale notar uma simplificação importante da VM Hack em relação a linguagens com procedimentos aninhados (Pascal, Scheme): o frame guarda apenas um **control link** — a cadeia de retorno, que permite restaurar o chamador — mas nenhum **access link** (que permitiria acessar variáveis de um escopo léxico envolvente). Como Jack não tem subroutines aninhadas, essa complexidade simplesmente não existe aqui, e o acesso a variáveis é sempre direto, via `LCL`/`ARG` da própria função.

## O que o tradutor gera, visto de cima

Antes de entrar no Assembly de cada comando, vale ver o **esqueleto** do código gerado para um par chamador/função. Do lado esquerdo, código VM; do lado direito, a estrutura do `.asm` que o tradutor produz:

```
function Foo.main 4                 (Foo.main)
    ...                                 // inicializa as 4 variáveis locais
    push constant 19                    // push constant 19
    push local 3                        // push local 3
    call Bar.mult 2                     // salva o estado de Foo.main na pilha,
                                        // prepara ARG e LCL para Bar.mult, e:
                                        @Bar.mult
                                        0;JMP
                                    (Foo.main$ret.1)
    neg                                 // neg
    ...                                 ...
function Bar.mult 2                 (Bar.mult)
    ...                                 // inicializa as 2 variáveis locais
    push local 1                        // push local 1
    return                              // devolve o valor ao chamador, restaura o
                                        // estado de Foo.main, e salta para o
                                        // endereço de retorno (aqui, Foo.main$ret.1)
```

Três rótulos são criados e "plugados" pelo tradutor: `(Foo.main)` e `(Bar.mult)`, que marcam o ponto de entrada de cada função (é para lá que `call` salta), e `(Foo.main$ret.1)`, que marca o ponto, logo após a chamada, onde `Foo.main` deve continuar quando `Bar.mult` retornar. Como `return` não sabe estaticamente quem o chamou (`Bar.mult` pode ser chamada de vários lugares), esse endereço de retorno viaja **pela pilha**: o `call` o empilha, e o `return` o lê de volta.

## `call`: cinco operações atômicas

Em pseudocódigo, `call nomeDaFunção nArgs` faz:

```
push retAddrLabel       // empilha o endereço de retorno (um rótulo gerado)
push LCL                // salva os quatro ponteiros do chamador
push ARG
push THIS
push THAT
ARG = SP - 5 - nArgs    // reposiciona ARG para o primeiro argumento
LCL = SP                // reposiciona LCL
goto nomeDaFunção       // transfere o controle
(retAddrLabel)          // declara o rótulo de retorno
```

Em Assembly, operação por operação:

**1. Salvar o endereço de retorno** — um rótulo único é gerado para cada chamada traduzida (`Foo.main$ret.1`, ver a seção sobre símbolos adiante), e seu **endereço** é empilhado:

```
@Foo.main$ret.1
D=A
@SP
A=M
M=D
@SP
M=M+1
```

**2. Salvar `LCL`, `ARG`, `THIS`, `THAT`** — quatro blocos quase iguais ao anterior, com uma diferença importante: agora o que se empilha é o **conteúdo** do registrador (`D=M`), e não um endereço (`D=A`, como no rótulo de retorno):

```
@LCL
D=M          // D = valor de LCL — com D=A, empilharíamos o número 1
@SP
A=M
M=D
@SP
M=M+1
// (o mesmo bloco para ARG, THIS e THAT)
```

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
@nomeDaFunção
0;JMP
(Foo.main$ret.1)
```

O rótulo de retorno é declarado logo **depois** do salto — é para lá que o `return` da função chamada vai desviar quando terminar.

Custo total, na versão mostrada: 49 instruções (7 para o endereço de retorno, 4 × 7 para os registradores, 8 para `ARG`, 4 para `LCL` e 2 para o salto) — um overhead fixo, independente da complexidade da função chamada.

## `function`: inicializando variáveis locais

O comando `function nome nLocais` marca o ponto de entrada e tem uma única responsabilidade além de declarar o rótulo: garantir que as `nLocais` variáveis locais comecem zeradas (é o contrato do lado da função: `local` já alocado e zerado). Em pseudocódigo:

```
(nome)                  // declara o ponto de entrada
repeat nLocais times:
    push 0              // aloca e zera uma variável local
```

Como `nLocais` é conhecido em tempo de tradução, o mais simples é o tradutor **repetir** o código de "empilhar zero" `nLocais` vezes — sem laço nenhum em Assembly. Cada repetição é um `*SP = 0; SP++`:

```
(nome)
@SP
A=M
M=0          // *SP = 0
@SP
M=M+1        // SP++
// (... repetido nLocais vezes)
```

São 5 instruções por variável local (`function nome 0` gera só o rótulo). Repare que, como `LCL = SP` foi feito no `call`, essas posições zeradas são exatamente `local 0`, `local 1`, ..., `local nLocais-1`.

!!! warning "Cuidado com laços e rótulos internos"
    Uma alternativa é gerar um laço em Assembly que empilha zero `nLocais` vezes. Funciona, mas exige rótulos internos — e se eles forem nomeados como os rótulos do usuário, por exemplo `nome$LOOP_INIT`, uma função que declare `label LOOP_INIT` geraria o **mesmo** símbolo, e o programa saltaria para o lugar errado. Rótulos criados pelo tradutor precisam de nomes que o programador VM não consiga produzir. A versão desenrolada evita o problema por completo.

## `return`: desmontando o frame em ordem reversa

`return` é a operação inversa de `call`. Em pseudocódigo:

```
endFrame = LCL               // endFrame é uma variável temporária
retAddr = *(endFrame - 5)    // obtém o endereço de retorno
*ARG = pop()                 // reposiciona o valor de retorno para o chamador
SP = ARG + 1                 // reposiciona o SP do chamador
THAT = *(endFrame - 1)       // restaura THAT do chamador
THIS = *(endFrame - 2)       // restaura THIS do chamador
ARG  = *(endFrame - 3)       // restaura ARG do chamador
LCL  = *(endFrame - 4)       // restaura LCL do chamador
goto retAddr                 // salta para o endereço de retorno
```

O efeito líquido: o chamador volta a rodar, com o valor de retorno no topo da sua pilha — **no lugar onde estava o primeiro argumento**, que é exatamente o que o contrato promete. Os registradores foram salvos numa sequência (`retAddr`, `LCL`, `ARG`, `THIS`, `THAT`) e são lidos a partir de `endFrame`, cada um no seu deslocamento fixo. Em Assembly, usando `R14` para `endFrame` e `R15` para `retAddr`:

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

Custo total: 52 instruções (4 + 7 + 6 + 4 + 4 × 7 + 3).

Por que guardar `endFrame` e `retAddr` antes de qualquer outra coisa? São dois motivos diferentes:

- **`endFrame` em `R14`**: `LCL` é um dos registradores que o passo 5 vai sobrescrever. Sem uma cópia do seu valor original, perderíamos a referência para localizar os metadados salvos.
- **`retAddr` em `R15`**: o passo 3 grava o valor de retorno em `*ARG`. Se a função foi chamada com **zero argumentos**, `ARG` aponta exatamente para a posição onde o `call` guardou o endereço de retorno (`LCL - 5`) — e o passo 3 o apagaria. Lendo o endereço de retorno no passo 2, antes disso, o problema desaparece.

E por que `ARG` é restaurado **depois** de ser usado nos passos 3 e 4? Porque, até ali, `ARG` ainda é o da função que está retornando — e é justamente ele que diz onde o valor de retorno deve ficar.

## Acompanhando uma chamada na pilha global

Para ver o protocolo inteiro funcionando, vamos acompanhar a pilha durante a execução de `3 + 8 * 5`, em que `main` chama a função `mult` mostrada no início do capítulo:

```
function Main.main 0                 function Main.mult 2
    push constant 3                      (o código de mult mostrado na
    push constant 8                       seção sobre desvios)
    push constant 5
    call Main.mult 2
    add
    ...
```

Suponha que, quando `Main.main` começa, `SP = 261`, `LCL = 261` e `ARG = 256`, com `THIS = 3000` e `THAT = 4000` (é o estado que um script de teste do Project 8 prepararia). Os números abaixo foram obtidos executando o Assembly gerado pela tradução deste capítulo:

**1. Logo antes do `call`**, depois dos três `push`:

| Endereço | 261 | 262 | 263 |
|---|---|---|---|
| Valor | 3 | 8 | 5 |

`SP = 264`, `LCL = 261`, `ARG = 256`. Os dois argumentos de `mult` (8 e 5) estão no topo da pilha de trabalho de `main`.

**2. Logo depois do `call`** (na entrada de `Main.mult`):

| Endereço | 261 | 262 | 263 | 264 | 265 | 266 | 267 | 268 |
|---|---|---|---|---|---|---|---|---|
| Valor | 3 | 8 | 5 | 70 | 261 | 256 | 3000 | 4000 |
| Papel | pilha de `main` | `argument 0` | `argument 1` | retAddr | LCL salvo | ARG salvo | THIS salvo | THAT salvo |

O `call` empilhou cinco valores (`SP` passou de 264 a 269); `ARG = 269 - 5 - 2 = 262` agora aponta para o 8, e `LCL = SP = 269`. O 70 é o endereço, na ROM, do rótulo `Main.main$ret.0` nesta tradução.

**3. Depois de `function Main.mult 2`**: `RAM[269] = RAM[270] = 0` (`local 0` e `local 1`), `SP = 271`. A pilha de trabalho de `mult` começa vazia, em 271.

**4. Depois do `return`** de `mult` (que empilhou `local 0 = 40` antes de retornar):

| Endereço | 261 | 262 |
|---|---|---|
| Valor | 3 | 40 |

`SP = ARG + 1 = 263`, e `LCL = 261`, `ARG = 256`, `THIS = 3000`, `THAT = 4000` — exatamente como antes do `call`. O 40 ocupa o lugar do primeiro argumento, e tudo acima dele (o resto do frame de `mult`) foi "reciclado": os valores ainda estão na RAM, mas `SP` está abaixo deles, e o próximo `push` vai sobrescrevê-los.

**5. Depois do `add`**: `RAM[261] = 43`, `SP = 262`.

Do ponto de vista de `main`, nada de especial aconteceu: ele empilhou 8 e 5, chamou `mult`, e recebeu 40 no lugar — exatamente como se `mult` fosse um comando primitivo. Repare também que `Main.main` e `Main.mult` podem ter, cada uma, um `label END` sem conflito: eles viram `Main.main$END` e `Main.mult$END`.

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

Cada chamada recursiva empilha um novo frame completo — `fib(3)` chamando `fib(2)` chamando `fib(1)` — sem que nenhuma delas jamais veja ou corrompa o frame das demais: cada uma tem o seu próprio `argument 0`, porque cada `call` reposiciona `ARG`, e cada `return` o restaura. A pilha cresce a cada `call` e encolhe a cada `return`; é essa disciplina de empilhamento estrito que garante a correção, sem qualquer análise especial para o caso recursivo.

## Bootstrap: dando a partida

Programas VM seguem duas convenções: um dos arquivos se chama `Main.vm` e contém uma função `main`; e a execução não começa em `Main.main`, mas numa função do sistema operacional, `Sys.init`, sem argumentos, que inicializa o SO e então chama `Main.main` (e entra num laço infinito quando ela termina — `Sys.init` nunca retorna). A implementação da VM, por sua vez, garante que a pilha começa em `RAM[256]`.

As duas convenções são realizadas por um pequeno **código de bootstrap**, que deve ocupar o início da ROM (endereço 0):

```
SP = 256
call Sys.init 0
```

O `SP = 256` em Assembly:

```
@256
D=A
@SP
M=D
```

seguido de um `call Sys.init 0` — usando exatamente o mesmo mecanismo de `call` descrito acima. Esse código é emitido uma única vez, no início do arquivo `.asm` final, antes da tradução de qualquer arquivo `.vm`. Como `Sys.init` nunca retorna, o endereço de retorno empilhado por esse `call` nunca é usado — mas mantê-lo deixa a pilha exatamente no formato que o resto do programa espera (`Sys.init` começa com `SP = LCL = 261` e `ARG = 256`).

## Os símbolos do programa traduzido

Com o tradutor completo, o `.asm` gerado usa cinco famílias de símbolos. Vale reuni-los numa tabela, porque é ela que garante que nenhum símbolo colide com outro:

| Símbolo | Uso |
|---|---|
| `SP`, `LCL`, `ARG`, `THIS`, `THAT` | Pré-definidos: `RAM[0]` a `RAM[4]` (topo da pilha e bases dos segmentos) |
| `R13`, `R14`, `R15` | Pré-definidos: registradores de rascunho do tradutor |
| `Xxx.i` | Variável `static i` do arquivo `Xxx.vm`; o Assembler a aloca a partir de `RAM[16]` |
| `nomeDaFunção$rótulo` | Um `label rótulo` dentro de `nomeDaFunção` (destino de `goto`/`if-goto`) |
| `nomeDaFunção` | Ponto de entrada da função (gerado pelo comando `function`) |
| `nomeDaFunção$ret.i` | Endereço de retorno da i-ésima chamada feita **de dentro** de `nomeDaFunção` (gerado pelo `call`) |

Os rótulos das comparações do capítulo 12 (`TRUE_EQ_0`, ...) também são criados pelo tradutor; basta que o contador seja global ao programa para que sejam únicos.

## Traduzindo um programa inteiro

Até aqui, cada comando foi traduzido isoladamente. Um programa real, porém, é um **diretório** com vários arquivos `.vm` — um por classe Jack, mais os do sistema operacional —, e o VM Translator precisa juntar tudo em um único `.asm`:

- **Entrada:** um arquivo `Xxx.vm` (gera `Xxx.asm`) ou um diretório `Xxx/` (gera `Xxx/Xxx.asm`, com a tradução de todos os `.vm` do diretório, um depois do outro).
- **`static` por arquivo:** o prefixo do símbolo de `static i` é o nome do arquivo **sendo traduzido no momento** — `static 0` em `Main.vm` vira `@Main.0`, em `Ball.vm` vira `@Ball.0`. É isso que dá a cada classe o seu próprio segmento `static`.
- **Rótulos únicos no programa todo:** os rótulos de retorno do `call` e os de comparação precisam ser únicos no `.asm` final, não só dentro de cada arquivo. O formato `nomeDaFunção$ret.i`, com um contador por função, resolve o primeiro caso; um contador global resolve o segundo.
- **Bootstrap só quando há `Sys.init`:** os primeiros testes do Project 8 não têm `Sys.vm` e inicializam a máquina pelo próprio script `.tst`; eles devem ser traduzidos **sem** o bootstrap. Os testes que trazem `Sys.vm` com uma função `Sys.init` exigem o bootstrap.

A arquitetura é a mesma do capítulo 12, com acréscimos. O `Parser` passa a reconhecer os seis comandos novos (tipos `label`, `goto`, `if`, `function`, `call`, `return`). O `CodeWriter` ganha uma operação por comando novo — `writeLabel(rótulo)`, `writeGoto(rótulo)`, `writeIf(rótulo)`, `writeFunction(nome, nLocais)`, `writeCall(nome, nArgs)` e `writeReturn()` —, mais `setFileName(nome)`, que o `Main` chama ao começar cada arquivo do diretório (para os símbolos `static`), e o código de bootstrap, emitido uma vez no início da saída. O estado do `CodeWriter` passa a incluir o nome da função corrente (para os rótulos `função$rótulo`) e os contadores de chamadas.

## Exercício guiado

Sua atividade é completar o `VMTranslator` do capítulo 12 com `label`, `goto`, `if-goto`, `function`, `call` e `return`, mais a tradução de diretórios e o bootstrap. Os testes oficiais (pasta `projects/08`), na ordem sugerida:

| Teste | O que exercita | Bootstrap? |
|---|---|---|
| `ProgramFlow/BasicLoop` | `label`, `if-goto`: soma 1 + 2 + ... + n num laço | não |
| `ProgramFlow/FibonacciSeries` | `goto`, `if-goto`, laços com memória: grava os n primeiros termos da série de Fibonacci a partir de um endereço dado (processamento de array via `that`) | não |
| `FunctionCalls/SimpleFunction` | `function` e `return` com variáveis locais (o script monta na RAM um frame "falso", como se a função tivesse sido chamada) | não |
| `FunctionCalls/NestedCall` | chamadas aninhadas (`Sys.init` → `Sys.main` → `Sys.add12`); verifica que `local` é zerado e que todos os registradores do frame são preservados | indiferente (ver abaixo) |
| `FunctionCalls/FibonacciElement` | recursão, vários arquivos (`Main.vm` + `Sys.vm`) traduzidos num único `FibonacciElement.asm` | sim |
| `FunctionCalls/StaticsTest` | `static` separado por arquivo (`Class1.vm`, `Class2.vm`, `Sys.vm`) | sim |

O caso de `NestedCall` é curioso: o script prepara a memória como se o bootstrap já tivesse chamado `Sys.init` (`SP = LCL = 261`, `ARG = 256` e um frame falso em 256–260), e o estado que o bootstrap produziria é exatamente esse — por isso o teste passa com ou sem bootstrap.

Como no capítulo 12, cada teste tem um `XxxVME.tst` para ver o comportamento esperado no VM Emulator, e um `Xxx.tst` que roda o seu `.asm` no CPU Emulator e compara o resultado com o `.cmp`. Se `SimpleFunction` passa mas `FibonacciElement` falha ou trava, `NestedCall` é o teste intermediário recomendado: ele vem com uma documentação própria (`NestedCall.html`) que mostra o estado esperado da pilha a cada passo.

## Teste rápido

??? question "1. Traduza `if-goto END` dentro da função `Main.loop`."
    `@SP`, `AM=M-1`, `D=M`, `@Main.loop$END`, `D;JNE`. A condição é desempilhada, e qualquer valor diferente de zero desvia.

??? question "2. Uma função é chamada com `call Foo.bar 3` quando `SP = 300`. Quanto valem `ARG` e `LCL` logo depois do `call`? E `SP` depois de `function Foo.bar 2`?"
    O `call` empilha 5 valores: `SP` passa a 305, `ARG = 305 - 5 - 3 = 297` (o primeiro dos três argumentos, empilhados em 297, 298, 299) e `LCL = 305`. `function Foo.bar 2` empilha dois zeros: `SP = 307`.

??? question "3. Depois do `return` dessa mesma chamada, onde está o valor de retorno, e quanto vale `SP`?"
    Em `RAM[297]` (o antigo `ARG`, onde estava o primeiro argumento), e `SP = 298`. Os três argumentos sumiram e um único valor ficou no lugar.

??? question "4. Por que `return` lê o endereço de retorno antes de gravar o valor de retorno em `*ARG`?"
    Porque, numa função chamada com zero argumentos, `ARG` aponta para a mesma célula onde o `call` guardou o endereço de retorno. Gravar o valor primeiro apagaria o endereço.

??? question "5. Por que o rótulo de retorno de cada `call` precisa ser único, se o `return` lê o endereço da pilha?"
    Porque o rótulo é declarado no `.asm` (logo depois do salto) e o Assembler exige que cada rótulo seja declarado uma única vez. Cada `call` volta para um ponto diferente do código, e por isso precisa do seu próprio rótulo — `Foo.main$ret.0`, `Foo.main$ret.1`, ...

## Comparação com convenções de chamada reais

Vale posicionar esse protocolo ao lado de convenções usadas em processadores reais, para enxergar o que é essencial e o que é escolha de projeto:

| Responsabilidade | cdecl (C, x86) | stdcall (Windows) | VM Hack |
|---|---|---|---|
| Empilhar argumentos | chamador | chamador | chamador |
| Descartar argumentos após a chamada | chamador | callee (via `RET n`) | callee (implícito, via `ARG`) |
| Salvar registradores | parcialmente ambos | parcialmente ambos | caller salva todos (no código do `call`); callee restaura (no `return`) |
| Valor de retorno | registrador (`EAX`) | registrador (`EAX`) | pilha (posição `ARG`) |
| Overhead por chamada | variável (poucos registradores movidos) | variável | fixo (49 no `call` + 52 no `return`, na versão mostrada) |

A VM Hack sacrifica desempenho (um overhead fixo e relativamente alto por chamada) por simplicidade extrema de implementação — não há alocação de registradores, não há análise de quais registradores precisam ser preservados: **tudo** é sempre salvo e restaurado, do mesmo jeito, sempre. É uma escolha de projeto coerente com o resto deste curso: corretude e clareza antes de desempenho. E o código gerado para `call`, `function` e `return` não depende de Jack nem do Hack em particular: o mesmo pseudocódigo pode ser realizado em qualquer linguagem de máquina.

## O que vem a seguir

Com o VM Translator completo (capítulos 12 e 13), qualquer programa Jack — via o compilador da Parte 2 — já pode ser traduzido, em duas etapas, até Assembly Hack. Falta uma última peça: transformar esse Assembly em código de máquina de verdade. Os capítulos 14 e 15 fecham essa lacuna: uma consolidação teórica do caminho até aqui e, finalmente, a construção do Assembler.
