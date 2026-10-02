# Base legal, normas e acessibilidade

Este documento não é parecer jurídico. Antes de qualquer promessa comercial de "conformidade",
o texto vigente das leis e normas tem de ser conferido.

## Legislação

| Norma | O que importa para o projeto |
| --- | --- |
| Lei 10.436/2002 | Reconhece a Libras como meio legal de comunicação e expressão. O poder público e as concessionárias de serviço público devem apoiar o uso e a difusão da língua. |
| Decreto 5.626/2005 | Regulamenta a Lei 10.436: ensino da Libras, formação de intérpretes, atendimento a pessoas surdas. |
| Lei 10.098/2000 | Primeira lei a tratar de acessibilidade de pessoas surdas em radiodifusão. |
| Lei 13.146/2015 (LBI) | Art. 67: os serviços de radiodifusão de sons e imagens devem permitir legenda oculta, janela com intérprete da Libras e audiodescrição. A LBI também trata de acessibilidade em sites e comunicação. |
| Lei 12.319/2010 | Regulamenta a profissão de tradutor e intérprete de Libras (alterada pela Lei 14.704/2023). Reforça que "intérprete" é uma profissão: o produto não deve se apresentar como intérprete. |
| ANCINE IN 116/2014 e IN 128/2016 | Regras de acessibilidade visual e auditiva (incluindo Libras) para obras audiovisuais e salas de cinema. |
| LGPD (Lei 13.709/2018) | Áudio e transcrição de reunião ou culto são dados pessoais. Exigem base legal, transparência e segurança, sobretudo se forem para a nuvem. |

## ABNT NBR 15290: janela de Libras

A NBR 15290 (*Acessibilidade em comunicação na televisão*, de 2005, revisada em 2016) define no
item 7 as diretrizes para a janela de Libras. Pela versão de 2005:

- A janela deve ter altura de pelo menos metade da altura da tela e largura de pelo menos
  um quarto da largura da tela, para que mãos e expressões faciais fiquem legíveis.
- Deve haver contraste nítido entre fundo, roupa e pele do intérprete.
- A janela não pode ser coberta por legendas ou tarjas.

Os tamanhos da revisão de 2016 precisam ser conferidos no texto da norma. Implicações para o
produto:

- A fonte do avatar no OBS deve vir com um preset "NBR 15290" (tamanho e posição mínimos) e
  avisar quando o usuário encolher a janela abaixo do mínimo.
- Em reunião, a câmera virtual deve mostrar o avatar ocupando o quadro e com o rosto visível,
  porque a expressão facial faz parte da gramática da Libras.
- Fundo sólido ou transparente com contorno, e cores de avatar e roupa com bom contraste.

## Avatar não é intérprete

Pontos levantados em pesquisas e avaliações com usuários surdos:

- A tradução automática para Libras é bem recebida como recurso, mas é criticada pela "falta de
  humanidade" e pela qualidade. Ainda não está madura para contextos críticos (saúde,
  jurídico, avaliação escolar), em que um erro de tradução causa prejuízo.
- Estudantes surdos e profissionais da educação apontam as limitações técnicas do avatar em
  comparação com o intérprete humano.
- Há trabalhos tentando incluir emoções e expressões faciais na tradução automática
  (artigos na revista CEI da UFPB e no SBSI), o que mostra que essa é uma lacuna conhecida.
- Existe avaliação de especialistas e usuários do VLibras Widget ("Hand-Speaking Technology",
  publicada no SBQS) que pode servir de roteiro para testes do projeto.

### Diretrizes para o LibrasAI

1. Comunicar o produto como recurso de acessibilidade complementar, para quando não há
   intérprete, e nunca como substituto em contexto oficial ou obrigatório.
2. Incluir pessoas surdas desde a prova de conceito, como consultoras e testadoras, e
   remunerá-las.
3. Mostrar legenda em português junto com o avatar (o player do VLibras já tem legenda), porque
   muitas pessoas surdas e com deficiência auditiva leem português.
4. Permitir que o operador pause ou desligue o avatar com um clique quando a tradução estiver
   ruim ou atrasada.
5. Registrar falhas (palavras sem sinal, fila atrasada) para melhorar o dicionário e o tradutor.

## Privacidade (LGPD)

- Preferir processamento local (Whisper no próprio computador). Assim o áudio não sai da
  máquina.
- Em reunião, avisar os participantes de que um tradutor automático está ouvindo, assim como as
  plataformas avisam sobre gravação e transcrição.
- Se usar ASR ou LLM em nuvem, documentar o provedor, a retenção e a finalidade, e não guardar o
  áudio.
