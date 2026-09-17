---
name: full-audit-api
description: Checklist do /full-audit pra API pura sem Supabase (Node/Deno custom, REST/GraphQL próprio) — rate limit, validação de schema, paginação, autenticação vs autorização. Sem entradas provadas ainda no catálogo geral. Use quando o usuário pedir pra auditar uma API própria de um app específico.
disable-model-invocation: true
---

# Full Audit — API pura (checklist)

Complementa as camadas 1-2 (`/full-audit-dados`, `/full-audit-negocio`) pra apps com API
própria fora do padrão Supabase/RLS. **Nenhum item aqui é um achado provado** — são pontos
de atenção genéricos de mercado, pra a varredura não pular esses ângulos. Só vira entrada do
catálogo geral (`/full-audit` seção 4) depois de confirmado contra um app real, passando
pelo loop de prova-antes-de-corrigir (`/full-audit` seção 2).

## Pontos de atenção

- **Rate limit por endpoint sensível** — login, reset de senha, envio de código, endpoint
  público sem autenticação. Código curto/adivinhável não é proteção suficiente sozinho.
- **Validação de schema no limite** — o contrato aceita um campo a mais sem erro (silêncio
  que pode virar mass assignment)? Aceita um campo a menos sem erro quando deveria ser
  obrigatório?
- **Paginação vazando contagem total de outro tenant** — `total`/`count` de uma resposta
  paginada expõe quantos registros existem no sistema todo, não só os que o usuário pode ver.
- **Autenticação vs. autorização confundidas na resposta** — 401 (não autenticado) devolvido
  quando deveria ser 403 (autenticado mas sem permissão) esconde a existência do recurso; o
  inverso vaza informação de que o recurso existe pra quem não deveria nem saber.

**Critério de conclusão:** todo item acima foi checado contra o app real (não só lido e
descartado) — resultado registrado como "não se aplica", "checado, sem achado", ou "achado,
promovido pro catálogo geral com a prova".

## Confirmado em: 8LOOP, 2026-08-26

Primeira execução formal desta skill.

- **Rate limit por endpoint sensível**: ✅ checado, sem achado novo. 30 de 34 functions já têm
  (subiu de 5/~30 em 2026-08-19). Os 4 "sem": 2 webhooks protegidos por verificação de
  assinatura do provedor (não por rate-limit por usuário), 1 endpoint deprecado (retorna 410
  sem lógica), 1 (`portaria-lookup`) já tinha rate-limit por IP via `audit_logs` — mecanismo
  diferente do padrão `isRateLimited()`, por isso a busca por palavra-chave inicial não achou.
- **Validação de schema no limite (mass assignment)**: 🔴 **achado real, corrigido**.
  Comparei as ~100 colunas de `users` contra a lista protegida por
  `protect_user_sensitive_fields()` — `qa_cancellation_count` (criado no dia anterior,
  2026-08-25, especificamente pra fechar o gap de antifraude assimétrico do lado QA já
  documentado desde 2026-08-19) nunca tinha sido adicionado à lista. Qualquer usuário
  conseguia zerar o próprio contador via PATCH direto em `/rest/v1/users`, contornando
  `register_qa_cancellation_penalty()` (já SECURITY DEFINER + REVOKE de PUBLIC/anon/
  authenticated) e evitando cair na fila de revisão + notificação ao admin — anulando na
  prática a proteção construída um dia antes. `disputas_total` incluído junto por
  consistência (não decide suspensão sozinho, mas não deveria ser editável). Testado em
  transação com ROLLBACK (seed como service_role, reset como usuário comum revertido pelo
  trigger), aplicado e confirmado em produção, commitado.
- **Paginação vazando contagem de outro tenant**: ✅ checado, sem achado. Todo uso de
  `count: "exact"` no projeto é consumo interno (decisão de negócio, checagem de limite) —
  nenhum é devolvido cru numa resposta ao cliente. Contagens do lado frontend já respeitam
  RLS por natureza (cliente autenticado normal, não service_role).
- **401 vs 403 confundidos**: ✅ checado, sem achado. Padrão consistente em toda a base: sem
  sessão válida → 401, sessão válida mas sem permissão pra ação específica → 403.

**Lição de metodologia (não achado do app):** ao testar o fix com transação+ROLLBACK,
`current_setting('request.jwt.claims')` fica NULL por padrão numa sessão `supabase db query
--linked` (não simula automaticamente nenhum papel) — a primeira tentativa de prova (seed
sem `SET request.jwt.claims = service_role` explícito) deu resultado inconclusivo (o guard
já bloqueava o próprio seed, mascarando se a proteção nova funcionava). Sempre setar o papel
explicitamente pros dois lados do teste (before como service_role, tentativa de bypass como
authenticated), nunca assumir que a conexão da CLI já carrega um papel implícito.
