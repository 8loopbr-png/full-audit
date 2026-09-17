---
name: full-audit-frontend
description: Camada 3 do /full-audit — audita touchpoints de escrita no frontend (insert/update/RPC disparados pela tela), não só telas de exibição. Use quando o usuário pedir pra auditar o frontend de um app específico.
disable-model-invocation: true
---

# Full Audit — Frontend (touchpoints de escrita)

Camada 3 da auditoria completa (`/full-audit`). Pressupõe que a etapa de alinhar escopo
(`/full-audit` seção 0) já rodou nesta sessão — se não rodou, invoque `/full-audit` primeiro.

## O que varrer

Todo lugar do frontend que escreve no banco — `insert`/`update`/chamada de RPC — não só as
telas que exibem dado. Notificação que nunca dispara e coluna trocada são invisíveis até
alguém provar contra o banco real, não bastam ler o código e "parecer certo".

Pra cada chamada de escrita: o `.select()` que alimenta qualquer condição de negócio
próxima tem literalmente o campo usado na condição? (achado real: `cancel-task` usava
`row.payment_method` numa condição sem esse campo estar no `.select()` — sempre `undefined`,
condição sempre falsa, sem erro nenhum em lugar nenhum.)

## Como provar cada achado

Siga o loop de prova-antes-de-corrigir do `/full-audit` (seção 2). Pra esse tipo de bug
específico (campo fora do select), a prova é determinística — leitura do código + conferir
o `.select(` mais próximo acima da condição — não precisa de transação/ROLLBACK.

## Padrões conhecidos específicos desta camada

Releia antes de começar (catálogo completo em `/full-audit` seção 4):
- **Coluna lida sem estar no SELECT — condição sempre falsa por engano de shape**

**Critério de conclusão:** todo touchpoint de escrita do escopo está em um de três estados —
revisado sem achado, achado com correção provada, ou risco aceito com justificativa escrita.

## Confirmado em: 8LOOP, 2026-09-04 (primeira execução formal)

Escopo por risco: preço/pagamento (QAServiceSheet.tsx/Create.tsx) → ações irreversíveis
(cancelar/avaliar em TaskDetail.tsx) → dado bancário (OnboardingBank.tsx). Achado curioso,
mesmo padrão já visto em `-gateway`: pelo menos 2 achados de "sem try/catch trava botão em
'Enviando…'" já tinham sido corrigidos informalmente em 2026-09-01 (comentário
`/full-audit-frontend 2026-09-01` no próprio código, TaskDetail.tsx cancel()/RatingModal),
sem a skill ter sido invocada como comando antes — mesma lição de sempre grep'ar o código por
comentário da própria skill antes de assumir "nunca rodou" só pela ausência desta seção.

2 achados novos e reais nesta execução, ambos no mesmo trigger de banco
(`validate_fixed_price_task`), achados a partir do write touchpoint de `tasks.insert()` em
QAServiceSheet.tsx/Create.tsx — ambos provados ao vivo (transação com ROLLBACK) e corrigidos:

1. **Teto de preço fail-open pra categoria desconhecida** [dinheiro, alto] — `ELSE NULL` no
   CASE de teto por categoria significava "categoria fora da lista = sem teto nenhum", só o
   limite geral da tabela (R$5-R$1.000) sobrava. Provado: `category='Retirar Portaria Extra'`
   (deveria travar em R$30) + `value=999` foi aceito. Corrigido pra `ELSE 30` (fail-closed) —
   migration `20260904000000`.
2. **Piso de categoria "negociada" só existia no frontend** [dinheiro, alto] — `prof_certificado`
   (mín R$200) e `quebra_galho` (mín R$100) pulavam toda checagem no trigger (`RETURN NEW`
   cego); comentário dizia que o piso era "garantido pela RLS", mas a RLS real só checa o
   intervalo geral. Provado: `service_type='prof_certificado'` + `value=10` foi aceito.
   Corrigido com piso real de servidor pras 2 categorias — migration `20260904010000`.

Áreas checadas sem achado novo nesta rodada: cancelamento/avaliação de tarefa (já corrigidas
em 01/09), preferência de velocidade de repasse em OnboardingBank.tsx (RPC/edge function
mediando o campo sensível de verdade — chave PIX —, update direto só pro campo não-sensível),
suspensão de usuário em Admin.tsx (via RPC `suspend_user_with_reason`, não write direto —
padrão mais seguro por desenho). Padrão que se repete entre os 2 achados: o comentário do
código *descrevia* uma proteção que não existia de verdade no banco — reforça a regra central
do orquestrador (nunca aceitar "parece certo" sem prova ao vivo, nem quando o código tem
comentário explicando por que "já está protegido").
