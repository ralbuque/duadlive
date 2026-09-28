# CLAUDE.md - Projeto Duad Live

> Este arquivo é lido por qualquer instância do Claude que trabalhe neste repositório
> (o dono usa vários computadores). **Mantenha-o atualizado** ao final de cada etapa
> relevante: decisões, resultados de testes e próximos passos.

## Dono e idioma
- Dono: Ricardo (ralbuque@gmail.com). Fuso: America/Sao_Paulo.
- Comunicação e textos de interface em **português do Brasil**.
- Domínio pretendido: **duad.live**. Hospedagem: servidor **Windows** já contratado (boa memória e banda).

## O que é o projeto
Uma página web com **dois players do YouTube ao mesmo tempo**:
1. A live do comentarista (só o rosto/voz dele).
2. A live que ele comenta (de outro canal), sincronizada com a primeira.

**Por quê:** retransmitir a live de terceiros com comentários pode gerar strike de copyright,
mesmo em uso legítimo (fair use). Com duas lives separadas, o comentarista não retransmite
o conteúdo alheio: o espectador é quem abre as duas, cada uma direto do YouTube.

**Problema central: sincronização.** O comentarista controla a própria live, mas não a do outro canal.
Ele reage ao que vê com atraso, então a live principal precisa ficar **atrás** do "agora"
(tipicamente 10-40 s) no player do espectador.

## Abordagem técnica
- Usar a **YouTube IFrame Player API** (`getCurrentTime`, `getDuration`, `seekTo`, `setVolume`,
  `setPlaybackRate`). Nada de captura de vídeo: o embed é cross-origin e não deve ser burlado.
- Requer que a live principal tenha **DVR ligado** e **embed liberado**.
- **Invariante de sincronia (adotado em `site/index.html`): distância até a borda ao vivo.**
  Para uma live, `getDuration()` devolve o tempo decorrido desde o início da transmissão (a borda) e
  `getCurrentTime()` a posição atual, então `edge = getDuration() - getCurrentTime()` é o quanto o player
  está atrás do "agora". O comentarista marca uma vez `L = edge(principal) - edge(minha)`. Cada espectador
  então mantém `edge(principal) = edge(minha) + L`, corrigindo com `seekTo` quando o erro passa de 2 s.
  Isso não depende da origem da linha do tempo de cada espectador nem de janela de DVR deslizante.
  (O protótipo `site/proto.html` usa a diferença de posições absolutas; a versão por borda é a preferida,
  mas **ambas ainda precisam ser validadas com lives reais**.)
- Precisão esperada: 0,5 a 2 s (suficiente para comentário; não é quadro a quadro).
- **Link por Duad Live (sem backend):** `https://duad.live/?h=<id minha live>&m=<id principal>&l=<L em s>&t=<título>`.
  Sem `h`/`m` (ou com `&studio=1`) a mesma página abre como **estúdio**: carregar as lives, marcar a sincronia
  e copiar o link. Como L é específico de uma transmissão, cada nova live do comentarista exige novo link
  (a fase de servidor automatiza isso).
- **Comentários:** iframe oficial `https://www.youtube.com/live_chat?v=<id da minha live>&embed_domain=<hostname>`.
  Para escrever, o espectador precisa estar logado no YouTube (o bloqueio de cookies de terceiros pode
  atrapalhar); não funciona se o chat da live estiver desativado. Apenas o chat da MINHA live é exibido.
- Fase final (planejada): página estúdio envia ao servidor (WebSocket/HTTP) a tabela de correspondência em tempo
  real, eliminando o L fixo e a necessidade de novo link a cada live.

### Limitações conhecidas
- Embed desativado pelo dono da live -> não funciona (erros 101/150).
- DVR desligado na live principal -> não dá para atrasá-la; sincronia impossível.
- Anúncios inseridos no player da principal deslocam a linha do tempo (o laço corrige depois).
- Autoplay exige clique do espectador (iniciar mudo, botão "Entrar").
- Abrir via `file://` causa erro 153: **sempre servir por http(s)**.
- Não é aconselhamento jurídico; conferir termos de uso do player embutido do YouTube.

## Layouts (implementados em `site/index.html`, escolha persistida em localStorage)
- **Celular (<= 760 px):** principal em cima, minha live embaixo, comentários abaixo (rolagem da página).
- **Desktop "1/3 + 2/3":** principal em 2/3; na coluna de 1/3, minha live (16:9) e os comentários abaixo dela.
- **Desktop "Painel no canto":** principal em tela cheia; painel no canto inferior esquerdo com **60% da altura**
  (o original de 30% ficou pequeno) e largura de 20% (mín. 360 px, ajustável 15-40% por controle),
  contendo minha live no topo e os comentários abaixo.
- **Comentários:** "abaixo da minha live" (padrão), "painel separado" (coluna/gaveta de 340 px à direita) ou ocultos.
- O YouTube exige player com no mínimo 200x200 px; o player da minha live nunca fica menor que 200 px de altura.
- Decisão de layout preferido do dono ainda em aberto; ambos disponíveis.

## Roadmap
1. **[FEITO] Banco de testes** (`site/proto.html`, em https://duad.live/proto.html): dois players, sincronia por delta
   marcado manualmente, painel de diagnóstico, testes de seek/velocidade, relatório copiável.
2. **[FEITO] Versão com link e comentários** (`site/index.html`): estúdio que gera link, visualizador com
   os dois layouts, chat da minha live, mobile empilhado, sincronia por borda ao vivo.
3. **[PENDENTE] Validar a sincronia com lives reais** (estabilidade de `getCurrentTime`/`getDuration` em lives,
   precisão do `seekTo`, se `setPlaybackRate(1.05)` é aceito, deriva em 30 min) e registrar em "Resultados dos testes".
4. Servidor + página de estúdio com tabela de sincronia em tempo real (elimina o link novo a cada live).
5. Multiusuário: login/senha; cada usuário cadastra sua "duadlive" (título, thumbnail própria,
   as duas lives) e recebe um **link próprio**. Ideia: produto útil para outros criadores.
6. Produção: HTTPS, domínio duad.live, hospedagem no servidor Windows (Node ou ASP.NET/SignalR).

## Estrutura do repositório
```
CLAUDE.md          este arquivo
site/index.html    estúdio (sem parâmetros) e visualizador (com ?h=&m=&l=&t=); é o produto
site/proto.html    banco de testes de sincronização com diagnóstico e relatório
deploy/Caddyfile.duad  bloco do Caddy para duad.live (importado pelo Caddyfile do veracibot)
```

## Como rodar localmente
Servir a pasta `site/` por HTTP, por exemplo `python -m http.server 8000` dentro dela, e abrir
`http://localhost:8000/` (nunca por `file://`). Em produção: https://duad.live/ (estúdio),
https://duad.live/proto.html (banco de testes).

## Implantação no servidor Windows
Situação do servidor (levantada em 2026-09-28):
- Já existe o **Caddy 2.11.4** rodando como serviço nssm `veracibot-caddy` (conta LocalSystem), ouvindo em 80/443.
  Ele atende o site `veraci.bot` (outro projeto), que é um app Python em `127.0.0.1:8000`.
- O Caddyfile em uso fica **no repo do veracibot**: `C:\Git\veracibot\deploy\windows\Caddyfile`
  (serviço: `run --config` esse arquivo; API admin em `127.0.0.1:2019`).
- **Não instalar um segundo Caddy** e não ocupar 80/443. O duad.live entra como mais um domínio no Caddy existente.

Como o duad.live é ligado:
1. O bloco do domínio fica **neste repo**, em `deploy/Caddyfile.duad` (versionado aqui).
2. No Caddyfile do veracibot acrescenta-se **apenas uma linha** (commitar lá para evitar conflito em pulls):
   `import C:/Git/duadlive/deploy/Caddyfile.duad`
3. Clonar este repo no servidor em `C:\Git\duadlive` (se for outro caminho, ajustar `root` no `Caddyfile.duad`, que aponta para a subpasta `site`).
4. DNS: registros A de `duad.live` e `www.duad.live` para o IP do servidor (o Caddy emite o HTTPS sozinho quando o DNS resolver).
5. Validar e recarregar sem derrubar o veraci.bot (rodar na pasta do Caddyfile do veracibot):
   `caddy validate --config C:\Git\veracibot\deploy\windows\Caddyfile --adapter caddyfile`
   `caddy reload   --config C:\Git\veracibot\deploy\windows\Caddyfile --adapter caddyfile`
   Fazer antes uma cópia de segurança do Caddyfile do veracibot.
6. Atualizar o site: `git pull` no servidor (o Caddy serve a pasta direto; sem build e sem reiniciar).
   Se `deploy/Caddyfile.duad` mudou, rodar também o `caddy validate` e o `caddy reload` do passo 5.

Quando existir backend (sincronia em tempo real, login): rodar como novo serviço nssm em outra porta local
(ex.: `127.0.0.1:8001`, `8000` é do veracibot) e acrescentar `reverse_proxy /api/* 127.0.0.1:8001` no bloco do duad.live.
O Caddy repassa WebSocket sem configuração extra.

## Resultados dos testes
_(ainda não realizados; preencher com o relatório copiado da página)_

## Convenções
- Arquivo único HTML/JS/CSS sem build no protótipo; manter simples até validar a sincronia.
- Atualizar este CLAUDE.md quando algo relevante mudar (decisão, resultado, estrutura).
- Não commitar segredos (chaves de API, senhas, .env).
