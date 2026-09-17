---
name: full-audit-dados
description: Camada 1 do /full-audit — audita RLS/policies, autenticação e colunas sensíveis de um app Supabase/Postgres em produção. Use quando o usuário pedir pra auditar RLS, policies, ou segurança de dados de um app específico.
disable-model-invocation: true
---

# Full Audit — Dados/Segurança

Camada 1 da auditoria completa (`/full-audit`). Pressupõe que a etapa de alinhar escopo
(`/full-audit` seção 0) já rodou nesta sessão — se não rodou, invoque `/full-audit` primeiro.

## O que varrer

- **RLS/policies de autorização** — toda tabela com dado de usuário tem RLS habilitada?
  Toda policy de SELECT/INSERT/UPDATE/DELETE reflete a regra de negócio real (dono vê só o
  próprio dado, admin vê tudo, etc.) — não só "existe uma policy", mas "a policy certa".
- **Autenticação** — endpoints/RPCs que deveriam exigir login realmente exigem
  (`verify_jwt`/checagem de `auth.uid()`); nenhum checa e-mail ou UUID hardcoded em vez de
  função de papel (`current_user_is_admin()` ou equivalente).
- **Colunas sensíveis** — campos que só admin/sistema deveria mudar (status de aprovação,
  flags de pagamento, saldo) têm proteção own a nível de coluna, não só de linha.

**Fora de escopo desta camada, de propósito:** RLS decide *quem* pode escrever/ler uma
linha, nunca *o que* tem dentro do conteúdo escrito. Uma policy de INSERT/UPDATE correta
não garante que o texto salvo é seguro pra reexibir depois (XSS armazenado) — isso é
achado de UI ao vivo, não de policy. Ver `/full-audit-fuzzing` pra esse eixo.

## Como provar cada achado

Siga o loop de prova-antes-de-corrigir do `/full-audit` (seção 2): hipótese → provar o
exploit ao vivo dentro de uma transação com ROLLBACK (papel/token do usuário real, não
superusuário) → corrigir fora da transação de teste → reprovar os dois lados → documentar.

**Cross-tenant (checar se usuário A vê/afeta dado de B) usa sempre duas contas de teste
suas, descartáveis — nunca uma conta real de cliente do dono do app** (regra da seção 0 do
orquestrador). É justamente nesta camada que essa regra mais importa, porque RLS/policy é o
tipo de achado que se prova trocando o "dono" do dado — fácil de sem querer usar um ID de
cliente real em vez de uma segunda conta de teste.

## Padrões conhecidos específicos desta camada

Releia antes de começar (catálogo completo em `/full-audit` seção 4):
- **RLS por linha sem RLS por coluna**
- **RETURNING exige SELECT policy**
- 🔴 **Policy que faz EXISTS/subquery direto em outra tabela com RLS** — risco de recursão
  infinita; usar função `SECURITY DEFINER`.
- 🔴 **Bucket de Storage público com policy sem dono, corrigido pela metade** — se algum fluxo
  de upload já foi migrado pra bucket privado, `grep` pelo nome do bucket antigo em todo o
  repo antes de dar a tabela/domínio como resolvido — quase sempre sobra um segundo fluxo
  esquecido. Ver entrada completa no catálogo (seção 4 do orquestrador).
- 🔴 **View `SECURITY DEFINER` auto-atualizável com GRANT amplo demais** — view de leitura
  que virou porta de escrita real na tabela de baixo (bypassa RLS, inclusive DELETE onde
  ninguém deveria poder deletar). Checagem rápida e determinística: advisor nativo do Supabase
  (`security_definer_view`) + `information_schema.views.is_updatable`/`is_insertable_into` +
  `role_table_grants` pra `anon`/`authenticated`. Ver entrada completa no catálogo (seção 4).

**Critério de conclusão:** toda tabela/policy/RPC do escopo está em um de três estados —
revisado sem achado, achado com correção provada, ou risco aceito com justificativa escrita.

## Confirmado em: 8LOOP, 2026-08-25

126 policies revisadas literal (schemas `public`+`storage`). 1 achado crítico corrigido
(bucket público, ver entrada no catálogo geral); constraint de tabela alinhada com RLS que já
tinha mudado sem ela acompanhar (mesma lição: reescrever uma policy sem reler o schema
inteiro deixa outra camada de defesa desatualizada); bucket morto/órfão limpo. 4 achados
menores fecharam como risco aceito documentado — nem todo achado precisa virar correção,
desde que a decisão fique escrita com o motivo.
