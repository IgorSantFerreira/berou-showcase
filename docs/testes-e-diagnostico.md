# Testes, diagnóstico e telemetria

## Testes automatizados

**Implementado.** Cada componente tem sua própria suíte:

| Componente | Ferramenta | Tamanho (contagem estática) |
|---|---|---|
| Control plane (Python) | pytest | 233 funções de teste em 40 arquivos |
| Motor de mídia (Rust) | testes nativos do Cargo | 49 testes |
| Backend de sinalização | `node --test` | 13 arquivos de teste |
| Build e instalador | unittest | 2 arquivos de teste |

As contagens foram feitas por busca no código em outubro de 2026. Uma execução parcial, em Linux e sem as dependências de Windows, deu o seguinte resultado observado:

- backend de sinalização: 49 de 50 testes passaram e 1 foi pulado, por exigir os clientes Python reais. Os 14 testes de observabilidade passaram;
- control plane: 190 testes passaram. As falhas e os erros de coleta se concentraram em código que depende de Windows ou da interface gráfica, que não estavam disponíveis no ambiente;
- motor de mídia: não foi executado, porque o alvo é Windows. No CI ele roda em Windows.

Essa execução não comprova que a suíte completa passa no Windows.

Os testes do motor cobrem pontos em que bugs reais apareceram. Exemplos: a ordem de fallback e a memória de rejeição dos encoders, as regras de admissão por cadência e o comportamento do socket de áudio quando o buffer de envio enche (ver [queda de mídia por socket de áudio](estudos/socket-de-audio.md)). No backend, uma correção de concorrência foi acompanhada de um teste que falhava antes da correção e passava depois (ver [estabilidade do signaling](estudos/estabilidade-do-signaling.md)).

**Limitação:** parte dos testes do control plane depende de APIs exclusivas do Windows e não é coletada em Linux. Um teste do motor que depende de hardware real é pulado no CI, e o CI do control plane roda apenas 9 dos 40 arquivos de teste.

## CI

**Implementado.** Os quatro componentes têm CI no GitHub Actions. O motor de mídia, o control plane e o instalador rodam em Windows, e o backend de sinalização em Linux, com uma simulação do deploy. Como cada um evolui em repositório próprio, a combinação usada em cada release é controlada pelo arquivo de revisões fixadas do repositório de integração. Não há pipeline de integração automatizado.

## Telemetria e observabilidade

**Implementado na 4.4.** O cliente envia eventos e métricas de sessão para um Worker de telemetria, que os encaminha para uma stack Grafana com Loki (logs) e Tempo (traces).

**Limitação:** ainda não há análise agregada dos dados de uso. As medições citadas neste repositório vêm de testes pontuais, não de dados de produção.

Tipos de dado coletados:

- eventos do ciclo de vida da sessão e do transporte;
- progresso da codificação reportado pelo FFmpeg, como quadros, tempo, FPS e velocidade;
- contadores do transporte.

Separadamente, o backend de sinalização registra em log, por minuto, um resumo da saúde das conexões, sem dados que identifiquem participantes.

Os eventos de dois participantes da mesma sessão são correlacionados por um identificador de trace derivado da requisição que os originou. Com isso, é possível reconstruir em uma única linha do tempo o que o host e o viewer viram.

A telemetria não inclui métricas em Prometheus nem rastreamento distribuído completo, e é best-effort: pode perder eventos.

## Método de investigação

As investigações do projeto seguem um padrão: sintoma, hipóteses, evidência, causa, correção e verificação. Cada relatório separa o que foi provado, o que não foi reproduzido e o que não foi testado. Dois hábitos aparecem com frequência:

- **Medir a grandeza certa.** No [estudo de cadência](estudos/cadencia-de-video.md), o FPS reportado pelo encoder estava correto e mesmo assim enganava. A métrica útil era o número de mudanças reais de conteúdo por segundo, medida com um padrão de teste sintético.
- **Desconfiar da correlação.** No [estudo do socket de áudio](estudos/socket-de-audio.md), os relógios das duas máquinas não estavam sincronizados, então os eventos foram alinhados por identificador de sessão e ordem, não por horário. Um fallback de encoder que aparecia logo depois da falha foi descartado como causa.

## Exemplo de medição: formato de pixel e uso de GPU

**Resultado observado, sem significância estatística.** Para avaliar o efeito de entregar NV12 em vez de BGRA ao encoder de hardware, uma fonte sintética 1440p60 foi codificada em HEVC 1080p a 10 Mbps em uma GPU AMD RX 6750 XT, com cinco amostras do contador de uso de GPU do Windows por variante:

| Métrica (uso médio da GPU) | BGRA | NV12 |
|---|---|---|
| Motor Encode/3D | 5,23% | 3,98% |
| Motor Video Codec | 16,07% | 15,42% |

A redução no motor 3D é coerente com a eliminação da conversão de cor na GPU. No player não houve ganho claro. Cinco amostras de uma execução curta não sustentam uma conclusão estatística, e o próprio registro interno diz isso.
