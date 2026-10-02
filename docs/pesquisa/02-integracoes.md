# Integrações por plataforma

Premissa: um motor local (o "motor LibrasAI") expõe uma API em `localhost` para receber áudio
ou texto e serve a página do avatar. As integrações abaixo são adaptadores para esse motor.

Resumo:

| Plataforma | Entrada (de onde vem o conteúdo) | Saída (onde o avatar aparece) | Esforço do MVP |
| --- | --- | --- | --- |
| OBS Studio | Áudio do microfone ou do sistema; no plugin nativo, áudio de uma fonte do OBS | Browser Source com fundo transparente | Baixo (Browser Source) / médio (plugin C++) |
| Holyrics | Texto do slide via script JS (`addTriggerListener`) | Página do avatar no OBS, ou em tela/monitor | Baixo |
| ProPresenter 7 | Texto do slide via API (`/v1/status/slide`) | NDI ou câmera virtual como Video Input; ou OBS | Baixo/médio |
| Zoom | Áudio do sistema (loopback) ou RTMS | Câmera virtual como participante; fase 2: Zoom App (Camera Mode) | Baixo (câmera virtual) / alto (app) |
| Teams | Áudio do sistema (loopback) | Câmera virtual + *sign language view*; fase 2: app de reunião | Baixo / muito alto (bot de mídia) |
| Google Meet | Áudio do sistema (loopback) | Câmera virtual; fase 2: Meet add-on | Baixo / alto |

## OBS Studio

### Caminho 1: Browser Source (MVP, sem compilar nada)

- O Browser Source do OBS é um Chromium embutido (CEF), com WebGL. Roda o player Unity WebGL do
  VLibras.
- A página do avatar fica em `http://localhost:<porta>/avatar`, com `background: transparent`
  para o avatar ficar por cima da cena.
- A página pode usar `window.obsstudio` (API JS do obs-browser), que entre outras coisas tem
  `startVirtualcam()` e `stopVirtualcam()` e recebe eventos de cena.
- O áudio é captado pelo motor (microfone ou loopback do Windows/WASAPI), fora do OBS.

### Caminho 2: plugin nativo (fase 2)

- Base oficial: `obsproject/obs-plugintemplate` (C++17, CMake 3.16 ou mais), com documentação
  em `docs.obsproject.com/plugins`.
- Um filtro de áudio (`obs_source_info` com o callback `filter_audio`) aplicado à fonte do
  pregador recebe o áudio já mixado no OBS e manda para o motor. O usuário só escolhe a fonte de
  áudio e um botão liga ou desliga a tradução.
- O plugin também pode criar uma fonte "Janela de Libras" já posicionada e dimensionada.
- Referência direta: LocalVocal (`royshil/obs-localvocal`, antes `occ-ai/obs-localvocal`).
  Faz transcrição e tradução locais dentro do OBS com whisper.cpp, escreve em fontes de texto,
  gera .txt/.srt e manda legendas para RTMP. Exige OBS 30.2 ou mais. É a melhor base para estudar
  captura de áudio por filtro, empacotamento do Whisper e distribuição para Windows, macOS e Linux.

### Controle externo

- obs-websocket 5.x já vem no OBS 28 ou mais, na porta padrão 4455. O motor pode usá-lo para
  ligar/desligar a fonte do avatar, trocar cena ou ler estado. O Holyrics também fala com o OBS
  por esse protocolo (receptores `obs_v4`/`obs_v5`).

## Holyrics

Software brasileiro de projeção para igrejas (letras, Bíblia, avisos). Tem um ambiente de
scripts JavaScript documentado no GitHub (`holyrics/jslib`, `holyrics/JSCommunity`).

### Entrada: texto do slide (sem ASR)

- Os scripts usam ECMAScript 5.1, com *arrow functions* aceitas a partir da v2.23.0. A biblioteca
  é acessada por `jslib` ou `h`.
- `h.addTriggerListener` (v2.21.0 ou mais) registra gatilhos. Os mais úteis aqui:
  - `when: 'displaying'` com `item: 'any_song_slide'`. O objeto (`SongSlideInfo`) traz `text`,
    `title`, `slide_index`, `slide_total` etc.
  - `item: 'any_text_slide'` (`TextSlideInfo`: `text`, `title`, `slide_index`).
  - `when: 'event'` com `item: 'verse_presentation_changed'`. Traz só a referência
    (`book`, `chapter`, `verse`, `reference` como `"Gn 1:1"`), sem o texto. O texto do versículo
    precisa vir de `h.hly('GetCurrentPresentation')` ou de uma Bíblia própria do motor.
  - `when: 'closing'` serve para mandar o avatar parar.
- `h.apiRequest(urlOuReceptor, { type: 'POST', headers, data })` faz a chamada HTTP. Para chamar
  um IP ou host direto, ele precisa estar em *Arquivo > Configurações > Avançado > JavaScript >
  Configurações > Requisições permitidas*. Também há `h.ws(...)` para WebSocket.

Esboço do adaptador:

```javascript
h.addTriggerListener({
  id: 'librasai_song',
  when: 'displaying',
  item: 'any_song_slide',
  action: function (obj) {
    h.apiRequest('http://127.0.0.1:8765/v1/text', {
      type: 'POST',
      headers: { 'Content-Type': 'application/json' },
      data: { source: 'holyrics', kind: 'song', text: obj.text, title: obj.title }
    });
  }
});
```

### Saída

- O Holyrics já gera URLs na rede local para o OBS mostrar letras e versículos. O avatar entra
  como outro Browser Source na mesma cena.
- O plano Advanced tem uma "API online" para receber requisições pela internet. Existe também uma
  "API Server" local com tokens e permissões por ação (citada no módulo Companion
  `bitfocus/companion-module-limagiran-holyrics`). O formato REST dela não foi confirmado nesta
  pesquisa. Para o MVP, o script JS basta.

## ProPresenter 7

API oficial a partir da 7.9. Habilita-se em *Settings > Network* (porta padrão 50000).
Especificação: `openapi.propresenter.com` (Swagger em `openapi.propresenter.com/swagger.json`,
194 rotas na versão consultada). Há também a documentação comunitária
`jeffmikels/ProPresenter-API`.

### Entrada: texto do slide (sem ASR)

- `GET /v1/status/slide` retorna o texto do slide atual e do próximo.
- `GET /v1/presentation/active` e `GET /v1/presentation/slide_index` dão contexto.
- Vários endpoints aceitam `?chunked=true`: a conexão fica aberta e chega um novo pedaço a cada
  mudança, o que elimina o polling. Para assinar vários de uma vez, há
  `POST /v1/status/updates`, que agrega streams (navegadores só aguentam 5 ou 6 conexões abertas).
- Ter o próximo slide permite pré-traduzir e baixar as animações antes, o que reduz a
  latência.

### Saída

- `GET /v1/video_inputs` e `/v1/video_inputs/{id}/trigger` mostram que o ProPresenter trabalha
  com entradas de vídeo. O caminho seguro é levar o avatar como NDI (via OBS + plugin
  DistroAV/obs-ndi) ou câmera virtual e usá-lo como Video Input, numa camada ou em uma *Look*
  (`/v1/look/...`).
- Alternativa comum em igreja: o ProPresenter manda o texto e o OBS monta a transmissão com o
  avatar. A integração com o ProPresenter fica só na entrada.
- Não confirmado: se o ProPresenter abre página web como mídia de forma estável.

## Reuniões: estratégia comum

Em Zoom, Teams e Meet, o caminho que funciona sem aprovação de loja é:

1. O motor capta o áudio de saída do computador (loopback WASAPI no Windows), ou seja, o que
   os outros participantes falam.
2. O avatar é renderizado e publicado como câmera virtual (OBS Virtual Camera, ou câmera
   própria do LibrasAI em fase futura).
3. Uma segunda conta ou dispositivo entra na reunião como "LibrasAI (intérprete)" usando essa
   câmera.
4. O participante surdo fixa esse vídeo:
   - Teams: *sign language view* permite priorizar até dois sinalizantes, com proporção e
     qualidade preservadas, e lembrar intérpretes preferidos.
   - Zoom: canais de interpretação de língua de sinais criados pelo host, ou *pin/spotlight*.
   - Meet: fixar o participante.

Limitação: só traduz o que o computador onde o motor roda está ouvindo. Funciona bem para um
usuário surdo que leva o próprio tradutor, ou para o organizador que liga o intérprete virtual.

### Zoom (fase 2)

- Zoom Apps SDK: o app roda dentro do cliente Zoom. Em Camera Mode
  (`runRenderingContext`), a Layers API desenha o participante (`drawParticipant`) e uma webview
  transparente por cima (`drawWebView`). Dá para pôr o avatar sobre o próprio vídeo, como uma
  janela de Libras "embutida". Há relatos de `drawWebView` em branco no Windows no fórum de
  desenvolvedores.
- RTMS (Realtime Media Streams): disponível para todos os desenvolvedores. Entrega áudio PCM
  L16 16 kHz mono e transcrição por participante via WebSocket, iniciado por auto-start, REST
  ou `startRTMS()` no Zoom Apps SDK. Pode substituir o ASR próprio em reuniões Zoom.
- Legenda de terceiros: `GET /v2/meetings/{id}/token` devolve a URL de *closed caption* e o
  texto é enviado por POST `text/plain` com `seq`. Serve para mandar o texto (ou a glosa) como
  legenda, em paralelo ao avatar.
- Publicar na Zoom Marketplace exige revisão.

### Microsoft Teams (fase 2)

- App de reunião (TeamsJS v2): painel lateral e `shareAppContentToStage` para pôr a página do
  avatar no palco para todos. O domínio precisa constar em `validDomains` do manifesto (senão
  dá erro 501). Só o apresentador interage com o que está no palco.
- Áudio dentro do app: o app de painel não recebe o áudio da reunião. Para isso é preciso
  um bot de mídia em tempo real (Graph Communications API, SDK em C#/.NET, hospedado em
  Windows Server/Azure, com consentimento específico por reunião, o RSC). É bem mais complexo e
  caro. Deixar para depois de validar o produto.

### Google Meet (fase 2)

- Meet add-ons SDK (`@googleworkspace/meet-addons`): painel lateral (`MeetSidePanelClient`) e
  palco principal (`createAddonSession`). A página do avatar pode rodar no palco.
- Meet Media API (áudio e vídeo em tempo real via WebRTC): ainda em Developer Preview. O
  projeto Cloud, o usuário OAuth e todos os participantes precisam estar inscritos no programa,
  e só chegam os 3 participantes mais relevantes. Não serve para produção hoje. Sem isso, o add-on
  depende do áudio local.
