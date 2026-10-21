# 14. Geração de código alvo (consolidação teórica)

Antes de mergulhar na especificação de Assembly Hack (capítulo 14) e construir o Assembler (capítulo 15), vale um momento de retrospectiva: o que exatamente aconteceu nos capítulos 11 e 12, e por que essa arquitetura em particular?

## O que já construímos

Percorremos, sem perceber explicitamente, uma tradução em **duas etapas** dentro do que costuma ser chamado, em conjunto, de "geração de código alvo":

```
Código VM (IR, capítulo 7)
    ↓ VM Translator (capítulos 11-12)
Assembly Hack (simbólico, capítulo 14)
    ↓ Assembler (capítulo 15)
Código de máquina Hack (binário)
```

O VM Translator não é, estritamente, "geração de código alvo" no sentido mais estrito do termo (ele traduz entre duas representações textuais, ambas ainda "legíveis" por um humano treinado) — mas cumpre exatamente o papel que a geração de código alvo cumpre em um compilador tradicional: transformar uma representação independente de máquina em uma representação específica de uma arquitetura real. A diferença sutil é que aqui esse trabalho foi dividido em duas sub-etapas (VM → Assembly, Assembly → binário) em vez de uma só (IR → binário direto) — uma divisão que, como vimos no capítulo 6, tem tudo a ver com simplicidade de implementação.

## Por que uma etapa intermediária extra

Um compilador "clássico" de livro-texto costuma ir direto de uma IR para código de máquina binário. O Nand2Tetris insere Assembly Hack como uma parada intermediária **legível por humanos** entre a VM e o binário, por uma razão pedagógica e prática ao mesmo tempo: cada etapa da tradução — VM → Assembly (capítulos 11-12) e Assembly → binário (capítulo 15) — se torna pequena e independentemente verificável. Você pode ler o `.asm` gerado pelo seu VM Translator e confirmar, linha a linha, que ele faz sentido, antes de sequer pensar no Assembler. Compiladores de produção fazem escolhas semelhantes por razões de engenharia: o LLVM, por exemplo, tem uma IR textual (a "LLVM IR") que pode ser inspecionada exatamente por esse motivo — depurar em texto é mais fácil que depurar em binário.

