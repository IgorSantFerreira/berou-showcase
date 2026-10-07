# Decisões técnicas

Cada decisão abaixo traz o problema que a motivou, o que foi escolhido e o custo da escolha.

## 1. Todo o processamento de mídia em um processo Rust

**Implementado desde a 4.0.**

Nas versões 3.x, parte do pipeline de mídia e do transporte passava pelo runtime Python. Havia problemas de cadência e um processo FFmpeg por viewer, o que duplicava o trabalho de codificação no host. O GIL e as pausas de coleta de lixo eram fatores de risco para a previsibilidade do processamento, mas não foram isolados como a causa única das falhas observadas.

A 4.0 moveu captura, codificação, distribuição e transporte para um motor em Rust, executado como processo separado. O Python passou a cuidar só da interface e da orquestração.

**Trade-off:** o protocolo entre Python e Rust passou a exigir versionamento e testes dos dois lados. Em contrapartida, o processamento de mídia saiu do runtime Python e passou a ser isolado em outro processo. Isso permite tratar falhas do motor sem necessariamente encerrar a interface.

## 2. Uma codificação para múltiplos espectadores

**Implementado.**

O host mantém uma única captura e um único encoder de vídeo. O fluxo resultante é publicado no MediaMTX local e distribuído aos viewers sem nova codificação para cada conexão.

**Impacto:** adicionar espectadores não cria novos processos de captura ou de encoding, nem multiplica a carga principal de CPU e GPU associada à produção do vídeo. A distribuição ainda utiliza rede, buffers e processamento de transporte por conexão, mas não repete o trabalho pesado de codificação. Como todos recebem o mesmo fluxo, ainda não existe adaptação de bitrate individual.

## 3. HEVC com encoder de hardware e fallback ordenado

**Implementado.**

O motor tenta encoders de hardware de AMD, NVIDIA e Intel, nessa ordem, e usa o libx265 por software como último recurso. Um encoder só é aceito se sustentar pelo menos 95% da cadência pedida. Um encoder recusado fica fora das tentativas por um período curto e depois volta a ser testado. Os encoders de hardware recebem o formato NV12, evitando uma reorganização adicional do formato planar no caminho de codificação.

**Trade-off:** HEVC pode entregar qualidade comparável com menos bitrate, dependendo da configuração e do conteúdo, mas exige suporte à decodificação no viewer. A regra de admissão nasceu de um problema real, descrito em [cadência de vídeo](estudos/cadencia-de-video.md).

## 4. SRT dentro do túnel do motor

**Implementado.**

O SRT oferece controle de perda e latência pensado para vídeo ao vivo, então o Berou o usa entre o servidor de mídia do host e o player do viewer. Mas nenhum socket SRT é exposto à rede. Os pacotes atravessam a internet cifrados e autenticados dentro de um túnel UDP controlado pelo motor Rust, sobre um caminho estabelecido por ICE.

**Trade-off:** há duas camadas de transporte para manter. Em troca, o SRT continua cuidando do que faz bem, e a superfície exposta à rede é uma só, sob controle do motor.

## 5. Conexão pela internet sempre via relay TURN

**Implementado desde a 4.1.3.**

O Berou não tenta conexão direta entre as máquinas. Todo tráfego pela internet passa por um relay TURN. Versões anteriores dependiam de conexão direta ou de VPN de terceiros, que falhavam conforme a rede de cada participante.

**Trade-off:** o relay acrescenta latência e tem custo por tráfego, controlado por um orçamento no backend. Em troca, a conexão se comporta de forma previsível em redes diferentes. Como só há candidatos de relay, o desenho também evita a troca direta de endereços entre os participantes, embora isso não tenha sido testado especificamente.

## 6. Backend serverless

**Implementado.**

A sinalização roda em Cloudflare Workers com Durable Objects, que guardam o estado de cada sala.

**Trade-off:** o backend de sinalização não precisa manter um servidor de aplicação sempre ligado nem processar os streams de vídeo. Isso não elimina o tráfego e o custo do relay TURN, que distribui os pacotes entre os participantes. A implementação também fica sujeita ao modelo de execução e aos limites da plataforma.

## 7. Observabilidade sem SDK no cliente

**Implementado na 4.4.**

O cliente usa uma API de instrumentação pequena e própria, e não um SDK de observabilidade. Os eventos vão para um Worker dedicado, que valida cada campo contra uma lista permitida e só então os converte para OpenTelemetry (OTLP sobre HTTP com JSON). As credenciais da stack de observabilidade ficam apenas no servidor.

O envio é best-effort: se falhar, o cliente tenta de novo com backoff exponencial com jitter, e a telemetria pode ser descartada sem afetar a sessão.

**Trade-off:** JSON é menos eficiente que protobuf, escolha aceitável para o volume baixo de um app desktop em beta. A validação por lista permitida exige atualizar o servidor para cada métrica nova, mas impede que dados não previstos cheguem à stack.

## 8. Atualizações assinadas com canais separados

**Atualizações assinadas implementadas desde a 3.7. Canais separados desde a 4.3.**

Cada versão tem um manifesto assinado, e o cliente verifica a assinatura antes de instalar. Há dois canais isolados: Stable, o padrão, e Beta, que o usuário precisa escolher explicitamente.

**Trade-off:** o processo de release ganha etapas de assinatura e promoção, mas uma versão beta não chega a quem não a pediu.

## 9. Revisões de componentes fixadas

**Implementado.**

Com o código dividido em repositórios, um arquivo no repositório de integração fixa a revisão de cada componente usada em um release, e o build embute essas revisões no executável.

**Trade-off:** cada release exige atualizar deliberadamente o arquivo de revisões. Em troca, é possível rastrear quais versões dos componentes entraram em cada build e identificar essa combinação em um diagnóstico. A reprodução integral de um build também depende das ferramentas, das dependências e do ambiente de empacotamento.
