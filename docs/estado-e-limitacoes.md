# Estado e limitações

Situação em outubro de 2026. A última versão estável publicada é a 4.4.0, e a linha 4.5 é distribuída em canal beta.

## Linha do tempo

| Versão | Mudança principal |
|---|---|
| 3.x | Pipeline de mídia em Python, interface migrada para Qt Quick, primeiras atualizações assinadas |
| 4.0 | Motor de mídia em Rust |
| 4.1 | Conexão pela internet somente via relay TURN |
| 4.2 | Correções de cadência e de estabilidade do streaming; divisão do código em repositórios |
| 4.3 | Chat temporário, voz com Opus, canal Beta |
| 4.4 | Observabilidade com OpenTelemetry e Grafana |
| 4.5 (beta) | Continuidade de sala, controles de voz, capacidade dinâmica, fallback TCP para o relay |

## Implementado

- Compartilhamento de tela e do áudio de um aplicativo para múltiplos viewers, com uma única codificação no host.
- Encoders de hardware AMD, NVIDIA e Intel, com fallback para software.
- Conexão pela internet via relay TURN, com túnel cifrado.
- Salas por convite, chat temporário e voz.
- Telemetria de sessão e correlação de eventos entre participantes.
- Instalador, atualização assinada e canais Stable e Beta.

## Em desenvolvimento (linha 4.5)

- Recuperação da mídia após falhas de sinalização ou expiração das credenciais do relay, sem encerrar a sala.
- Preservação da sala quando o participante que a coordena muda.
- Fallback do relay por TCP para redes que bloqueiam UDP.
- Limites de uso de CPU na captura e na reprodução.

## Limitações conhecidas

- **Somente Windows.** Captura, áudio e gerenciamento de processos usam APIs do Windows.
- **Mesma qualidade para todos os viewers.** Não há adaptação de bitrate por viewer.
- **Latência e custo do relay.** Todo tráfego pela internet passa por TURN, e o uso é limitado por um orçamento no backend.
- **Causa de desconexão em aberto.** A desconexão sem fechamento do protocolo descrita no [estudo de signaling](estudos/estabilidade-do-signaling.md) não teve a causa original provada.
- **Sem pipeline de integração.** A combinação de componentes de um release é fixada por arquivo, mas não há teste automatizado da integração entre eles.
- **Cobertura de CI parcial.** O CI do control plane roda só parte da suíte, e alguns testes dependem de hardware ou de Windows.
- **Sem análise agregada de telemetria.** Os dados são coletados, mas ainda não há análise consolidada de uso real.
- **Interface legada.** Uma interface antiga em Tkinter ainda existe como opção.

## Planos

- Suporte a câmera, hoje apenas planejado.
