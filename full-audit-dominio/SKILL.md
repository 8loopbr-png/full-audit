---
name: full-audit-dominio
description: Checklist do /full-audit pra completude de domínio real — se o produto representa corretamente a vida/convívio do contexto físico-social que ele serve (ex: condomínio: portaria, visitantes, veículos, moradores indo/vindo, harmonia, regras de uso). Diferente das outras camadas (bug de código/segurança), aqui o achado é lacuna de representação da realidade. Sem entradas provadas ainda no catálogo geral. Use quando o usuário pedir pra checar se o app "cobre tudo" do mundo real que ele diz servir.
disable-model-invocation: true
---

# Full Audit — Completude de domínio real (checklist)

Complementa as demais camadas do `/full-audit`. **Diferente de todas as outras** (que caçam
bug de código/segurança), aqui o achado não é "isso quebra" — é "isso existe no mundo real
que o app serve, mas o produto não representa, ou representa errado". Um app pode estar 100%
seguro (RLS certo, sem XSS, sem bug de preço) e ainda ter essa lacuna: a UI/fluxo não reflete
uma situação real que o usuário vive todo dia.

**Nenhum item aqui é um achado provado** — são pontos de atenção pra a varredura não pular
esses ângulos. Só vira entrada do catálogo geral (`/full-audit` seção 4) depois de confirmado
contra um app real. A "prova" aqui não é transação+ROLLBACK (não é bug técnico) — é confirmar
com o dono do produto (ou com dado real de uso) se a situação realmente acontece e se o app
hoje lida com ela, não lida, ou lida errado.

Exemplo motivador (8LOOP, condomínio fechado): o app pode estar auditado e seguro em todas as
camadas técnicas e ainda não ter pensado em "o que acontece quando o morador que pediu a
tarefa viaja e some por uma semana no meio da negociação" — isso não é bug, é lacuna de
domínio.

## Como usar

Pra cada item abaixo, perguntar: **essa situação acontece de verdade no contexto real do
app? Se sim, o produto trata ela, ignora ela, ou (pior) dá a entender que trata sem tratar de
verdade?** A terceira resposta é a mais perigosa — cria falsa expectativa de segurança/
cobertura que o produto não cumpre.

## Pontos de atenção (exemplo: condomínio — adaptar pro contexto real de cada app)

- **Convívio entre moradores** — quando duas partes discordam (tarefa mal feita, atraso,
  comportamento), o app dá um caminho claro de resolução (denúncia, disputa, avaliação) que
  não vira briga direta entre vizinhos que se veem todo dia no elevador? Sistema de nota/
  reputação é visível de um jeito que não estigmatiza publicamente quem errou uma vez.
- **Segurança/portaria/limpeza como categoria de serviço** — quando o app oferece uma
  categoria que toca a operação física do prédio (ex: "retirar encomenda na portaria",
  "levar lixo"), o texto/fluxo reflete corretamente como aquilo funciona de verdade ali
  (precisa identificação? tem horário?) ou assume um processo genérico que não bate com a
  portaria real daquele condomínio?
- **Visitantes e controle de acesso — cuidado com falsa promessa** — se o app não controla
  entrada física de visitante (decisão de escopo comum em apps C2C de vizinhança), checar
  que nenhum texto/ícone/fluxo **insinua** que controla (ex: linguagem tipo "autorizado",
  "verificado na portaria" usada fora do contexto certo) — isso cria sensação de segurança
  que o produto não entrega.
- **Veículo/deslocamento** — se o serviço envolve alguém se deslocando dentro ou até o
  condomínio (ida e volta, carregar algo), a fórmula de tempo/distância/preço reflete a
  geografia real (torres, portões, andares) ou usa uma aproximação genérica que erra sistema-
  aticamente pra layout de prédio grande?
- **Morador (ou prestador) indo e voltando — ausência temporária** — o que o app faz quando
  quem pediu ou quem aceitou uma tarefa fica inacessível por dias (viagem, hospital)? Existe
  timeout/cancelamento automático, ou a tarefa fica "presa" indefinidamente sem ninguém saber
  o que aconteceu?
- **Harmonia geral do prédio** — o uso do app, no agregado, tende a aproximar vizinhos ou
  criar atrito novo (ex: fila de quem "sempre pega as melhores tarefas", percepção de
  favoritismo)? Não é bug técnico, é efeito social do produto — vale registrar como risco de
  produto mesmo sem solução técnica óbvia.
- **Clareza do que o usuário pode fazer** — a UI deixa explícito, em algum lugar visível, o
  conjunto de ações que aquele papel (morador comum, prestador, síndico) tem dentro do app?
  Funcionalidade que existe mas está escondida/não descoberta é, na prática, funcionalidade
  que não existe pro usuário comum.
- **Regras internas do condomínio (convenção) não contrariadas** — o produto nunca dá a
  entender que substitui, sobrepõe ou libera o usuário de uma regra que o condomínio já tem
  (ex: horário de silêncio, uso de área comum) — mesmo que legalmente o condomínio não possa
  proibir o app em si (ver base jurídica já registrada em memória do produto), o app não deve
  parecer estar "por cima" da convenção existente.

## Critério de conclusão

Todo item acima foi checado contra o app real e confirmado (ou não) com o dono do produto —
resultado registrado como "não se aplica" (situação não existe nesse contexto), "checado, sem
lacuna", ou "lacuna encontrada, promovida pro catálogo geral com a descrição de como foi
confirmada" (aqui, "confirmada" pode ser relato do dono/usuário real, não só prova técnica).

## Confirmado em: 8LOOP, 2026-08-26

Primeira execução formal. 5 itens checáveis pelo código, 3 genuinamente dependentes de
confirmação do dono (natureza desta camada, não falta de esforço).

**Checados pelo código, sem lacuna:**
- **Ausência temporária** — coberto nos dois lados: OA (5 dias sem iniciar +
  `cron-expire-noshow-professional`; 6h travado no meio + `cron-expire-abandoned-inprogress`)
  e QA (48h auto-aprovação, já documentado na FAQ).
- **Visitantes/controle de acesso, falsa promessa** — nenhuma linguagem tipo "autorizado"/
  "verificado" usada fora de contexto de dado digital (RLS, permissão de admin); a única
  menção a "portaria" como controle é `/portaria`, que é lookup público pro funcionário da
  portaria verificar um OA, não promessa de controle de entrada de visitante.
- **Convívio/estigma público** — avaliação exibida é só média agregada (estrelas), nunca nota
  individual atribuída a um vizinho específico.
- **Clareza do que o usuário pode fazer / feature escondida** — comparei todas as 24 rotas
  reais do `App.tsx` contra referências de navegação (`nav()`/link) no resto do app. Só
  `/portaria` tem zero links internos — confirmado intencional (página pública pra staff da
  portaria acessar por fora, não pra morador/agente navegar de dentro do app), não é lacuna.
- **Veículo/deslocamento** — checado em 2026-09-03 (ficou de fora, sem explicação, da
  passagem de 2026-08-26 — 1 dos 8 itens do checklist passou batido em silêncio na 1ª
  execução; lição de processo: reconferir a lista completa ao fechar, não só os itens que
  já viraram achado). `Create.tsx` confirma preço 100% fixo, escolhido pelo morador
  (`customValue`), zero fator de distância entre torres/andares — a lacuna da auditoria de
  2026-08-21 (`project_8loop_full_audit_2026_08_21.md`) segue tecnicamente real. **Decisão do
  dono (2026-09-03): aceitar como está** — a contraproposta do OA (recusar/propor valor
  próprio se achar baixo pro deslocamento real) já é a válvula de escape, não precisa de
  cálculo automático de distância. Risco aceito, com justificativa registrada.

**Dependentes de confirmação do dono, registrados como pendência real, não resolvidos:**
- **Segurança/portaria/limpeza reflete o processo real** — reconfirmado pendente em
  2026-09-03 (ainda "não sei, preciso checar") — texto das categorias nunca foi validado
  contra o processo real de nenhum condomínio específico com síndico/morador. Gatilho pra
  revisitar: antes de expandir pra condomínio novo, vale essa validação.
- **Harmonia geral / atrito social (favoritismo, fila injusta)** — dono respondeu "ainda não
  tenho uso real suficiente" — base de usuários pequena demais pro padrão aparecer ainda.
  Gatilho pra revisitar: quando o volume de tarefas/usuários crescer o suficiente pra um
  padrão social real emergir.
- **Regras internas da convenção não contrariadas** — não perguntado nesta rodada (tempo),
  fica pendente pra próxima passagem por esta camada.
