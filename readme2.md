# Escolha do conversor D/A e do microcontrolador

## Requisitos do projeto

O gerador deve produzir ondas **senoidal**, **triangular**, **dente de serra** e **retangular**, de `1 Hz` a `100 kHz`, com amplitude ajustável até `10 Vpp` e offset de `-5 V` a `+5 V`. A síntese é digital, por *acumulador de fase*, com conversor D/A externo operando a taxa de amostragem fixa.

---

## Necessidades derivadas

### Taxa de amostragem e resolução

A onda **dente de serra** é a mais exigente do conjunto: seus harmônicos decaem como `1/n` e contemplam ordens pares e ímpares, o que obriga a cadeia analógica a passar até `2 MHz`.

Com amostragem em `10 MHz`, as imagens espectrais surgem em `9,9 MHz`, distantes o bastante para que um filtro de 5ª ordem as rejeite em **69 dB**.

> **Nota de Projeto:** O alvo de `69 dB` decorre da resolução escolhida: um conversor de **12 bits** tem piso de ruído de quantização em `74 dB`, e não faria sentido deixar imagens espectrais acima desse piso. ***Taxa de amostragem e resolução são um único requisito expresso em dois eixos.***

Adota-se, portanto, **12 bits a 10 MS/s**.

### Arquitetura de geração

O acumulador de fase é executado em ***software***. Um DMA circular sobre buffer de comprimento variável produziria um sintetizador funcional, mas com resolução de frequência limitada a comprimentos inteiros de buffer, insuficiente para cobrir cinco décadas com ajuste fino.

A arquitetura adotada prevê que:
* A **CPU** calcule as amostras num buffer, com acumulador de `32 bits`;
* O **DMA** transfira esse buffer ao conversor em taxa fixa, disparado por temporizador.

Com *buffer duplo*, enquanto o DMA transmite uma metade, a CPU preenche a outra.

Disso decorre um requisito que **não é evidente**: não basta que o DMA sustente a vazão. A CPU precisa produzir as amostras continuamente, ao custo de aproximadamente **6 ciclos por amostra** em núcleo de 32 bits, o que representa `60 MHz` de carga contínua a 10 MS/s[cite: 2].

Somam-se a isso **80 kB de memória**, dos quais `64 kB` correspondem à tabela de onda em buffer duplo[cite: 2]. O buffer duplo é exigido pelo *ajuste de simetria das rampas*: alterar a simetria significa reescrever a tabela enquanto ela está sendo lida, o que produziria uma onda híbrida durante a transição[cite: 2].

### Ajuste de amplitude e offset

Estes ajustes não passam pelo conversor de síntese, que trabalha **sempre em escala completa**[cite: 2]. Escalar os números da tabela para reduzir amplitude custaria resolução efetiva, e a qualidade passaria a depender da amplitude selecionada[cite: 2].

* **Offset:** Conversor de saída em tensão, somada ao sinal no nó de terra virtual do estágio final[cite: 2]. Resolvido por um dos conversores internos do microcontrolador[cite: 2].
* **Amplitude:** **Conversor multiplicador (MDAC)** na malha de realimentação do estágio de ganho, comandado por *código digital*[cite: 2]. Atua como rede resistiva variável, preservando precisão e mantendo a distorção desprezível no caminho do sinal[cite: 2].

---

## Escolha do conversor D/A de síntese

### Requisitos

| Requisito | Justificativa |
| :--- | :--- |
| **12 bits** | Piso de ruído em `74 dB`, contra `50 dB` de um conversor de 8 bits[cite: 2]. |
| **≥ 10 MS/s** | Distância até a primeira imagem espectral[cite: 2]. |
| **Entrada paralela** | Transferência por DMA a partir de uma porta GPIO[cite: 2]. |
| **Registrador de entrada com clock** | Ver justificativa abaixo[cite: 2]. |
| **Arquitetura segmentada** | Contenção da energia de *glitch*[cite: 2]. |
| **Saída em corrente** | Conversão I-V com terra virtual[cite: 2]. |

O **registrador de entrada** é o requisito menos óbvio e o mais decisivo[cite: 2]. Sem ele, o conversor é *transparente*: a saída acompanha continuamente os pinos de entrada[cite: 2]. Quando o DMA escreve um novo valor na porta GPIO, os bits não comutam exatamente no mesmo instante, e durante essa janela de alguns nanossegundos o conversor produz **valores intermediários espúrios**[cite: 2].

> **Atenção:** Isso pesa especialmente na dente de serra, cujo retorno é um **salto de escala completa síncrono com o sinal**, uma vez por ciclo[cite: 2]. *Glitch* síncrono não se comporta como ruído: é distorção correlacionada, que aparece como raia fixa no espectro e que o filtro de reconstrução **não remove**, por estar dentro da banda passante[cite: 2].

### Caracterização da classe de conversores

Os componentes adequados para esta função pertencem a uma classe de conversores paralelos rápidos projetados para síntese digital direta (DDS). Eles são caracterizados por apresentarem *latch* de entrada acionado por borda de clock, arquitetura interna segmentada para mitigação de *glitch*, saídas diferenciais em corrente (tipicamente na faixa de 2 a 20 mA), referência interna de tensão integrada, operação com fonte única de alimentação de 2,7 V a 5,5 V e taxas máximas de amostragem situadas entre 125 e 165 MS/s.

### Componentes avaliados

| Conversor | Características | Avaliação |
| :--- | :--- | :--- |
| **AD5689** | 2 canais, 16 bits, saída em tensão, comunicação SPI [2][cite: 2] | ❌ ***Descartado.*** A atualização serial e o tempo de acomodação limitam a taxa a valores muito abaixo dos 10 MS/s[cite: 2]. A resolução de 16 bits favorece a precisão, mas não compensa a limitação de velocidade[cite: 2]. |
| **DAC0808** | 8 bits, entrada paralela, saída em corrente, acomodação típica de 150 ns [3][cite: 2] | ❌ ***Descartado.*** Resolução de `50 dB` contra os `74 dB` necessários, ausência de registrador de entrada, *glitch* não especificado, e acomodação típica de `150 ns` incompatível com o período de `100 ns` — valor sem máximo garantido[cite: 2]. |
| **DAC902** | 12 bits, 165 MS/s, *latch* por borda, arquitetura segmentada, *glitch* de 3 pV·s, saída diferencial em corrente, referência interna, alimentação única | ✅ ***Escolhido.*** |

### Vantagens do conversor escolhido

* **Alimentação simples:** Fonte única, sem trilho negativo dedicado ao conversor[cite: 2].
* **Sem deslocamento de nível:** O trilho digital pode ser `3,3 V`, ligado diretamente ao microcontrolador[cite: 2].
* **Menor acoplamento digital:** Operar com excursão lógica reduzida diminui o *feedthrough* de dados e o ruído interno do conversor[cite: 2].
* **Margem de taxa:** Os `165 MS/s` disponíveis contra os `10 MS/s` exigidos eliminam a acomodação como fator limitante[cite: 2].

> *Nota de Montagem:* Nenhum componente dessa classe está disponível em encapsulamento DIP[cite: 2]. Para montagem manual, prefira **SOIC** (passo de `1,27 mm`) a **TSSOP** (`0,65 mm`)[cite: 2].

---

## Escolha do microcontrolador

| Microcontrolador | Recursos relevantes | Avaliação |
| :--- | :--- | :--- |
| **dsPIC33CK** | 100 MHz, 16 bits, 24 kB de RAM, 3 conversores e 4 comparadores internos [4][cite: 2] | ❌ ***Descartado.*** Memória insuficiente contra os 80 kB necessários, e o núcleo de 16 bits eleva o acumulador a ~11 ciclos/amostra, ultrapassando 100% de carga[cite: 2]. |
| **ESP32-S3** | Dois núcleos a 240 MHz, GDMA e interface LCD paralela [5][cite: 2] | ❌ ***Descartado.*** Atende à vazão com *25% de carga*, mas não possui periferia analógica, tem alimentação restrita e traz um **transceptor de rádio no mesmo encapsulamento**, a centímetros do conversor[cite: 2]. |
| **RP2350** | Dois Cortex-M33 a 150 MHz, DMA, máquinas PIO e interpoladores [6][cite: 2] | ❌ ***Descartado.*** **Melhor determinismo do grupo**, reduzindo carga a ~*20%*[cite: 2]. Porém, sem conversores/comparadores, exige cerca de **cinco CIs adicionais**[cite: 2]. |
| **STM32H743** | Cortex-M7 a 480 MHz, 1 MB de RAM, LTDC e DMA2D [7][cite: 2] | ❌ ***Descartado.*** Folga confortável (*12% de carga*) e **aceleração gráfica**[cite: 2]. Contudo, possui apenas 2 conversores/comparadores, múltiplos domínios de VDD e gestão complexa de *coerência de cache*[cite: 2]. |
| **STM32G474** | Cortex-M4F a 170 MHz, 128 kB, 7 conversores, 7 comparadores e HRTIM de 184 ps [8][cite: 2] | ✅ ***Escolhido.*** Melhor periferia analógica do grupo e domínio de alimentação único, com *35% de carga*[cite: 2]. |

> **Aviso de Bancada:** As taxas de saída paralela são **estimativas de arquitetura**, não valores de folha de dados[cite: 2]. Nenhum fabricante especifica taxa máxima de saída paralela para síntese de forma de onda[cite: 2]. Todas exigem *confirmação em bancada* com analisador lógico[cite: 2].

### Justificativa da escolha

O **STM32G474** é o único candidato projetado explicitamente para aplicações de *sinal misto*[cite: 2].

1. Seus **7 conversores** e **7 comparadores** internos resolvem o **offset**, os limiares da onda retangular e deixam canais livres para proteção[cite: 2]. Isso elimina componentes externos[cite: 2].
2. O **domínio de alimentação único** é consideravelmente mais simples de projetar[cite: 2].
3. A **ausência de cache** elimina erros de coerência em buffers de DMA[cite: 2].
4. O temporizador de alta resolução de `184 ps` oferece resolução de ciclo de trabalho **onze vezes superior** à do H743[cite: 2].

*A escolha não indica incapacidade das alternativas.*[cite: 2] O **RP2350** é forte para geração pura, e o **STM32H743** seria a escolha natural caso fosse exigida interface gráfica colorida com *framebuffer*[cite: 2].

### Ressalvas Críticas

1. **A taxa de saída fica no limite inferior da estimativa:** A faixa de `10–15 MS/s` coincide no limite inferior com o requisito de `10 MS/s` (margem nula no pior caso)[cite: 2]. Este é o **primeiro item a medir no protótipo**[cite: 2].
2. **A referência interna não basta para um instrumento calibrado:** O *VREFBUF* atende ao erro de amplitude de `5%`, mas sua deriva térmica e ruído exigirão uma **referência dedicada externa** para erros abaixo de `1%`[cite: 2].

---

## Implementação prevista

O fluxo do sinal e processamento pretendido é:

```text
Tabela de Onda
  ↳ CPU (Acumulador de Fase)
    ↳ Buffer Duplo em Memória
      ↳ DMA disparado por temporizador
        ↳ Porta GPIO de 12 bits
          ↳ Conversor D/A
            ↳ Conversão I-V
              ↳ Filtro de Reconstrução
                ↳ Chave Seletora
                  ↳ Ganho (MDAC)
                    ↳ Offset
                      ↳ Estágio de Saída
