# librasAI

Tradutor automático de áudio/texto em português para Libras (avatar 3D) com integrações para
OBS, Holyrics, ProPresenter, Zoom, Teams e Meet. Repositório: https://github.com/zagcode/librasAI

## Estado

Fase de pesquisa. Leia `docs/pesquisa/00-resumo.md` antes de propor arquitetura ou código: ali
estão as decisões e os riscos já levantados.

## Decisões já tomadas na pesquisa

- Motor local único (ASR → glosa → página do avatar em localhost) + adaptadores finos por
  plataforma. Não duplicar o pipeline dentro de cada integração.
- Avatar: player WebGL do VLibras (LGPLv3) via `player.play(glosa)`, com tradutor controlado pelo
  projeto. O endpoint público `traducao2.vlibras.gov.br` é só fallback (estava fora do ar em
  02/10/2026).
- Holyrics e ProPresenter mandam o texto do slide. Não usar ASR para letra/versículo.
- Reuniões: começar por câmera virtual + loopback de áudio. Apps nativos (Zoom App, Teams, Meet
  add-on) ficam para a fase 2.
- sign.mt é CC BY-NC-SA: serve só de referência, não incorporar código.

## Regras

- Comunicar como acessibilidade complementar, nunca como substituto de intérprete.
- Processamento local por padrão (LGPD). Nuvem só como opção explícita.
