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

## Abordagem técnica (plano)
- Usar a **YouTube IFrame Player API** (`getCurrentTime`, `getDuration`, `seekTo`, `setVolume`,
  `setPlaybackRate`). Nada de captura de vídeo: o embed é cross-origin e não deve ser burlado.
- Sincronizar por **posição na linha do tempo** (não por relógio de parede), o que independe da
  latência de cada espectador. Requer que a live principal tenha **DVR ligado** e **embed liberado**.
- Fase final: página "estúdio" do comentarista envia ao servidor (WebSocket/HTTP) uma tabela
  "posição na minha live -> posição na live principal". O espectador lê a posição da live do
  comentarista, consulta a tabela e faz `seekTo` na principal. Laço de correção a cada 1-2 s
  (seek se erro > ~2 s; ajuste fino por velocidade só se o YouTube aceitar).
- Precisão esperada: 0,5 a 2 s (suficiente para comentário; não é quadro a quadro).

### Limitações conhecidas
- Embed desativado pelo dono da live -> não funciona (erros 101/150).
- DVR desligado na live principal -> não dá para atrasá-la; sincronia impossível.
- Anúncios inseridos no player da principal deslocam a linha do tempo (o laço corrige depois).
- Autoplay exige clique do espectador (iniciar mudo, botão "Entrar").
- Abrir via `file://` causa erro 153: **sempre servir por http(s)**.
- Não é aconselhamento jurídico; conferir termos de uso do player embutido do YouTube.

## Layouts desejados
- **Celular:** uma live sobre a outra (principal em cima, comentarista embaixo).
- **Desktop:** ainda em decisão entre (a) 2/3 principal + 1/3 comentarista, ou
  (b) principal em tela cheia + painel do comentarista no canto inferior esquerdo (20% largura x 30% altura).
  O protótipo já tem os dois com um seletor.

## Roadmap
1. **[FEITO] Protótipo de teste** (`prototype/duadlive-proto.html`): dois players, sincronia por delta
   marcado manualmente, painel de diagnóstico, testes de seek/velocidade, relatório copiável.
2. **[PENDENTE] Rodar o protótipo com lives reais** e colher: estabilidade de `getCurrentTime` em lives,
   precisão/tempo do `seekTo`, se `setPlaybackRate(1.05)` é aceito, deriva em 30 min.
   Registrar os resultados na seção "Resultados dos testes" abaixo.
3. Servidor + página de estúdio com tabela de sincronia em tempo real.
4. Multiusuário: login/senha; cada usuário cadastra sua "duadlive" (título, thumbnail própria,
   as duas lives) e recebe um **link próprio**. Ideia: produto útil para outros criadores.
5. Produção: HTTPS, domínio duad.live, hospedagem no servidor Windows (Node ou ASP.NET/SignalR).

## Estrutura do repositório
```
CLAUDE.md                     este arquivo
prototype/duadlive-proto.html protótipo de teste (arquivo único, sem dependências além da API do YouTube)
deploy/Caddyfile.duad         bloco do Caddy para duad.live (importado pelo Caddyfile do veracibot)
```

## Como rodar o protótipo
Servir por HTTP na pasta `prototype/`, por exemplo `python -m http.server 8000`, e abrir
`http://localhost:8000/duadlive-proto.html`. Procedimento de teste está descrito na própria página.

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
3. Clonar este repo no servidor em `C:\Git\duadlive` (se for outro caminho, ajustar `root` no `Caddyfile.duad`).
4. DNS: registros A de `duad.live` e `www.duad.live` para o IP do servidor (o Caddy emite o HTTPS sozinho quando o DNS resolver).
5. Validar e recarregar sem derrubar o veraci.bot (rodar na pasta do Caddyfile do veracibot):
   `caddy validate --config C:\Git\veracibot\deploy\windows\Caddyfile --adapter caddyfile`
   `caddy reload   --config C:\Git\veracibot\deploy\windows\Caddyfile --adapter caddyfile`
   Fazer antes uma cópia de segurança do Caddyfile do veracibot.
6. Atualizar o site: `git pull` no servidor (o Caddy serve a pasta direto; sem build e sem reiniciar).

Quando existir backend (sincronia em tempo real, login): rodar como novo serviço nssm em outra porta local
(ex.: `127.0.0.1:8001`, `8000` é do veracibot) e acrescentar `reverse_proxy /api/* 127.0.0.1:8001` no bloco do duad.live.
O Caddy repassa WebSocket sem configuração extra.

## Resultados dos testes
_(ainda não realizados; preencher com o relatório copiado da página)_

## Convenções
- Arquivo único HTML/JS/CSS sem build no protótipo; manter simples até validar a sincronia.
- Atualizar este CLAUDE.md quando algo relevante mudar (decisão, resultado, estrutura).
- Não commitar segredos (chaves de API, senhas, .env).
