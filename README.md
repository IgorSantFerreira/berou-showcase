# Berou: showcase técnico

O Berou é um aplicativo de compartilhamento de tela e streaming de baixa latência para Windows, atualmente em beta. Sou responsável pela definição do produto, pela arquitetura e por grande parte da implementação. Outro colaborador participou da separação dos repositórios, da configuração de CI e containers e da documentação interna.

Reuni aqui a arquitetura, algumas decisões de engenharia e investigações de problemas encontrados durante o desenvolvimento. O código-fonte e a infraestrutura do Berou são privados e não fazem parte deste repositório.

## Estágio atual

- A última versão estável publicada é a 4.4.0. A linha 4.5 está em desenvolvimento e é distribuída em canal beta.
- Desde a 4.0.0, todo o processamento de mídia roda em um motor escrito em Rust. As versões 3.x usavam um pipeline em Python e são citadas aqui apenas como contexto histórico.
- Em setembro de 2026, o antigo monorepo foi dividido em componentes com repositórios e CI próprios, integrados por um arquivo que fixa as revisões de cada um.

Os textos distinguem recursos implementados, funcionalidades em desenvolvimento, medições observadas e limitações conhecidas.

## Documentação

| Documento | Conteúdo |
|---|---|
| [Arquitetura](docs/arquitetura.md) | Componentes, responsabilidades e como eles se comunicam |
| [Decisões técnicas](docs/decisoes-tecnicas.md) | Escolhas de engenharia e seus trade-offs |
| [Testes, diagnóstico e telemetria](docs/testes-e-diagnostico.md) | Estratégia de testes, CI, observabilidade e método de investigação |
| [Estudo: cadência de vídeo](docs/estudos/cadencia-de-video.md) | Por que "60 FPS" no encoder não significava 60 imagens por segundo |
| [Estudo: queda de mídia por socket de áudio](docs/estudos/socket-de-audio.md) | Um erro de socket no Windows que derrubava a transmissão |
| [Estudo: estabilidade do signaling](docs/estudos/estabilidade-do-signaling.md) | Uma investigação com causa original não provada e o que foi corrigido mesmo assim |
| [Estado e limitações](docs/estado-e-limitacoes.md) | O que está pronto, o que está em andamento e o que ainda limita o projeto |

## Tecnologias

Python e Qt/QML (PySide6) no cliente, Rust no motor de mídia, FFmpeg para captura e codificação, SRT e MediaMTX na distribuição local do stream, ICE/TURN no transporte entre máquinas, Cloudflare Workers com Durable Objects no backend de sinalização, OpenTelemetry e Grafana na observabilidade, PyInstaller e Inno Setup no empacotamento.
