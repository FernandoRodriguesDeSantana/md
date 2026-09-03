# Escolha do conversor D/A e do microcontrolador

## Requisitos do projeto

O gerador deve produzir ondas **senoidal**, **triangular**, **dente de serra** e **retangular**, de `1 Hz` a `100 kHz`, com amplitude ajustável até `10 Vpp` e offset de `-5 V` a `+5 V`. A síntese é digital, por *acumulador de fase*, com conversor D/A externo de **12 bits** operando a taxa de amostragem fixa.

---

## Necessidades derivadas

### Taxa de amostragem e resolução

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

### Conversores de controle

O ajuste de **offset** é feito por um conversor de saída em **tensão**, somada ao sinal no nó de terra virtual do estágio final. Pode ser interno ao microcontrolador.

O ajuste de **amplitude** exige um **conversor multiplicador (MDAC)** na malha de realimentação do estágio de ganho, comandado por *código digital*. Ele funciona como rede resistiva variável, com precisão de conversor e distorção desprezível. Um conversor de saída em tensão **não serve** para essa função, pois exigiria um multiplicador analógico ou *VCA* no caminho do sinal, com distorção incompatível com a especificação de `THD < 1%`.

---

## Escolha do conversor D/A

### Requisitos

| Requisito | Justificativa |
|---|---|
| **12 bits** | Piso de ruído em `74 dB`, contra `50 dB` de um conversor de 8 bits |
| **≥ 10 MS/s** | Distância até a primeira imagem espectral |
| **Entrada paralela** | Transferência por DMA a partir de uma porta GPIO |
| **Registrador de entrada com clock** | Ver justificativa abaixo |
| **Arquitetura segmentada** | Contenção da energia de *glitch* |
| **Saída em corrente** | Conversão I-V com terra virtual |

O **registrador de entrada** é o requisito menos óbvio e o mais decisivo. Sem ele, o conversor é *transparente*: a saída acompanha continuamente os pinos de entrada. Quando o DMA escreve um novo valor na porta GPIO, os bits não comutam exatamente no mesmo instante, e durante essa janela de alguns nanossegundos o conversor produz **valores intermediários espúrios**.

> Isso pesa especialmente na dente de serra, cujo retorno é um **salto de escala completa síncrono com o sinal**, uma vez por ciclo. *Glitch* síncrono não se comporta como ruído: é distorção correlacionada, que aparece como raia fixa no espectro e que o filtro de reconstrução **não remove**, por estar dentro da banda passante.

### Componentes avaliados

| Conversor | Características | Avaliação |
|---|---|---|
| **AD5689** | 2 canais, 16 bits, saída em tensão, comunicação SPI [2] | Atualização serial e tempo de acomodação o excluem do caminho de síntese. ***Adotado para o controle de offset***, onde a exigência é precisão e não velocidade; o segundo canal fica disponível para limiar de comparador |
| **DAC0808** | 8 bits, entrada paralela, saída em corrente, acomodação típica de 150 ns [3] | ***Não atende***: resolução insuficiente, sem registrador de entrada, sem *glitch* especificado, e a acomodação típica de `150 ns` não é compatível com o período de `100 ns`. **Adotado na versão preliminar** de bancada |
| **Conversor de síntese de 12 bits** | Entrada paralela com *latch* por borda, arquitetura segmentada, saídas diferenciais em corrente de 2 a 20 mA, referência interna, alimentação única de 2,7 a 5,5 V | ***Adotado na versão final*** |

### Versão preliminar com o DAC0808

O DAC0808 permanece útil como **etapa de validação**. Uma versão limitada a `20 kHz`, com amostragem em `3 MS/s`, permite construir e depurar toda a cadeia analógica — a parte difícil e demorada — sem lidar simultaneamente com barramento de alta velocidade, montagem em encapsulamento SMD e integridade de sinal:

```
fs = 3 MS/s,  fmáx = 20 kHz,  corte do filtro = 400 kHz
Primeira imagem: 2,98 MHz  →  2,90 oitavas
Filtro de 4ª ordem: 69,5 dB de rejeição
```

A margem de acomodação justifica os `3 MS/s` em vez dos `5 MS/s` de uma estimativa direta: o período de `200 ns` a 5 MS/s oferece apenas **1,33 vezes** o tempo típico de 150 ns, sobre um parâmetro que o fabricante *não garante com valor máximo*. Um critério defensável é de 2,5 a 3 vezes.

> Nessa versão a ausência de registrador de entrada persiste. Ou se acrescenta um ***latch* externo de 8 bits** comandado pelo mesmo sinal que dispara o DMA, ou a degradação é aceita e documentada.

### Vantagens do conversor da versão final

- **Alimentação mais simples.** Fonte única, sem o trilho negativo que o DAC0808 exige.
- **Sem deslocamento de nível.** O trilho digital pode ser `3,3 V`, ligado diretamente ao microcontrolador.
- **Menor acoplamento digital.** Operar com excursão lógica reduzida diminui o *feedthrough* de dados e o ruído interno do conversor.

Nenhum desses componentes está disponível em encapsulamento DIP. Para montagem manual, prefira **SOIC** (passo de `1,27 mm`) a **TSSOP** (`0,65 mm`).

---

## Escolha do microcontrolador

| Microcontrolador | Recursos relevantes | Avaliação |
|---|---|---|
| **dsPIC33CK** | 100 MHz, 16 bits, 24 kB de RAM, 3 conversores e 4 comparadores internos [4] | ***Descartado.*** Memória insuficiente contra os 80 kB necessários, e o núcleo de 16 bits eleva o acumulador de 32 bits a cerca de **11 ciclos por amostra**, ultrapassando 100% de carga |
| **ESP32-S3** | Dois núcleos a 240 MHz, GDMA e interface LCD paralela [5] | Atende à vazão com *25% de carga*, mas não possui periferia analógica, tem alimentação restrita e traz um **transceptor de rádio no mesmo encapsulamento**, a centímetros do conversor |
| **RP2350** | Dois Cortex-M33 a 150 MHz, DMA, máquinas PIO e interpoladores [6] | **Melhor determinismo do grupo:** o PIO gera o clock de amostragem em hardware e os interpoladores reduzem a carga a cerca de *20%*. Sem conversores, comparadores ou referência interna, exige cerca de **cinco CIs adicionais** |
| **STM32H743** | Cortex-M7 a 480 MHz, 1 MB de RAM, LTDC e DMA2D [7] | Folga confortável, com *12% de carga*, e única opção com **aceleração gráfica dedicada**. Em contrapartida, apenas 2 conversores e 2 comparadores internos, alimentação com múltiplos domínios e necessidade de gerenciar *coerência de cache* nos buffers de DMA |
| **STM32G474** | Cortex-M4F a 170 MHz, 128 kB, 7 conversores, 7 comparadores e temporizador de alta resolução de 184 ps [8] | ***Escolhido.*** Melhor periferia analógica do grupo e domínio de alimentação único, com *35% de carga* |

> ⚠️ As taxas de saída paralela são **estimativas de arquitetura**, não valores de folha de dados. Nenhum fabricante especifica taxa máxima de saída paralela para síntese de forma de onda. Todas exigem *confirmação em bancada* com analisador lógico.

### Escolha: STM32G474

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
  → ganho (MDAC)
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
