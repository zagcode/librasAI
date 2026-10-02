# LibrasAI

Tradução automática de áudio em português para Libras (Língua Brasileira de Sinais), mostrada
por um avatar 3D, para usar em:

- OBS Studio: lives e transmissões de culto
- Holyrics e ProPresenter: projeção em igrejas e eventos
- Zoom, Microsoft Teams e Google Meet: reuniões

> Status: pesquisa e planejamento. Ainda não há código.

O LibrasAI é um recurso de acessibilidade complementar. Não substitui intérprete de Libras
humano, que continua sendo o padrão exigido pela NBR 15290 e pela Lei Brasileira de Inclusão.

## Como deve funcionar

Um motor local recebe áudio (ou o texto do slide, quando ele já existe), converte em texto,
traduz para glosa de Libras e anima um avatar numa página web local. Cada programa recebe essa
página de um jeito: Browser Source no OBS, câmera virtual nas reuniões, script ou API no Holyrics
e no ProPresenter.

## Documentação

- [Resumo da pesquisa e arquitetura proposta](docs/pesquisa/00-resumo.md)
- [Pipeline de tradução](docs/pesquisa/01-pipeline-traducao.md)
- [Integrações por plataforma](docs/pesquisa/02-integracoes.md)
- [Base legal e acessibilidade](docs/pesquisa/03-legal-e-acessibilidade.md)
- [Fontes](docs/pesquisa/04-fontes.md)
