# full-audit

A reusable, 19-layer security, AI-competence, and production-readiness audit methodology, built and battle-tested on real production systems (8LOOP, Kurax).

Not a scanner, not a checklist you run once. It's a **prove-before-fix** method: form a hypothesis, prove the exploit live (safely, with rollback), fix it, re-prove both sides, document. Real findings from this method include a Row-Level Security bug that broke user data isolation, a CORS inconsistency copied across 20 production edge functions, a "dead" AI integration silently calling an external API with no credentials configured, and a personal WHOIS record leaking the operator's name and partial national ID in plaintext — all found, fixed or flagged, and verified.

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
- `full-audit-continuidade/` — backup and business-continuity: is restore actually tested, not just configured
- `full-audit-ia/` — technical competence auditing generative-AI usage itself: is the model call actually live, instrumented, and reviewed — or just wired up and trusted blindly
- `full-audit-reconexterno/` — external black-box recon: what an attacker sees with zero internal access (ports, DNS, TLS, headers, WHOIS)
- `full-audit-desktop/` — native desktop app attack surface (Electron/Tauri/native) — checklist ready, not yet run against a live target
- `full-audit-mobile/` — native mobile app attack surface (iOS/Android/hybrid) — checklist ready, not yet run against a live target
- `full-audit-pentest/` — third-party engagement mode: formal authorization, box-color-based scoping and pricing, client-facing report format

Written as Claude Code skills (Markdown instructions, no attached scripts) — usable with any AI coding agent that can follow structured instructions and execute a proof loop against a real environment.

## Method in one line

Never trust that AI-generated code is correct — or that an AI feature is actually doing what its code implies. Prove it: hypothesis, live exploit or live check with rollback, fix, re-prove, document. Every catalog entry exists because it was proven against a real system, not because it's theoretically possible.
