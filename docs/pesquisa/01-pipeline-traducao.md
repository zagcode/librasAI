# Pipeline de tradução: áudio → Libras

São quatro etapas, e cada uma tem alternativas. O que vem abaixo é o estado em 02/10/2026.

```
áudio ─► [1] ASR (fala → texto) ─► [2] segmentação e ritmo ─► [3] texto → glosa ─► [4] glosa → avatar
```

Libras não é português sinalizado. Tem gramática própria (ordem dos termos, classificadores,
expressões faciais com função gramatical). Por isso os sistemas convertem primeiro o português
em glosa, uma escrita intermediária com os sinais em maiúsculas na ordem da Libras (por
exemplo `BOM DIA TODOS BEM-VINDO CULTO HOJE`), e só depois animam.

## 1. Reconhecimento de fala (ASR) em pt-BR

### Opções locais (sem nuvem, mais privacidade, custo zero por minuto)

| Opção | Notas |
| --- | --- |
| whisper.cpp (ggerganov) | C/C++, roda em CPU com AVX, CUDA, Vulkan/OpenCL e Metal. É o que o plugin LocalVocal usa dentro do OBS. Bom para embutir num plugin nativo. |
| faster-whisper (CTranslate2) | Python, quantização int8, mais rápido em GPU NVIDIA. Bom para um motor em Python. |
| whisper_streaming e variantes | Adaptam o Whisper (que trabalha em blocos) ao streaming, confirmando trechos estáveis. |
| Vosk | Leve e de streaming nativo, mas os modelos de português são pequenos e menos precisos que o Whisper. |
| Silero VAD | Detector de voz (janelas de 32 ms a 16 kHz). Evita mandar silêncio ao Whisper, que tende a "alucinar" texto quando só há silêncio. |

Precisão: blogs comerciais citam WER perto de 8% para o Whisper large-v3 em pt-BR com áudio
limpo, pior com ruído e sotaques do Norte e Nordeste. São números de fontes secundárias. O projeto
deve medir no próprio material (pregação gravada, reunião, live), porque igreja e evento têm
eco, música de fundo e termos bíblicos.

### Opções em nuvem

| Opção | Notas |
| --- | --- |
| Azure AI Speech | Streaming, suporta pt-BR, permite *phrase lists* para termos específicos. |
| Google Cloud Speech-to-Text | Streaming, pt-BR. |
| Zoom RTMS | Em reuniões Zoom, entrega o áudio (PCM 16 kHz) e a transcrição por participante via WebSocket. Pode dispensar o ASR próprio (ver 02). |

### Recomendação

Começar com faster-whisper ou whisper.cpp + Silero VAD rodando localmente, com o modelo
`small` ou `medium` para CPU e `large-v3`/`turbo` quando houver GPU. Deixar a etapa atrás de
uma interface, para poder trocar por nuvem.

## 2. Segmentação e controle de ritmo (o problema principal)

O ASR produz texto continuamente e o avatar consome devagar, porque cada sinal leva frações de
segundo e há transições entre eles. Numa pregação rápida, a fila cresce e o atraso vira minutos.
Estratégias a testar:

- Cortar em frases pelo VAD (pausas) e pela pontuação do ASR. Enviar frase completa ao
  tradutor, porque a glosa depende do contexto da frase.
- Velocidade do avatar: o player VLibras tem `setSpeed(speed)`.
- Descartar o que ficou velho: se a fila passa de N segundos, pular para a frase mais recente
  (melhor perder trechos que ficar fora de sincronia).
- Resumir com LLM antes da glosa quando a fila estiver cheia ("simplificação de texto").
  Isso muda o conteúdo, então tem de ser opcional e transparente.
- Texto pronto primeiro: letra de música e versículo vêm do Holyrics e do ProPresenter já
  segmentados por slide, sem ASR.

Orçamento de latência a validar na prova de conceito: ASR em 1 a 3 s após o fim da frase, glosa em
0,1 a 1 s, início da animação em menos de 1 s. A meta é o avatar começar a sinalizar 2 a 4 s depois
do fim de cada frase.

## 3. Português → glosa

### VLibras (principal candidato)

Organização no GitHub: `spbgovbr-vlibras` (código também no Portal do Software Público).
Licença LGPLv3. Repositórios relevantes:

| Repositório | Papel |
| --- | --- |
| `vlibras-translator-text-core` | Tradutor português → glosa em Python 3.12. Híbrido: regras + `vlibras-translate`/`vlibras-deeplearning` + refinamento por LLM (GPT-4o por padrão, outros provedores configuráveis). Consome jobs de uma fila RabbitMQ (`translate.to_text`). Roda com `docker-compose up`. |
| `vlibras-translator-api` | API Node.js (v3.2.0) que orquestra os *cores* de texto e vídeo. Usa PostgreSQL, Redis e RabbitMQ. Documentação em `/docs` quando roda local. |
| `vlibras-player-webjs` | Player JS com o avatar em Unity WebGL (ver seção 4). |
| `vlibras-web-browsers` | Widget de sites + extensões Chrome/Firefox (v7.12.2; Vite, Preact, TypeScript). Mostra a configuração atual dos serviços. |
| `vlibras-mobile-cross-platform` | App móvel. |

O endpoint público usado pelo widget é `POST https://traducao2.vlibras.gov.br/translate` com
corpo `{"text": "...", "domain": "<host do site>"}`. A resposta é a glosa em texto. No teste de
02/10/2026 ele devolveu `422` quando `domain` estava vazio e
`500 {"error":"Translation Core Unavailable"}` com `domain` preenchido. Ou seja, estava fora do
ar. Não depender dele em produção.

### Hand Talk (comercial)

API "Hand Talk for Devs": SDK só em JavaScript/TypeScript, só para web com WebGL, limite de 1.000
caracteres por texto (acima disso, erro 400). Traduz para Libras e ASL com os avatares Hugo e
Maya. Preço e autenticação não estão na página de introdução da documentação, então é preciso
consultar a empresa.

### Rybená (comercial)

Tem documentação pública de API de tradução e de controle do player (`docs.rybena.com.br`). Libras
só a partir do português.

### LLM com dicionário de sinais

Pesquisas de 2025 (SBC/Latinoware, WiCS, arXiv 2508.18183 "AulSign") mostram LLMs gerando glosa
com *prompting* dinâmico e exemplos, associando cada sinal a uma descrição. O ponto crítico é
restringir a saída ao vocabulário que o avatar sabe sinalizar. Palavra sem sinal tem de virar
datilologia (soletração manual) ou sinônimo existente. O próprio text-core do VLibras já usa LLM
como etapa de refinamento.

### Dados e corpora

- V-LIBRASIL (UFPE): 4.089 vídeos de 1.364 sinais, cada um feito por pelo menos três
  intérpretes. `libras.cin.ufpe.br`.
- MINDS-Libras (Zenodo): 20 sinais, 12 sinalizantes, 5 repetições. Foco em reconhecimento.
- VLibrasBD: corpus bilíngue português/Libras citado em trabalhos de tradução neural.
- O VLibras declara cerca de 100 mil frases paralelas português/Libras no seu acervo.

## 4. Glosa → avatar

### Player do VLibras (`vlibras-player-webjs`)

Uso mínimo (do `demo/index.html` do repositório):

```html
<div id="wrapper"></div>
<script src="vlibras.js"></script>
<script>
  var player = new VLibras.Player({
    target: { name: 'rnp_webgl', path: 'target' }
  });
  player.on('load', function () { /* pronto */ });
  player.load(document.getElementById('wrapper'));
</script>
```

API de `Player` (lida em `src/Player.js`):

- `translate(texto)` manda o texto ao tradutor e toca a glosa. Se o tradutor falhar, toca o texto
  em maiúsculas, o que na prática vira datilologia.
- `play(glosa)` toca uma glosa pronta. É o ponto de integração para usar tradutor próprio.
- `pause()`, `continue()`, `stop()`, `repeat()`, `setSpeed(v)`, `changeAvatar(nome)`,
  `setRegion(regiao)`, `toggleSubtitle()`, `setPersonalization(...)`.
- Eventos: `load`, `translate:start`, `translate:end`, `gloss:start`, `gloss:end`,
  `animation:play`, `animation:pause`, `animation:end`, `animation:progress`, `error`.
- URLs em `src/config.js`: tradutor `traducao2.vlibras.gov.br/translate` e dicionário de sinais
  `dicionario2.vlibras.gov.br/2018.3.1/WEBGL/` (os *bundles* de animação de cada sinal são
  baixados sob demanda). O widget novo também usa `.../static/BUNDLES/...` e
  `.../static/TREES/2018.3.1.json`.

Pontos de atenção:

- As animações dos sinais vêm do servidor de dicionário do governo. Para funcionar offline ou com
  previsibilidade, é preciso espelhar os bundles localmente (avaliar volume e licença).
- O widget atual usa avatares regionais (Ícaro, Hosana, Guga).
- O player é Unity WebGL, então roda no Browser Source do OBS (CEF/Chromium) e em qualquer
  navegador com WebGL.

### Alternativas

- Hand Talk SDK (WebGL, comercial).
- sign.mt: texto → SignWriting → sequência de poses → esqueleto, vídeo "realista" (GAN) ou
  avatar 3D. É aberto, mas a licença CC BY-NC-SA 4.0 impede uso comercial. Útil como referência
  técnica. O suporte à Libras precisa ser conferido.
- Vídeos de sinais concatenados (arXiv 2409.01506): emendar clipes de sinais isolados. Fica
  mais humano, mas as transições ficam visíveis e o vocabulário é limitado.

### Recomendação

MVP com o player do VLibras e `play(glosa)`, alimentado por um tradutor que o projeto controla:
text-core hospedado ou LLM restrito ao vocabulário do dicionário VLibras. O tradutor público fica
só como *fallback*.
