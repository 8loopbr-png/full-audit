---
name: full-audit-gateway
description: Checklist do /full-audit pra gateway de pagamento (Stripe/Mercado Pago/outro) — idempotência de webhook, corrida entre captura manual e cron, arredondamento de moeda. Sem entradas provadas ainda no catálogo geral. Use quando o usuário pedir pra auditar integração de pagamento de um app específico.
disable-model-invocation: true
---

# Full Audit — Gateway de pagamento (checklist)

Complementa a camada 2 (`/full-audit-negocio`) pra apps com gateway de pagamento
(Stripe/Mercado Pago/outro). **Nenhum item aqui é um achado provado** — são pontos de
atenção genéricos de mercado, pra a varredura não pular esses ângulos. Só vira entrada do
catálogo geral (`/full-audit` seção 4) depois de confirmado contra um app real, passando
pelo loop de prova-antes-de-corrigir (`/full-audit` seção 2).

## Como provar sem gastar dinheiro de verdade

O loop de prova-antes-de-corrigir (`/full-audit` seção 2) usa ROLLBACK, mas ROLLBACK só
desfaz o banco — uma chamada real à API do Stripe/MP (captura, estorno, criação de cobrança)
já aconteceu antes do ROLLBACK rodar e **não é desfeita**. Pra essa camada, a prova ao vivo
tem que usar o **modo sandbox/test do provedor** (chaves de teste, cartão de teste) — nunca
disparar uma chamada real contra a conta de produção do gateway só pra provar um exploit.
Se o app não distingue chave de teste de chave real na função sob teste (ou se o bug é
justamente nisso), documentar como achado à parte antes de prosseguir.

## Pontos de atenção

- **Idempotência de webhook** — o mesmo evento chega duas vezes (retry do provedor,
  timeout que faz o provedor reenviar). O efeito duplica (paga duas vezes, credita XP duas
  vezes)? Existe uma chave de idempotência conferida antes de processar?
- **Condição de corrida entre captura manual e automática** — cron de auto-aprovação e
  admin clicando "aprovar" ao mesmo tempo sobre a mesma cobrança. As duas tentam capturar?
  Existe lock/status intermediário que impede a segunda tentativa?
- **Arredondamento de moeda** — centavos perdidos ou ganhos em split de pagamento ou cálculo
  de taxa (ex: 20% de um valor com centavos quebrados — pra onde vai o resto?).
- **Reconciliação assíncrona** — o status muda no gateway antes ou depois do webhook
  chegar. O app trata os dois casos (consulta ativa + espera passiva)?
- **Replay de webhook antigo** — depois de corrigir um bug de processamento, um evento
  antigo reenviado (replay manual, ou reentrega automática do provedor) roda com a lógica
  nova sobre um payload velho. Isso produz um resultado diferente do que produziria na
  época? É um problema?

**Critério de conclusão:** todo item acima foi checado contra o app real (não só lido e
descartado) — resultado registrado como "não se aplica" (app não tem esse fluxo), "checado,
sem achado", ou "achado, promovido pro catálogo geral com a prova".

## Confirmado em: 8LOOP, 2026-08-26

Primeira execução **formal** desta skill — achado curioso: os 5 itens já tinham sido cobertos
informalmente em sessões anteriores (2026-07-24, 2026-08-17, 2026-08-25), cada uma marcada
com comentário `/full-audit-gateway <data>` no próprio código, mesmo a skill nunca tendo sido
invocada como comando antes. A lição do checklist já estava sendo aplicada na prática, só não
rastreada formalmente aqui.

- **Idempotência de webhook**: ✅ checado, sem achado novo. Stripe usa tabela `webhook_events`
  (insere ANTES de processar, reverte se processar falhar — retry do provedor funciona certo).
  MP usa guarda de transição de status (`.eq("status","aceita")` + checar linhas afetadas) pro
  branch de pagamento (achado 2026-07-24) e o mesmo padrão foi estendido pro branch de
  chargeback (achado 2026-08-25, tinha ficado sem herdar a proteção quando foi criado).
- **Corrida captura manual vs automática**: ✅ checado, sem achado novo. `approve-task` já
  guarda por `UPDATE ... WHERE status='pendente_aprovacao'` + checa linhas afetadas — quem
  perde a corrida aborta sem tentar capturar/transferir de novo (achado 2026-08-17).
- **Arredondamento de moeda**: 🔴 **reaberto e corrigido em 2026-09-02** — "sem achado novo"
  acima não se sustentou: `_shared/fees.ts` consolidou a FÓRMULA (evita divergência de
  fórmula entre funções), mas não garantia que toda função **usasse a prioridade certa**
  entre valor real do gateway e estimativa. `approve-task` já lia
  `capData.transaction_details.net_received_amount` (taxa MP real, achado 2026-08-07) antes
  de cair pra estimativa; `cron-auto-approve` — mesmo importando a fórmula consolidada —
  nunca ganhou essa leitura, então toda tarefa em crédito MP auto-aprovada (48h sem resposta)
  calculava o repasse com a taxa estimada (2,99%) mesmo quando a real cobrada era maior
  (~4,98%, mesma discrepância do achado original). Consolidar a fórmula não é o mesmo que
  consolidar a PRIORIDADE de fonte de dado — checagem futura precisa perguntar as duas coisas
  separado. Corrigido: mesma extração de `net_received_amount` replicada em
  `cron-auto-approve`, deploy confirmado. Ver catálogo geral (seção 4) e `/full-audit-negocio`.
- **Reconciliação assíncrona**: 🟡 checado, sem achado, confiança menor que os 3 acima — app
  usa Supabase Realtime + polling em várias telas, mas não achei uma tela dedicada de "espera
  confirmação de pagamento" pra provar diretamente. Revisitar se o assunto voltar.
- **Replay de webhook antigo**: ✅ conceitual, sem achado — já coberto pelos mesmos mecanismos
  de idempotência do item 1 (evento bem-sucedido não reprocessa; evento que falhou é
  responsabilidade da janela de retry do próprio provedor, fora do controle do app).

**Lição de processo:** o comentário `/full-audit-gateway <data>` deixado no código por
sessões anteriores foi o que permitiu reconstruir esse histórico rapidamente — reforça o
valor de sempre citar o nome da skill/achado na mensagem de commit e no comentário do fix,
não só documentar em memória externa.
