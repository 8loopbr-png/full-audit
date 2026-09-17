---
name: full-audit-fuzzing
description: Camada 4 do /full-audit — teste exploratório adversarial via agente de navegador (fuzzing de formulários reais). Use quando o usuário pedir "teste exploratório", "fuzzing", "tenta quebrar o app" num app específico, ou testar um formulário/fluxo real com entradas absurdas/maliciosas.
disable-model-invocation: true
---

# Full Audit — UI ao vivo (fuzzing via navegador)

Camada 4 da auditoria completa (`/full-audit`). Pressupõe que a etapa de alinhar escopo
(`/full-audit` seção 0) já rodou nesta sessão — se não rodou, invoque `/full-audit` primeiro.

Diferente das camadas 1-3 (análise estática de código/RLS/prova via transação): aqui um
agente com visão navega o app de verdade, como um usuário tentando quebrar de propósito, e
joga entradas absurdas/maliciosas direto nos campos. Pega bugs invisíveis na leitura de
código — validação client-side que passa mas quebra o backend, XSS armazenado, campo aceita
algo que a regra de negócio não devia, fluxo trava com Unicode/emoji/string vazia/gigante.

**Não se aplica direto quando não há UI web** — app mobile nativo (iOS/Android) ou API pura
sem frontend de navegador não têm onde "navegar". Nesses casos, usar `/full-audit-api` como
equivalente de UI ao vivo: mesmo catálogo de payloads do passo 3 abaixo, disparado direto
contra a API (sem navegador). Registrar essa substituição na seção 0 do orquestrador ao
alinhar escopo — não pular esta camada em silêncio, deixar explícito por que ela virou
`/full-audit-api` naquele app.

## O loop (variante própria — não usa transação+ROLLBACK)

1. **Ambiente e conta** — por padrão, staging (não produção) com uma conta de teste
   **descartável**. Técnica de criação depende do método de login do app:
   - **OTP via Supabase** (padrão do 8LOOP/CORE) — técnica de OTP admin (`generateLink` +
     `email_otp`/`action_link`, sem mandar e-mail real — ver
     [[reference_otp_test_login_technique]]).
   - **E-mail + senha** — criar conta de teste normal com credencial descartável, documentada
     na sessão (não reaproveitar entre auditorias diferentes).
   - **OAuth só (Google/GitHub/etc.)** — usar conta de teste dedicada nesse provedor, nunca
     conta pessoal do dono ou do auditor.
   - **SSO corporativo** — pedir ao dono uma conta de teste já provisionada com papel mínimo;
     nunca usar conta de funcionário real, mesmo com permissão.
   - **Magic link/token custom** — mesma lógica do OTP Supabase: gerar o link via ferramenta
     admin do próprio provedor, sem depender de caixa de e-mail/SMS de verdade, se esse
     caminho existir.

   Em qualquer caso, o objetivo é sempre o mesmo: logar como usuário real sem depender de
   caixa de e-mail/celular de verdade nem de credencial de gente real. Só usar produção
   direto se o dono autorizar explicitamente para aquela sessão. Ao fechar, apagar a conta de
   teste e qualquer dado que ela tenha gravado (cascade delete — ver [[sql_delete_user]] pro
   padrão Supabase; equivalente manual pra outros backends).
2. **Mapear a superfície** — listar todo formulário/campo de entrada do fluxo em escopo
   (nome, texto livre, número, seletor, upload, parâmetro de URL) antes de começar a testar,
   não ir tateando.
3. **Catálogo de payloads** — para cada campo de texto, tentar pelo menos: string vazia, só
   espaços, string gigante (milhares de caracteres), `<script>alert(1)</script>` e variações
   de XSS (verificar se é sanitizado tanto ao salvar quanto ao **exibir** de volta em
   qualquer tela, incluindo painel admin), emoji/Unicode/RTL override, aspas simples e
   duplas soltas (indício de injection se algum caminho monta SQL na mão em vez de usar
   query parametrizada), markdown/HTML cru. Para campos numéricos/preço: negativo, zero,
   decimal absurdo, texto em campo numérico, valor maior que qualquer limite plausível. Para
   seletor/enum: valor fora da lista via manipulação direta (`form_input`/URL) ignorando o
   dropdown. Para parâmetro de URL (`?code=`, `?id=`, etc.): valores de outro registro/
   usuário (checar IDOR — será que troca o parâmetro e vê dado de outra pessoa?).

   **Tamanho do payload "gigante" tem teto:** o suficiente pra provar ausência de limite (ex:
   5-10 mil caracteres), não o maior tecnicamente possível. Em infra compartilhada de
   terceiro pequeno, payload desnecessariamente grande pode pressionar recursos de outros
   tenants na mesma instância — preferir staging sempre que a infra for multi-tenant real, e
   nunca escalar o tamanho só pra "testar o limite de verdade" depois que a ausência de
   validação já foi provada.
4. **Provar o efeito, não só a reação da tela** — se o campo aceitou (sem erro visível),
   confirmar **no banco real** (`supabase db query --linked`) o que foi persistido, e se
   aparece de volta em alguma outra tela (do próprio usuário, de outro usuário, ou do admin)
   sem escapar. Print/console log do navegador (`read_console_messages`) conta como prova de
   erro JS não tratado.
5. **Corrigir onde o dado realmente é validado** — client-side sozinho não é correção (dá
   pra contornar sem passar pela tela, ex: `form_input` direto ou chamada de API manual); a
   validação real tem que estar no backend (RPC/edge function/constraint de banco). Ajuste de
   UI (mensagem de erro, campo trava) é complemento, não substituto.
6. **Reprovar** — repetir exatamente o mesmo payload (deve falhar/ser rejeitado ou
   sanitizado agora) e confirmar que uma entrada legítima normal continua passando. Depois
   de qualquer deploy pra revalidar, checar se o app usa service worker/PWA — cache
   agressivo pode servir o bundle antigo pro navegador de teste mesmo já publicado; limpar
   `caches`/`serviceWorker.getRegistrations()` antes de reprovar se for o caso.

**Freio de segurança (parar, não insistir):** enviar muitos payloads em sequência rápida
pode disparar a própria defesa anti-abuso do app/infra (rate limit, WAF, bloqueio de IP,
CAPTCHA) — o mesmo tipo de gatilho que já rendeu restrição real de conta em outro contexto
(disparo em massa de WhatsApp). Se o app começar a responder com 429, CAPTCHA, ou qualquer
padrão de bloqueio: **isso já é o achado** (documentar que a proteção existe e dispara) —
não tentar contornar, não acelerar de novo pra "terminar o catálogo", e não repetir o teste
várias vezes seguidas contra o mesmo endpoint em produção. Espaçar as tentativas e, se
possível, preferir staging pra essa camada justamente por isso.
7. **Documentar e limpar** — achado no relatório final (não precisa aprovação por item, a
   menos que o dono tenha pedido pausa por achado na seção 0.4 do `/full-audit`); ao final,
   apagar a conta de teste e revalidar que não sobrou lixo na tabela.

## Padrão conhecido específico desta camada

- **Erro de validação sem `data-has-error` — página não rola até o erro** — quando a lógica
  de "rolar até o primeiro erro" depende de um `document.querySelector` procurando um
  atributo marcador (ex: `[data-has-error='true']`), qualquer campo/seção de erro que não
  tenha esse atributo fica invisível pra essa função — o erro é setado e até renderizado no
  DOM, mas o clique no botão de submit "parece não fazer nada" pra quem já rolou a página
  pra baixo. Como achar: listar toda variável de erro (`xErr`) do formulário e conferir se
  cada uma tem um wrapper com o atributo marcador — não vale "a maioria tem", tem que ser
  todas. Achado real: 8LOOP `Register.tsx` — bloco de identificação (RG/naturalidade/pais) e
  bloco de apartamento (convite/torre/andar/apto) tinham a mensagem de erro renderizando
  normal mas sem o atributo, então "Criar conta" parecia travado sem aviso nenhum quando
  essas seções falhavam a validação "tudo ou nada". Promover pro catálogo geral (`/full-audit`
  seção 4) na próxima vez que aparecer em outro app.

**Critério de conclusão:** todo campo mapeado no passo 2 foi testado com o catálogo do passo
3, cada aceitação inesperada foi provada contra o banco/tela real, e a conta de teste foi
removida ao final.

## Confirmado em: 8LOOP, 2026-08-26

Primeira execução real desta camada no 8LOOP (staging, escopo por risco: cadastro →
onboarding bancário → preço de tarefa, não literal). 132+ campos mapeados só nas telas
principais antes de decidir por risco em vez de literal.

1 achado real corrigido: campo `users.name` sem sanitização nenhuma no backend (só existia
máscara no frontend — provado contornando via chamada direta à API REST do Supabase,
autenticado com token real extraído do `localStorage` da própria sessão do navegador). Não
virou XSS na UI porque React escapa por padrão em toda tela (só 1 `dangerouslySetInnerHTML`
no projeto inteiro, alimentado por SVG estático, não por dado de usuário) — mas o mesmo campo
era interpolado sem escapar num **e-mail HTML** (`verify-document`, função `notifyAdmin`),
canal onde a proteção do React não existe. Corrigido reaproveitando `esc()` (já usada em
`notify-lead-event`/`notify-profile-change`, extraída pra `_shared/html.ts` em vez de
duplicada — mesmo padrão de `/full-audit-duplicacao`). Deploy + commit feitos, conta de teste
e function de diagnóstico temporária removidas.

1 verificação positiva confirmada (sem correção necessária): constraint `tasks_value_min`
bloqueou tentativa de INSERT com valor negativo mesmo via SQL bruto direto, ignorando toda a
camada de aplicação — proteção de dinheiro está na camada mais profunda possível.

1 discrepância achada, resolvida no mesmo dia: `tasks_value_max` estava em R$500 (decisão do
dono de 2026-08-25, achada durante `/full-audit-dados`), memória antiga registrava R$1.000.
Dono decidiu reverter pra R$1.000 de propósito (não é a exceção antiga de R$5.000 pras
categorias pausadas) — ver `20260826120000_raise_task_value_ceiling_to_1000.sql`. RLS
`tasks_insert_own` e `tasks_counter_offer_amount_check` atualizadas junto (já divergiram 2x
antes por só uma camada ser corrigida).

### Continuação da mesma sessão — campos de risco restantes (PIX, reembolso, perfil)

Varredura sistemática de todo `${...}` interpolado em e-mail HTML nas 6 edge functions que
constroem HTML (`verify-document`, `notify-lead-event`, `confirm-pix-key-change`,
`request-pix-key-change`, `notify-profile-change`, `request-admin-refund`) — método mais
rápido que testar campo por campo na UI depois do primeiro achado confirmar o padrão.

1 achado real corrigido: `request-admin-refund` interpolava `task.category` (sem validação de
enum no banco, só "não vazio") sem escapar, no e-mail que vai pro **admin** processando
estorno — mesma classe de bug do achado anterior, canal e vítima diferentes (admin, não
usuário comum). Corrigido com o mesmo `esc()` de `_shared/html.ts`.

1 falso alarme meu, corrigido antes de "consertar" à toa: `notify-profile-change` parecia sem
escapar `c.from`/`c.to` numa leitura rápida — na volta com mais contexto, a function já tinha
`esc()` própria aplicada corretamente antes da interpolação. **Lição: ler a function inteira
antes de classificar como achado, não só a linha do `${...}`.**

2 verificações positivas confirmadas, sem correção necessária: (a) `confirm-pix-key-change`
só interpola data formatada pelo servidor, não texto de usuário; (b) `pix_key`/`pix_key_type`
e todos os campos relacionados têm validação de formato robusta na edge function
(`request-pix-key-change`, regex por tipo: CPF/e-mail/telefone/chave aleatória) **e** reforço
por trigger de banco (`protect_user_sensitive_fields`) que reverte qualquer escrita direta
nesses campos vinda de conexão que não seja `service_role` — mesmo contornando a function via
API direta, o valor volta pro antigo sozinho. Não testado ao vivo com conta de teste nesta
rodada (confiança vem da leitura do trigger, não de exploit provado) — revisitar com prova ao
vivo se o assunto voltar.

🔴 **Erro de execução real, corrigido na hora:** ao deployar a correção do
`request-admin-refund`, copiei a flag `--no-verify-jwt` do deploy anterior (`verify-document`)
sem checar o valor original desta function — **exatamente o erro já documentado no
CLAUDE.md do projeto** ("nunca copiar a flag do deploy anterior sem pensar"). Isso derrubou
`verify_jwt` pra `false` por ~1 deploy de intervalo (function que deveria continuar `true`,
usa JWT de usuário real via `getUser(token)`, não segredo compartilhado). Achado e corrigido
no mesmo minuto (redeploy sem a flag, confirmado `verify_jwt: true` restaurado) — mas reforça
que a checagem de flag atual (`supabase functions list -o json` antes de escolher) precisa
virar hábito automático, não só regra escrita.

**Lição de execução, não de achado:** viewport do navegador mudou de largura entre chamadas
de screenshot (1261px → 500px → 666px) no meio da sessão, fazendo clique por coordenada
(x,y) acertar campo errado e produzir dado visualmente "corrompido" que não era bug real, só
desalinhamento. Corrigido trocando pra clique por `ref` (via `read_page`), que segue o
elemento certo independente da largura da tela. **Preferir sempre `ref` a coordenada bruta
em fuzzing de formulário longo**, coordenada só quando não há alternativa.
