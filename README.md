# Escolha do microcontrolador

## Requisitos do projeto

O gerador deve produzir ondas **senoidal**, **triangular**, **dente de serra** e **retangular**, de `1 Hz` a `100 kHz`, com amplitude ajustável até `10 Vpp` e offset de `-5 V` a `+5 V`. A síntese é digital, por *acumulador de fase*, com conversor D/A externo de **12 bits** operando a taxa de amostragem fixa.

---

## Necessidades derivadas

A taxa de amostragem de **10 MS/s** decorre do espectro da onda dente de serra, a mais exigente do conjunto: seus harmônicos decaem como `1/n` e exigem banda até `2 MHz`. Com amostragem em `10 MHz`, as imagens espectrais ficam em `9,9 MHz`, distantes o bastante para que um filtro de 5ª ordem as rejeite em **69 dB**.

> Esse alvo de rejeição decorre da resolução escolhida: um conversor de 12 bits tem piso de ruído de quantização em `74 dB`, e não faria sentido deixar imagens espectrais acima desse piso. **Taxa de amostragem e resolução são um único requisito expresso em dois eixos.**

### Arquitetura de geração

O acumulador de fase é executado em ***software***, não pelo DMA. Um DMA circular sobre buffer de comprimento variável produz um sintetizador funcional, mas com resolução de frequência limitada a comprimentos inteiros de buffer, o que é insuficiente para cobrir cinco décadas com ajuste fino.

A arquitetura adotada prevê que:

- a **CPU** calcule as amostras num buffer, com acumulador de `32 bits`;
- o **DMA** transfira esse buffer ao conversor em taxa fixa, disparado por temporizador.

Com *buffer duplo*, enquanto o DMA transmite uma metade, a CPU preenche a outra.

Dessa arquitetura decorre um requisito que **não é evidente**: não basta que o DMA sustente a vazão. A CPU precisa produzir as amostras continuamente, ao custo de aproximadamente **6 ciclos por amostra** em núcleo de 32 bits, o que representa `60 MHz` de carga contínua a 10 MS/s.

Somam-se a isso **80 kB de memória**, dos quais `64 kB` correspondem à tabela de onda em buffer duplo. O buffer duplo é exigido pelo *ajuste de simetria das rampas*: alterar a simetria significa reescrever a tabela enquanto ela está sendo lida, o que produziria uma onda híbrida durante a transição.

---

## Correção de topologia nos conversores de controle

Versões anteriores desta análise previam **dois conversores de controle**, para amplitude e offset. Isso está correto para o offset, mas ***não*** para a amplitude.

Um conversor de controle produz **tensão**. Usá-la para ajustar amplitude exigiria um multiplicador analógico ou *VCA* no caminho do sinal, com distorção incompatível com a especificação de `THD < 1%`.

A topologia adequada é um **conversor multiplicador (MDAC)** na malha de realimentação do estágio de ganho, comandado por *código digital*, funcionando como rede resistiva variável com precisão de conversor e distorção desprezível.

**Consequência:** o projeto passa a exigir *um* conversor de controle por tensão para o offset — que pode ser interno ao microcontrolador — e *um* conversor multiplicador externo para a amplitude.

---

## Opções avaliadas

| Microcontrolador | Recursos relevantes | Avaliação |
|---|---|---|
| **dsPIC33CK** | 100 MHz, 16 bits, 24 kB de RAM, 3 conversores e 4 comparadores internos [4] | ***Descartado.*** Memória insuficiente contra os 80 kB necessários, e o núcleo de 16 bits eleva o acumulador de 32 bits a cerca de **11 ciclos por amostra**, ultrapassando 100% de carga |
| **ESP32-S3** | Dois núcleos a 240 MHz, GDMA e interface LCD paralela [5] | Atende à vazão com *25% de carga*, mas não possui periferia analógica, tem alimentação restrita e traz um **transceptor de rádio no mesmo encapsulamento**, a centímetros do conversor |
| **RP2350** | Dois Cortex-M33 a 150 MHz, DMA, máquinas PIO e interpoladores [6] | **Melhor determinismo do grupo:** o PIO gera o clock de amostragem em hardware e os interpoladores reduzem a carga a cerca de *20%*. Sem conversores, comparadores ou referência interna, exige cerca de **cinco CIs adicionais** |
| **STM32H743** | Cortex-M7 a 480 MHz, 1 MB de RAM, LTDC e DMA2D [7] | Folga confortável, com *12% de carga*, e única opção com **aceleração gráfica dedicada**. Em contrapartida, apenas 2 conversores e 2 comparadores internos, alimentação com múltiplos domínios e necessidade de gerenciar *coerência de cache* nos buffers de DMA |
| **STM32G474** | Cortex-M4F a 170 MHz, 128 kB, 7 conversores, 7 comparadores e temporizador de alta resolução de 184 ps [8] | ***Escolhido.*** Melhor periferia analógica do grupo e domínio de alimentação único, com *35% de carga* |

> ⚠️ As taxas de saída paralela são **estimativas de arquitetura**, não valores de folha de dados. Nenhum fabricante especifica taxa máxima de saída paralela para síntese de forma de onda. Todas exigem *confirmação em bancada* com analisador lógico.

---

## Escolha: STM32G474

O **STM32G474** foi selecionado por ser o único candidato projetado explicitamente para aplicações de *sinal misto*.

- Seus **7 conversores** e **7 comparadores** internos, contra 2 e 2 do STM32H743, resolvem o offset, os limiares do caminho da onda retangular e ainda deixam canais livres para proteção e detecção de sobrecarga.
- O **domínio de alimentação único** é consideravelmente mais simples de projetar que os múltiplos domínios com sequenciamento do H743.
- A **ausência de cache** elimina uma classe inteira de erros de coerência em buffers de DMA.
- O temporizador de alta resolução de `184 ps` — presente nas variantes **G474** e **G484**, mas *não* em toda a família G4 — oferece resolução de ciclo de trabalho cerca de **onze vezes superior** à do periférico equivalente no H743.

A escolha ***não*** indica incapacidade das alternativas. O **RP2350** é uma opção forte para a geração pura, e o **STM32H743** seria a escolha natural caso o projeto passe a exigir interface gráfica colorida com *framebuffer*, para a qual os 128 kB do G474 não deixam folga.

### Ressalvas

**1. A taxa de saída fica no limite inferior da estimativa.**
A faixa de `10–15 MS/s` coincide no limite inferior com o requisito de `10 MS/s`, o que significa ***margem nula*** no pior caso da estimativa. Este é o **primeiro item a medir no protótipo**.

**2. A referência interna não basta para um instrumento calibrado.**
O *VREFBUF* atende ao erro de amplitude de `5%` previsto, mas sua deriva térmica e seu ruído são insuficientes para erro abaixo de `1%`, caso em que uma **referência dedicada externa** será necessária.

---

## Implementação prevista

O fluxo pretendido é:

```
tabela de onda
  → CPU (acumulador de fase)
  → buffer duplo em memória
  → DMA disparado por temporizador
  → porta GPIO de 12 bits
  → conversor D/A
  → conversão corrente-tensão
  → filtro de reconstrução
  → chave seletora
  → ganho
  → offset
  → estágio de saída
```

Os **doze bits** deverão ocupar preferencialmente uma **única porta GPIO contígua**.

Pontos ainda a definir:

- [ ] Verificar a rota entre temporizador, controlador DMA, barramento e porta escolhida
- [ ] Definir os pinos
- [ ] Validar experimentalmente os `10 MS/s` com analisador lógico, ***antes*** de qualquer montagem analógica

---

## Um requisito que condiciona o layout

A rejeição de `69 dB` atribuída ao filtro trata **apenas das imagens de amostragem**, que são fenômeno do domínio do sinal. Ela ***não*** protege contra **acoplamento direto do barramento digital** para a seção analógica.

Um barramento paralelo de 12 bits comutando a `10 MHz` tem conteúdo espectral que se estende a **centenas de megahertz**. Esse conteúdo chega à parte analógica por *capacitância parasita entre trilhas* e por *acoplamento nos laços de retorno*, **contornando o filtro**.

> Em placas de síntese digital, este mecanismo costuma ser o **espúrio dominante**.

Isso não altera a escolha do microcontrolador, mas eleva o layout de *cuidado de acabamento* a **requisito de projeto**, no mesmo nível da ordem do filtro:

1. **Resistores série de 33 a 47 Ω** em cada linha do barramento de dados
2. **Comprimento mínimo** entre microcontrolador e conversor
3. **Plano de terra contínuo, sem cortes**, com separação feita por *posicionamento físico* e o conversor na fronteira entre os domínios
