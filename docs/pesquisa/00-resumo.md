# Resumo da pesquisa

Levantamento feito em 02/10/2026 para orientar o projeto LibrasAI: um "ativador" que pega o
áudio (ou o texto) de uma transmissão, culto, apresentação ou reunião e mostra um avatar
sinalizando em Libras.

Documentos desta pasta:

| Arquivo | Conteúdo |
| --- | --- |
| [01-pipeline-traducao.md](01-pipeline-traducao.md) | Reconhecimento de fala, português → glosa, glosa → avatar, latência |
| [02-integracoes.md](02-integracoes.md) | OBS, Holyrics, ProPresenter, Zoom, Teams, Google Meet |
| [03-legal-e-acessibilidade.md](03-legal-e-acessibilidade.md) | Leis, normas ABNT, janela de Libras, posição da comunidade surda, LGPD |
| [04-fontes.md](04-fontes.md) | Lista de fontes com links |

## Conclusões principais

1. O VLibras pode ser o motor de tradução. O projeto, do governo federal com o LAVID/UFPB, é
   código aberto (LGPLv3) e tem cerca de 21 mil sinais animados em 3D, um tradutor
   português → glosa (regras + rede neural + refinamento por LLM) e um player WebGL com método
   `translate(texto)`. Com isso, o esforço do projeto vai para as integrações e para o tempo real.

2. O serviço público do VLibras não serve como dependência de produção. No teste de
   02/10/2026, o endpoint `https://traducao2.vlibras.gov.br/translate` respondeu
   `500 Translation Core Unavailable` em todas as tentativas. Não há SLA. O plano precisa prever
   hospedar o tradutor (`vlibras-translator-text-core`) por conta própria, ou ter uma alternativa.

3. A arquitetura certa é um motor local mais adaptadores finos. Um único serviço local
   (o "motor LibrasAI") faz fala → texto → glosa e serve uma página web com o avatar em
   `http://localhost`. Cada programa só precisa de um adaptador pequeno:
   - OBS: Browser Source apontando para a página do avatar (fundo transparente). Um plugin
     C++ nativo é opcional, para captar o áudio de uma fonte específica.
   - Holyrics: script JavaScript com `h.addTriggerListener` que envia o texto do slide ou
     versículo ao motor. Não precisa de reconhecimento de fala.
   - ProPresenter: o motor lê `GET /v1/status/slide` (API oficial, com modo `chunked`) e
     recebe o texto do slide atual. Também dispensa reconhecimento de fala.
   - Zoom, Teams, Meet: o caminho que funciona nos três é uma câmera virtual com o avatar
     (OBS Virtual Camera ou uma câmera própria) entrando como "participante intérprete". Assim ele
     aproveita o *sign language view* do Teams e os canais de interpretação do Zoom. Apps nativos
     ficam para uma segunda fase (detalhes em 02).

4. Em igreja, o texto já existe. Letras de música e versículos já estão no Holyrics e no
   ProPresenter. Traduzir esse texto pronto é mais rápido e mais preciso do que reconhecer a fala.
   O reconhecimento de fala só é necessário na pregação, na fala livre e nas reuniões.

5. Velocidade é o problema técnico mais difícil. A sinalização por avatar é mais lenta que a
   fala corrida. Sem alguma estratégia (resumir, acelerar, descartar trechos), a fila de sinais
   cresce sem parar e o avatar fica minutos atrasado. Isso precisa ser resolvido no MVP (ver 01).

6. O avatar não substitui intérprete humano, e o produto precisa dizer isso. A NBR 15290 e
   a LBI falam em *janela com intérprete*. Pesquisas com usuários surdos apontam que a tradução
   automática não está madura para contextos críticos. O posicionamento honesto é "acessibilidade
   complementar quando não há intérprete", com pessoas surdas validando o resultado.

## Arquitetura proposta (rascunho)

```
            ┌──────────── entradas ────────────┐
 áudio  ──► │ captura (mic / loopback / fonte  │
            │ OBS / RTMS do Zoom)              │──► ASR pt-BR (faster-whisper / whisper.cpp
            └──────────────────────────────────┘     com VAD, ou nuvem: Azure/Google)
                                                         │ texto
 texto pronto (Holyrics script, ProPresenter API) ───────┤
                                                         ▼
                                       segmentação + controle de ritmo
                                                         │
                                                         ▼
                                    português → glosa (VLibras text-core
                                    hospedado, ou LLM com dicionário de sinais)
                                                         │ glosa
                                                         ▼
                          página do avatar (VLibras player WebGL) em localhost
                                                         │
          ┌──────────────────┬───────────────────────────┼─────────────────────┐
          ▼                  ▼                           ▼                     ▼
   OBS Browser Source   OBS → NDI / câmera virtual   Câmera virtual      Painel/side panel
   (live, culto)        → ProPresenter Video Input   → Zoom/Teams/Meet   (apps nativos, fase 2)
```

## Riscos e pontos em aberto

- Licença do VLibras (LGPLv3): permite uso em produto, inclusive comercial, desde que as
  partes LGPL possam ser substituídas e o código delas modificado seja publicado. O sign.mt usa
  CC BY-NC-SA 4.0, que proíbe uso comercial, então só serve como referência.
- Hand Talk: API comercial, só para web/WebGL, com limite de 1.000 caracteres por pedido. Pode
  ser um segundo motor opcional, mas cria custo e dependência.
- Áudio de reunião: Teams (bot de mídia via Graph, em C#/Windows) e Meet (Media API ainda em
  Developer Preview, com todos os participantes inscritos) não oferecem caminho simples para o
  áudio dentro de um app. O Zoom tem o RTMS, já disponível para todos os desenvolvedores.
- LGPD: o áudio de reunião é dado pessoal. Processar localmente reduz o risco. Se for para a
  nuvem, é preciso aviso e base legal.
- Não verificado nesta pesquisa: o formato exato da "API Server" REST do Holyrics (a
  documentação pública achada cobre só os scripts JS) e se o ProPresenter abre página web como
  mídia (o caminho seguro é NDI ou câmera de captura como Video Input).

## Próximos passos sugeridos

1. Prova de conceito do motor: whisper (local) → glosa → player VLibras numa página localhost,
   medindo a latência de ponta a ponta.
2. Subir o `vlibras-translator-text-core` em Docker e confirmar a qualidade das glosas.
3. Adaptador Holyrics (script JS) e adaptador ProPresenter (cliente da API), porque não dependem
   de reconhecimento de fala e mostram resultado rápido.
4. OBS: Browser Source com fundo transparente e posição/tamanho da janela de Libras.
5. Reuniões: câmera virtual com o avatar e captura do áudio do sistema (loopback).
6. Testar com pessoas surdas antes de qualquer divulgação.
