# Apêndice C — Especificação do Assembly Hack

Referência compacta, extraída do capítulo 11, para consulta rápida durante o VM Translator (capítulos 12 e 13) e o Assembler (capítulo 15).

## Formato binário

```
A-instruction:  0 vvvvvvvvvvvvvvv                     (v = valor de 15 bits)
C-instruction:  1 1 1 a c1 c2 c3 c4 c5 c6 d1 d2 d3 j1 j2 j3
```

## `comp` (7 bits, incluindo o bit `a`)

| `comp` | `a c1..c6` | `comp` | `a c1..c6` |
|---|---|---|---|
| `0` | `0101010` | `M` | `1110000` |
| `1` | `0111111` | `!M` | `1110001` |
| `-1` | `0111010` | `-M` | `1110011` |
| `D` | `0001100` | `M+1` | `1110111` |
| `A` | `0110000` | `M-1` | `1110010` |
| `!D` | `0001101` | `D+M` | `1000010` |
| `!A` | `0110001` | `D-M` | `1010011` |
| `-D` | `0001111` | `M-D` | `1000111` |
| `-A` | `0110011` | `D&M` | `1000000` |
| `D+1` | `0011111` | `D\|M` | `1010101` |
| `A+1` | `0110111` | | |
| `D-1` | `0001110` | | |
| `A-1` | `0110010` | | |
| `D+A` | `0000010` | | |
| `D-A` | `0010011` | | |
| `A-D` | `0000111` | | |
| `D&A` | `0000000` | | |
| `D\|A` | `0010101` | | |

## `dest` (3 bits)

| Mnemônico | Bits | Armazena em |
|---|---|---|
| (vazio) | `000` | nenhum lugar |
| `M` | `001` | `RAM[A]` |
| `D` | `010` | registrador `D` |
| `MD` | `011` | `RAM[A]` e `D` |
| `A` | `100` | registrador `A` |
| `AM` | `101` | `A` e `RAM[A]` |
| `AD` | `110` | `A` e `D` |
| `AMD` | `111` | `A`, `D` e `RAM[A]` |

## `jump` (3 bits)

| Mnemônico | Bits | Salta se |
|---|---|---|
| (vazio) | `000` | nunca |
| `JGT` | `001` | `comp > 0` |
| `JEQ` | `010` | `comp == 0` |
| `JGE` | `011` | `comp >= 0` |
| `JLT` | `100` | `comp < 0` |
| `JNE` | `101` | `comp != 0` |
| `JLE` | `110` | `comp <= 0` |
| `JMP` | `111` | sempre |

## Símbolos pré-definidos

| Símbolo | Endereço RAM |
|---|---|
| `SP` | 0 |
| `LCL` | 1 |
| `ARG` | 2 |
| `THIS` | 3 |
| `THAT` | 4 |
| `R0`–`R15` | 0–15 |
| `SCREEN` | 16384 |
| `KBD` | 24576 |

Variáveis definidas pelo usuário recebem endereços a partir de `RAM[16]`, na ordem em que são encontradas pelo Assembler (capítulo 15).
