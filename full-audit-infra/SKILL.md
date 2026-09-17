---
name: full-audit-infra
description: Checklist do /full-audit pra infra/CI-CD — segredo em log de build, preview público sem gate, migration fora do pipeline, rollback sem reverter schema, env var crítica ausente em deploy de worktree, backup sem teste de restauração ou apagável por admin comprometido, fornecedor crítico sem plano de contingência. Use quando o usuário pedir pra auditar deploy/infra de um app específico.
disable-model-invocation: true
---

# Full Audit — Infra/CI-CD (checklist)

Complementa as demais camadas do `/full-audit` pro pipeline de deploy em si. A maioria dos
itens abaixo são pontos de atenção genéricos de mercado, sem entrada provada ainda — pra a
varredura não pular esses ângulos. Só vira entrada do catálogo geral (`/full-audit` seção 4)
depois de confirmado contra um app real, passando pelo loop de prova-antes-de-corrigir
(`/full-audit` seção 2). Onde já existe achado real confirmado, ver "Confirmado em" no fim
deste arquivo.

## Pontos de atenção

- **Segredo em histórico de git ou no bundle do frontend** — chave/token commitado em algum
  momento (mesmo removido depois, continua no histórico via `git log -p`/`git log -S`), ou
  chave que deveria ser só de backend vazando pro JS que o navegador baixa. Checagem rápida
  já roda antes das camadas (`/full-audit` seção 0.8) — aqui é a versão completa/mais
  profunda, revisando o repo inteiro, não só uma varredura de 2 minutos.
- **Dependências com CVE conhecida** — `npm audit`/`pip-audit`/equivalente da stack. Achado
  vira crítico ativo (seção 0 do orquestrador) só se a CVE for explorável no contexto real do
  app (exposta a input de usuário, não só presente no lockfile); caso contrário, registrar
  como risco a avaliar (atualizar dependência, ou aceitar com justificativa).
- **Segredo em log de build/deploy** — chave/token aparecendo em texto puro na saída do CI,
  do wrangler, ou de qualquer comando de deploy que fica arquivado em algum lugar.
- **Branch de staging/preview acessível publicamente sem gate** quando deveria exigir login
  — dado de teste ou até de produção espelhado num link sem autenticação.
- **Migration aplicada em produção sem passar pelo pipeline** — mudança feita direto no SQL
  Editor ou via `db query` sem migration correspondente no repo; dívida técnica silenciosa
  (mesmo padrão já documentado nos reforços do CLAUDE.md do 8LOOP — o repo vira uma mentira
  sobre o estado real do banco).
- **Rollback de deploy que não reverte migration de banco junto** — código volta pra versão
  antiga, mas o schema já mudou pra versão nova; a versão antiga do código roda contra um
  schema que ela não entende.
- 🔴 **Env var crítica ausente/vazia num deploy rodado de git worktree** — `.env` é
  gitignored, um worktree novo nunca vem com ele; build de framework tipo Vite não falha sem
  a var, só embute `undefined` no bundle final ("sucesso" normal), e o app quebra 100% em
  silêncio só no navegador do usuário real — invisível pra `curl`/HTTP status. Detalhe
  completo e prevenção estrutural (script de verificação em 2 fases: antes do build falha
  rápido, depois do build confirma o valor no bundle gerado) no catálogo geral (`/full-audit`
  seção 4).
- **Função/tabela/trigger vivo em produção sem NENHUMA migration** — não presumir que só 1-2
  escapam; rodar comparação sistemática (script, não grep manual peça por peça) entre TUDO
  que existe no banco vs. TUDO que tem `CREATE FUNCTION`/`CREATE TABLE`/`CREATE TRIGGER` no
  repo. Detalhe completo no catálogo geral (`/full-audit` seção 4).
- **Backup sem teste de restauração, ou apagável por quem tem acesso normal ao banco** —
  "backup automático existe" não é o mesmo que "backup funciona quando precisar". Confirmar
  duas coisas separadas: (1) já rodou uma restauração de teste de verdade (não só o painel
  dizendo "backup concluído"); (2) o backup é imutável — nem um admin comprometido (chave de
  serviço vazada, sessão sequestrada) consegue apagar ou sobrescrever os pontos de
  restauração já existentes, só criar novos. Sem isso, um sequestro de dados que também apaga
  o backup deixa a "proteção" inútil no exato momento em que seria usada.
- **Fornecedor crítico sem plano do que acontece se ele cair ou for comprometido** — mapear
  todo serviço de terceiro do qual o app depende pra funcionar (gateway de pagamento, banco
  gerenciado, CDN/hosting, provedor de e-mail/SMS) e perguntar, um por um: se esse fornecedor
  específico ficar fora do ar por um dia, ou tiver as próprias credenciais comprometidas, o
  que quebra no app, e existe algum plano (mesmo que seja só "aceitar o risco, documentado")?
  Ausência de contrato formal com um fornecedor que tem acesso a dado real do app conta como
  achado aqui, não só indisponibilidade técnica.

**Critério de conclusão:** todo item acima foi checado contra o app real (não só lido e
descartado) — resultado registrado como "não se aplica", "checado, sem achado", ou "achado,
promovido pro catálogo geral com a prova".

## Confirmado em: 8LOOP, 2026-08-25

- Deploy de worktree sem `.env` — app inteiro em branco em produção por ~45min.
- 19 funções + 1 trigger crítico (`on_auth_user_created`, cadastro de usuário novo) vivos sem
  nenhuma migration — achado por script comparando as 80 funções do schema, não descoberto
  aos poucos.
- Chave real do Stripe (`sk_test_...`) commitada no histórico do git — confirmada morta
  (testada em modo só-leitura), sem reescrever histórico (exigiria force-push).

Ver `/full-audit` seção 4 pro detalhe completo de cada um.
