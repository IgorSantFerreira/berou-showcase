# Estudo: estabilidade do signaling

**Investigado em setembro de 2026. Parcialmente resolvido.**

## Sintoma

Clientes perdiam a conexão WebSocket com o backend de sinalização sem receber a mensagem de fechamento do protocolo. Do lado do cliente, a conexão simplesmente terminava.

## O que não foi possível provar

A investigação procurou as causas mais prováveis, como expiração por falta de ping, expiração de sessão e limites da plataforma. Nenhuma delas apareceu nas evidências. A causa da desconexão original continua sem prova, e o relatório interno registra isso explicitamente em vez de declarar o problema resolvido.

## O que foi encontrado e corrigido

Durante a análise apareceu uma condição de corrida real no backend. Se um participante se desconectasse entre o momento em que o servidor localizava sua conexão e o momento do envio, a exceção escapava do tratamento. A correção tornou o envio tolerante a esse caso e a limpeza da conexão idempotente.

Um teste novo reproduz a corrida. Ele falhava antes da correção e passa depois. A validação incluiu também testes no cliente Python e um cenário no runtime local da Cloudflare.

## O que mudou no diagnóstico

O backend passou a registrar, por minuto e sem dados que identifiquem participantes, um resumo da saúde das conexões. Se o problema voltar, haverá dados para distinguir as hipóteses.

No cliente, a reentrada automática em uma sala salva já existia, com número limitado de tentativas. A mídia não é reiniciada automaticamente nesse caso.

## O que este caso ensina

Nem toda investigação termina com a causa encontrada. Corrigir um defeito real encontrado no caminho, sem afirmar que ele era a causa, e instrumentar o sistema para a próxima ocorrência é um resultado honesto e útil.
