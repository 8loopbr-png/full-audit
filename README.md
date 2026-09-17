# full-audit

A reusable, 13-layer security and production-readiness audit methodology, built and battle-tested on real production systems (8LOOP, Kurax).

Not a scanner, not a checklist you run once. It's a **prove-before-fix** method: form a hypothesis, prove the exploit live (safely, with rollback), fix it, re-prove both sides, document. Real findings from this method include a Row-Level Security bug that broke user data isolation and a CORS inconsistency copied across 20 production edge functions — both found, fixed, and verified in production.

## Layers

- `full-audit/` — orchestrator: shared scope-alignment, proof loop, validation, and pattern catalog
- `full-audit-dados/` — data layer: RLS policies, auth, isolation
- `full-audit-negocio/` — business logic and backend/money flows
- `full-audit-frontend/` — user-facing write paths
- `full-audit-fuzzing/` — live UI adversarial testing
- `full-audit-gateway/` — payment gateway integration risk
- `full-audit-api/` — pure API / non-web-UI systems
- `full-audit-infra/` — infrastructure, CI/CD, backups, vendor risk
- `full-audit-dominio/` — does the product correctly represent the real-world context it serves
- `full-audit-duplicacao/` — code duplication as a maintainability/latent-bug risk
- `full-audit-security/` — continuous defensive posture + incident response runbook
- `full-audit-conteudo/` — risk by content type (chat, geolocation, photo/video, calendar)
- `full-audit-juridico/` — legal/regulatory exposure by jurisdiction (Brazil + US)

Written as Claude Code skills (Markdown instructions, no attached scripts) — usable with any AI coding agent that can follow structured instructions and execute a proof loop against a real environment.

## Method in one line

Never trust that AI-generated code is correct. Prove it — hypothesis, live exploit with rollback, fix, re-prove, document. Every catalog entry exists because it was proven against a real system, not because it's theoretically possible.
