# Palmas duplas → luzes (ESP32 + INMP441 + ESPHome + Home Assistant)

Duas palmas seguidas acendem ou apagam as luzes. A detecção roda **dentro do ESP32**
(funciona sem internet); o Home Assistant só recebe o pulso "palmas duplas" e decide
o que ligar.

> **EN:** double-clap detector for an ESP32 with an I2S INMP441 microphone, written
> as an ESPHome config. It exposes a `Palmas Duplas` (double clap) binary sensor to
> Home Assistant; an example automation toggles your lights. Comments in the code
> are in Brazilian Portuguese.

## O que precisa

- ESP32 (testado em ESP32 DevKit, `esp32dev`)
- Microfone INMP441 (I2S)
- ESPHome 2026.8 ou mais novo (usa o componente `sound_level` e `microphone.on_data`)
- Home Assistant com a integração ESPHome

## Ligação

| INMP441 | ESP32  |
|---------|--------|
| VDD     | 3V3    |
| GND     | GND    |
| L/R     | GND    |
| WS      | GPIO26 |
| SCK     | GPIO27 |
| SD      | GPIO33 |

L/R no GND escolhe o canal esquerdo (é o que o YAML usa). Para outros pinos, mude o
bloco `i2s_audio` / `microphone` em `esphome/palmas.yaml`.

## Instalação

1. Copie `esphome/secrets.yaml.example` para `esphome/secrets.yaml` e preencha Wi-Fi e chave da API.
2. Grave o firmware: `esphome run esphome/palmas.yaml`.
3. No Home Assistant, adote o dispositivo "Palmas ESP32".
4. Cole `homeassistant/automacao.yaml` numa automação nova e troque as entidades `light.*` pelas suas. Confira o nome do `binary_sensor` do dispositivo.

## Entidades que aparecem no HA

| Entidade | Para quê |
|----------|----------|
| `binary_sensor` Palmas Duplas | pulsa 600 ms a cada par de palmas — é o gatilho |
| `number` Palmas: Limiar (dB acima do fundo) | sensibilidade. Padrão 18. Mais alto = menos sensível. Alteração vale na hora, sem regravar |
| `switch` Palmas: Detector Ativo | liga/desliga o detector sem regravar |
| `sensor` Palmas: Nível de Fundo | o fundo que o detector está aprendendo (útil para ajustar o limiar) |
| `sensor` Nível de Som (dBFS) | nível do microfone, 1 leitura por segundo |

## Como funciona (e por que não é só "som alto")

A versão ingênua (RMS de banda larga acima de um limiar) dispara sozinha. Medindo o
áudio cru do microfone, as causas foram:

1. **Ar/vento/pressão de 0,5–5 Hz** no microfone domina o RMS e vira "palma". Correção:
   passa-alta Butterworth de 2ª ordem em 100 Hz, aplicado antes do RMS.
2. **Voz e ruído grave** (100–500 Hz) com pausa curta viravam "palma dupla". Correção:
   um segundo passa-alta em 700 Hz e uma *regra de forma*: palma tem energia espalhada
   acima de 700 Hz, voz não. O detector só considera "alto" se
   `RMS(700 Hz) − RMS(100 Hz) > −6 dB` (palmas medidas: −2,2 a −0,1 dB; resto, mediana −15 dB).
3. **Som contínuo** (música, TV) não pode armar o detector: se o som passa de 250 ms,
   o estado vai para "espera acabar" e só rearma depois de um bloco silencioso.

Resto da lógica: fundo adaptativo (média móvel, congelada durante transientes),
limiar relativo ao fundo (18 dB), segunda palma entre 200 ms e 900 ms depois da
primeira, carência de 3 s entre disparos.

## Limites conhecidos

- **Estalo de interruptor, porta batendo, clique duplo** têm formato de palma e podem disparar.
- A margem entre "palma longe do microfone" e "digitação" é de uns 3 dB nos dois lados.
  Calibrado com palmas na mesma sala do microfone (perto e longe); palma de outro
  cômodo não foi testada. Se não pegar, baixe o limiar (14 foi o menor valor testado).
- Os coeficientes dos filtros são para **16 kHz**. Se o seu microfone/ESPHome usar outra
  taxa, o detector não funciona (retorna sem calcular) e o log avisa:
  `passa-alta: microfone N bits, N canal(is), N Hz`.
- Dois pares de palmas em menos de 3 s: o segundo par é ignorado.

## Licença

MIT — ver [LICENSE](LICENSE).
