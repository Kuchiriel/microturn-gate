# microturn-gate — classificador de turn-taking para voz em português

LoRA de 4 camadas (r16) para **Qwen/Qwen3.5-4B** que classifica cada fala do
usuário em `RESPOND | WAIT | SILENCE | BARGE`. Nasceu dentro do JARVIS duplex
(full-duplex, PT-BR) e foi extraído para ser reutilizável por qualquer harness
de voz — OpenCode, pi, llama.cpp ou o que vier.

> Publicado em 17/11/2026. **Nada de segredo aqui**: só pesos, prompt, métricas
> e o script de conversão. O código original permanece em `nixos-ai`.

## Por que existe

Num assistente full-duplex, o problema difícil não é transcrever nem sintetizar
— é **decidir quando falar**. Regras hardcoded (`eh_pergunta`, listas de
backchannel) quebram com TV ao fundo, frases incompletas e interrupções. Um
classificador minúsculo, ligado só no momento da decisão (7 ms para togglar o
adaptador no llama.cpp), resolve com latência de 132–587 ms em CPU.

## O que medimos (honesto)

Fonte: `nixos-ai/docs/DUPLEX-LAYAS-MEASUREMENTS.md` e handoffs de 12–13/11.

| versão | held-out | resultado | latência |
|---|---|---|---|
| v21 original (r16 f32, 84.9 MB) | 27 falas PT-BR | recall 1.00, precision 0.65, **F1 0.79**, 7 FP | 252–587 ms |
| gate v3 (LoRA + regras) | 52 falas, sem sobreposição com treino | **40/52** (SILENCE 4/16 → 7/16; RESPOND 16/16; BARGE 7/7; WAIT 10/13) | ~149 ms |
| baseline sem gate | mesmas 52 | 37/52 | — |
| v3 em GGUF q8_0 | conversão local | 3.06 MB | classify ~132 ms |

**Limitação conhecida:** o benchmark de 27 falas sobrepõe com o template de
treino — recall 1.00 ali é memorização, não generalização. O número que vale
é o held-out de 52. Mesmo assim, SILENCE ainda erra (7/16 acertos no v3):
fala completa não dirigida ao assistente é confundida com RESPOND.

`metrics.json` incluído aqui mede um conjunto menor (16 falas, acc 0.5625,
BARGE 0/3) — é um diagnóstico de classe, não o número de发布. Use o held-out.

## Como usar

```bash
# llama.cpp com o modelo base carregado
llama-server -m Qwen3.5-4B.gguf \
  --lora microturn-4b-lora-v9.gguf --lora-init-without-apply

# No momento de classificar: liga o adaptador, /completion com CLASS_SYS,
# desliga. O chat normal continua com o modelo base, sem custo de estilo.
```

O prompt é **byte-for-byte** o do treino (não mude uma vírgula):

```text
Você é o JARVIS, assistente de voz. Classifique o turno do usuário em uma
palavra: RESPOND (fala completa que pede resposta), WAIT (frase incompleta,
mais vem), SILENCE (backchannel/ruído ou fala completa NÃO dirigida ao
assistente, como TV/narração/música), ou BARGE (o dono interrompeu sua fala).
```

Saída restrita por gramática: `RESPOND | WAIT | SILENCE | BARGE`.

## Arquivos

| arquivo | o que é |
|---|---|
| `adapter/adapter_model.safetensors` | adapter PEFT r16 (82 MB), treino original |
| `adapter/adapter_config.json` | config PEFT |
| `microturn-4b-lora.gguf` | conversão q8_0 completa (82 MB) |
| `microturn-4b-lora-v9.gguf` | versão compacta (3 MB), a recomendada |
| `metrics.json` | matriz de confusão do conjunto de 16 falas |
| `convert_lora_to_gguf.sh` | conversão local (sem Kaggle) |

## Licença

Apache-2.0. O modelo base Qwen3.5-4B é da Qwen (consulte a licença do base);
este adaptador e o prompt são da casa e podem ser usados, modificados e
redistribuídos com atribuição.

## Contribuições bem-vindas

1. **SILENCE**: gravar 100+ falas de TV/rádio/narração rotuladas — é a classe fraca.
2. **BARGE**: só 177 exemplos únicos no treino; falta diversidade real.
3. **Held-out fresco**: 24 itens sem vazamento (`bench_heldout2.py` no repo original).
4. **Port**: rodar o v9 em `llama.cpp` android-arm64 (Cactus/Needle é o paradigma).

Não abra PR de "melhor prompt" sem medir no held-out: o lab da casa provou que
teste inválido gera 537 medições sem sinal.
