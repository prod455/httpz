# Correções sobre o http.zig

Base: karlseguin/http.zig, ramo `master` (Zig 0.17), commit `1bb71ff128a044b1645df91046473c38abbf965c`.
As duas dependências dele, `metrics.zig` e `websocket.zig`, vivem em `deps/` nos commits que o upstream fixa
(`ac167295b6a8fd84ca7af852c1032e6b246101f0` e `f2b3bad6875595e8dd42c726e8e914885a8b0ba0`), por caminho: o build
não baixa nada.

Correções locais (não estão no upstream; diff completo em `PATCHES.diff`):

- A última resposta antes de fechar a conexão (limite `timeout.request_count`, ou `res.keepalive = false`) manda
  `Connection: Close`. Sem o aviso, o Caddy reaproveitava a conexão e o POST seguinte voltava 502. Teste:
  "httpz: last response before closing announces Connection: Close".
- No modo não bloqueante, cada pedido de uma conexão reaproveitada decide de novo se ela continua. Antes, o
  keep-alive do primeiro pedido valia para sempre e escondia o `Connection: close` do cliente e o
  `request_count`. Teste: "httpz: Connection: close from the client is honored on a reused connection".
- `Config.listener`: socket já aberto (o que o systemd guarda entre reinícios, `src/listen_socket.zig`). Teste:
  "httpz: serves on a listener it did not bind and leaves other copies listening".
- `Server.gracefulStop(grace_s)`: para de aceitar sem fechar a porta, responde o que já chegou com
  `Connection: Close` e espera o outro lado soltar as conexões ociosas. Teste: "httpz: gracefulStop finishes
  requests...". Só no modo não bloqueante (o do Linux); no bloqueante faz o mesmo que `stop()`.

- Linha de endereço ou bloco de cabeçalhos que enche o buffer sem terminar devolve `HeaderTooBig` (431). Antes, o
  parse respondia "preciso de mais dados" para sempre e a thread girava a 100% de CPU sem responder: um endereço de
  40 KB, sem login, prendia um núcleo até reiniciar o serviço. Teste: "request: url or header block that fills the
  buffer without ending is HeaderTooBig, not a busy loop". Achada no Autocorret (ADR 0006 de lá, correção 2).
- `swapList` solta o mutex da conexão antes de pô-la na lista de entrega: o laço de eventos recicla a conexão assim
  que ela aparece ali, e soltar um mutex reciclado derrubava o processo ("switch on corrupt value"). Autocorret, ADR
  0006, correção 4 (achada pelo rastro da pilha na bateria pesada; sem teste isolado).
- O `errdefer` do `accept` desfaz só o que o bloco fez: chamar `disown` ali destruía a `Conn` uma segunda vez (os
  `errdefer` de cima já fecham o socket e a destroem). Autocorret, ADR 0006, correção 3.

Para subir o upstream: trocar `src/` pelo do commit novo, reaplicar `PATCHES.diff` (`git apply`), rodar os
testes acima com `zig build test` e regenerar o diff.
