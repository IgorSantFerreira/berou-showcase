# Decisões técnicas

Cada decisão abaixo traz o problema que a motivou, o que foi escolhido e o custo da escolha.

## 1. Todo o processamento de mídia em um processo Rust

**Implementado desde a 4.0.**

Nas versões 3.x, o pipeline de mídia rodava em Python. Três problemas foram documentados nessa época: o GIL e as pausas de coleta de lixo quebravam a cadência entre áudio e vídeo, cada viewer exigia um processo FFmpeg próprio no host e o transporte ICE em Python ficava no caminho crítico da mídia.

A 4.0 moveu captura, codificação, distribuição e transporte para um motor em Rust, executado como processo separado. O Python passou a cuidar só da interface e da orquestração.

**Trade-off:** o protocolo entre Python e Rust virou uma interface que precisa ser versionada e testada dos dois lados. Em troca, a mídia ganhou um runtime sem GIL e sem coleta de lixo, e uma falha no motor não derruba a interface.

## 2. Uma codificação no host, N viewers

**Implementado.**

Em vez de codificar um stream por viewer, o host codifica uma vez e publica em um servidor MediaMTX local, de onde cada viewer lê.

**Trade-off:** o custo de CPU e GPU no host deixa de crescer com o número de viewers, mas todos recebem a mesma qualidade. Não há adaptação de bitrate por viewer.

## 3. HEVC com encoder de hardware e fallback ordenado

**Implementado.**

O motor tenta encoders de hardware de AMD, NVIDIA e Intel, nessa ordem, e usa o libx265 por software como último recurso. Um encoder só é aceito se sustentar pelo menos 95% da cadência pedida. Um encoder recusado fica fora das tentativas por um período curto e depois volta a ser testado. Os encoders de hardware recebem o formato de pixel NV12 diretamente, o que evita uma conversão de cor.

**Trade-off:** HEVC reduz o bitrate para a mesma qualidade, mas exige suporte a decodificação no viewer. A regra de admissão nasceu de um problema real, descrito em [cadência de vídeo](estudos/cadencia-de-video.md).

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

**Trade-off:** não há servidor sempre ligado para manter, e como o backend não toca em mídia, ele não paga pela banda dos streams. O preço é ficar preso ao modelo de execução e às limitações da plataforma escolhida.

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

**Trade-off:** cada release exige atualizar esse arquivo de forma deliberada. Em troca, qualquer build é reproduzível e um relatório de diagnóstico identifica a combinação exata de componentes.
