---
name: full-audit-negocio
description: Camada 2 do /full-audit — audita lógica de negócio em backend/edge functions que envolvem dinheiro ou dado sensível. Use quando o usuário pedir pra auditar funções de pagamento, escrow, aprovação, ou lógica de backend de um app específico.
disable-model-invocation: true
---

# Full Audit — Lógica de Negócio (backend/funções)

Camada 2 da auditoria completa (`/full-audit`). Pressupõe que a etapa de alinhar escopo
(`/full-audit` seção 0) já rodou nesta sessão — se não rodou, invoque `/full-audit` primeiro.

## O que varrer

Toda função/endpoint que envolve dinheiro ou dado sensível — captura de pagamento, escrow,
repasse, aprovação (manual e automática), cancelamento, cálculo de preço/taxa/comissão.

Atenção especial a **lógica duplicada em dois lugares** (ex: aprovação manual vs. automática
por cron, cálculo de preço em duas telas) — elas divergem sem ninguém perceber. Sempre que
achar um bug numa função, perguntar "essa mesma lógica existe em algum outro arquivo?" antes
de marcar como resolvido — grep pelo nome da tabela/coluna afetada em todo o repo.

Toda função de pagamento/escrow precisa de: `try/catch` cobrindo a chamada à API do
gateway, log identificável do resultado em tabela auditável (não só `console.log`), e teste
manual documentado (qual cenário, qual resultado — "rodei e não deu erro" não conta).

**Chamada interna também autentica, não confia por estar "dentro".** Quando uma function
chama outra function do mesmo backend (ex: cron dispara aprovação, uma edge function invoca
outra via HTTP interno), checar se o lado chamado confirma quem está chamando (segredo
compartilhado, JWT de serviço) ou se só confia por não estar exposto a `anon` na rota
pública — a mesma function que parece "protegida" por não aparecer na UI pode estar
acessível de verdade pra qualquer um que descubra a URL, se nada dentro dela checa a origem
da chamada.

## Como provar cada achado

Siga o loop de prova-antes-de-corrigir do `/full-audit` (seção 2): hipótese → provar o
exploit ao vivo dentro de uma transação com ROLLBACK → corrigir fora da transação de teste →
reprovar os dois lados → documentar.

## Padrões conhecidos específicos desta camada

Releia antes de começar (catálogo completo em `/full-audit` seção 4):
- **Guarda que nunca dispara na prática**
- **Lógica duplicada em gêmeos divergindo**
- **Correção em um caminho de dinheiro sem replicar no caminho gêmeo** (inclui a variante de
  app multi-gateway: recurso construído só pro gateway antigo, nunca replicado pro novo
  gateway principal — não é "esqueceram de atualizar", é "só existiu pra um desde o início")
- **Deploy de função reseta verify_jwt**

Se o app tiver 2+ papéis interagindo em uma transação (comprador/vendedor, prestador/
cliente), checar explicitamente se o antifraude cobre os DOIS lados — cancelamento repetido,
múltiplas contas, chargeback/estorno abusivo tendem a nascer cobrindo só o lado que executa
o serviço. Detalhe completo (checklist + achados reais) em `/full-audit-security` Parte A
item 2 — mas o achado em si costuma aparecer primeiro aqui, revisando função de negócio.

Se o app tiver gateway de pagamento (Stripe/MP/outro), complementar com o checklist de
`/full-audit-gateway` — idempotência de webhook, corrida entre captura manual e cron,
arredondamento de moeda.

**Critério de conclusão:** toda função/endpoint do escopo está em um de três estados —
revisado sem achado, achado com correção provada, ou risco aceito com justificativa escrita.

## Confirmado em: 8LOOP, 2026-08-25

4 cenários de antifraude do lado que paga/inicia (QA), corrigidos: webhook de chargeback
ausente pro gateway principal (ver entrada de gêmeo-divergente acima), comprovante de
conclusão aceito sem confirmar que o arquivo existia de verdade em storage, self-dealing via
CPF (cruzar CPF real entre as duas pontas de uma transação, não só dado autodeclarado), promo
de 1ª tarefa repetida via conta nova. Achado à parte, sobre o próprio processo de auditoria:
o teste E2E planejado depois revelou que a correção de CPF cobria menos do que o relatório
original descrevia (constraint de unicidade no banco torna 2 contas com mesmo CPF impossíveis
de existir, então o cenário de teste "ingênuo" nunca dispara — ver `/full-audit` seção 3).
