# Estudo: queda de mídia por socket de áudio

**Problema histórico, corrigido na 4.2.4.**

## Sintoma

O viewer perdia a imagem do host de repente, embora a conexão de rede continuasse ativa. O problema apareceu em três sessões registradas no log do host, e o encoder estava estável, perto de 58 FPS, até o momento da falha.

## Investigação

Os logs indicavam um erro de escrita no socket local que leva o áudio capturado até o FFmpeg. No Windows, esse erro (WSAEWOULDBLOCK, código 10035) significa que a operação bloquearia, e não que a conexão falhou.

A causa estava no modo do socket. O servidor local era criado em modo não bloqueante, e no Windows o socket aceito herda esse modo. Quando o buffer de envio enchia por um instante, a escrita falhava em vez de esperar. O erro subia até o status do host, que reconstruía a transmissão inteira.

Dois detalhes explicam por que o problema podia aparecer em qualquer sessão:

- definir um timeout de escrita não torna bloqueante um socket que não é;
- mesmo sem som, o motor envia áudio de silêncio para manter a linha do tempo, então esse caminho estava sempre ativo.

O defeito existia desde a introdução do motor Rust.

## Cuidado com a correlação

Os relógios das duas máquinas não estavam sincronizados, então os eventos foram alinhados pelo identificador de sessão e pela ordem em que ocorreram, não pelo horário. Uma troca de encoder registrada logo depois da falha foi descartada como causa: ela era consequência da reconstrução.

## Correção

O socket aceito passou a ser configurado explicitamente como bloqueante, com timeout de escrita de 500 ms. Foram adicionados testes que reduzem o buffer de envio para forçar a condição de buffer cheio.

## O que este caso ensina

O erro reportado não era a falha. Era uma condição normal de rede tratada como fatal, e entender o modelo de sockets da plataforma foi o que separou causa de efeito.
