# Estudo: cadência de vídeo

**Problema histórico, corrigido entre as versões 4.2.3 e 4.4.**

## Sintoma

Em transmissões de uma tela 1440p, a imagem parecia travada e pouco fluida no viewer. Ao mesmo tempo, o encoder reportava cerca de 60 FPS. Pela métrica disponível, não havia nada errado.

## Medição

O FPS de saída do encoder conta quadros emitidos, não quadros diferentes. Para medir o que o viewer realmente via, foi usado um padrão sintético de 2560x1440 com marcadores nos quatro cantos e um contador que muda a cada quadro. O stream publicado localmente foi decodificado e o número de transições do contador foi contado.

| | Quadros analisados | Transições do contador | Mudanças reais por segundo | FPS reportado |
|---|---|---|---|---|
| Antes | 360 | 132 | cerca de 22 | cerca de 60 |
| Depois | 360 | 358 | cerca de 58,5 | 58,75 |

Os marcadores dos cantos verificavam ao mesmo tempo outro problema: se a escala da imagem cortava as bordas da tela.

## Causa

Foram dois fatores combinados:

1. A captura tinha caído do DXGI para o GDI, mais lento. Um filtro do FFmpeg que força taxa constante de quadros completava os buracos repetindo o quadro anterior. O encoder recebia 60 quadros por segundo, mas a maioria era cópia.
2. Vídeo e áudio passavam por um único grafo de filtros. Com isso, o agendamento do vídeo ficava acoplado à chegada do áudio. Testes isolados mostraram que captura e escala, sozinhas, preservavam o movimento.

## Correção

- Grafos de filtro separados para vídeo e áudio, com a mesma referência de tempo.
- Fim do preenchimento artificial de quadros: o stream passou a carregar só os quadros realmente capturados, com timestamps em microssegundos.
- Escala proporcional com bordas, sem cortar a imagem, e reconhecimento correto de DPI por monitor.
- Regra de admissão: um encoder só é aceito se sustentar pelo menos 95% da cadência pedida em uma janela de 2 segundos.
- Watchdogs que detectam linha do tempo parada e vídeo congelado e acionam a recuperação.

## Resultado e limites

Depois da correção, com captura DXGI e encoder AMD em 1080p60, foram 358 transições em 360 quadros, cerca de 58,5 mudanças por segundo, com os quatro cantos íntegros.

O teste foi local, com áudio silencioso e sem conexão de longa distância. Ele mostra que o pipeline preserva a cadência. Não garante 60 imagens únicas por segundo em qualquer rede ou máquina.

## O que este caso ensina

Uma métrica pode estar correta e mesmo assim medir a coisa errada. O FPS de saída era verdadeiro, mas não respondia à pergunta que importava. A medição útil exigiu um sinal de teste construído para isso.
