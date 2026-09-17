---
name: full-audit-security
description: Checklist do /full-audit pra postura defensiva contínua (rate-limit/detecção de comportamento suspeito em ambos os lados de uma transação bilateral, os 4 sinais dourados de latência/tráfego/erros/saturação, ferramenta terceira — Sentry/analytics/chat de suporte — vazando PII ou expondo sourcemap) + runbook de resposta a incidente em tempo real (o que fazer quando algo já está acontecendo de verdade — dado vazando, abuso confirmado, segurança física — incluindo mensagens-chave pré-escritas por público antes da crise acontecer). Eixo diferente das demais camadas: não é "achar e provar bug pontual", é "quão vigiado o app está continuamente" e "o que você segue quando não dá mais pra só investigar com calma". Use quando o usuário pedir revisão de segurança defensiva, gestão de crise, plano de resposta a incidente, ou perguntar "estamos vigiando isso o suficiente".
disable-model-invocation: true
---

# Full Audit — Segurança defensiva e gestão de crise (checklist + runbook)

Complementa as demais camadas do `/full-audit`. **Diferente de `-dados`/`-negocio`/`-frontend`/
`-fuzzing`** (que caçam um bug específico e provam o exploit antes de corrigir), e diferente de
`-dominio`/`-duplicacao` (que checam representação de domínio e manutenibilidade), esta camada
tem duas partes com naturezas distintas:

- **Parte A** é checklist de **postura contínua** — não "isso está errado agora", mas "existe
  vigilância rodando o tempo todo, ou só descobrimos problema quando alguém tropeça nele?".
- **Parte B** é **runbook operacional** — não roda numa auditoria tranquila, roda **durante**
  um incidente real, quando a prioridade deixou de ser "investigar com calma e provar antes de
  corrigir" (seção 2 do orquestrador) e virou "estancar o dano agora".

**Origem:** gap identificado na revisão defensiva do 8LOOP (2026-08-19) — o app tinha headers
fortes, zero secret hardcoded, webhooks validando assinatura, mas rate-limit só em 5 de ~30
edge functions, antifraude forte só pro lado que **executa** a transação (OA) sem equivalente
pro lado que **paga/pede** (QA), e um protocolo de incidente que existe só como documento
estático, nunca exercitado.

## Parte A — Postura defensiva contínua (checklist, sem entrada provada ainda)

Mesmo padrão de `-gateway`/`-api`/`-infra`: pontos de atenção genéricos de mercado, cada um só
vira entrada do catálogo geral (`/full-audit` seção 4) depois de confirmado contra um app real.

1. **Cobertura de rate-limit / detecção de padrão repetitivo.** Listar **todos** os endpoints
   que escrevem dinheiro, dado sensível, ou mudam estado importante — não só os "óbvios"
   (login, pagamento). Comparar contra os que já têm alguma trava contra repetição rápida
   (mesmo usuário/IP tentando N vezes em pouco tempo). A lacuna típica: a proteção nasce nos
   3-5 endpoints mais discutidos na hora do design e nunca é revisitada quando endpoints novos
   entram depois — rodar esse inventário de novo a cada camada `-negocio`/`-api` concluída,
   não só uma vez.
2. **Detecção de comportamento suspeito nos dois lados de uma transação bilateral.** Se o app
   tem 2+ papéis interagindo (comprador/vendedor, prestador/cliente, morador/prestador), o
   sistema antifraude tende a nascer protegendo só o lado que **executa** o trabalho (é o lado
   que gera disputa/avaliação, então é o lado mais fácil de instrumentar primeiro) — checar
   explicitamente se o lado que **inicia/paga** tem vigilância equivalente: cancelamento
   repetido testando limite, múltiplas contas associadas ao mesmo dado real (endereço,
   documento, cartão), chargeback/estorno abusivo. Ausência aqui não é bug técnico — é gap de
   design que só aparece quando alguém pergunta "e o outro lado?".
3. **Sinais que só existem "no olho" de um admin lendo a tela.** Pra cada painel administrativo
   que mostra dado agregado (lista de disputas, verificações, transações), perguntar: se
   ninguém abrisse essa tela por uma semana, o app perceberia sozinho que algo saiu do padrão?
   Se a resposta é não, não é detecção — é sorte de alguém ter olhado a tempo.
4. **Os 4 sinais dourados, nomeados um por um** (em vez de "monitoramento" genérico): latência
   (as respostas estão ficando mais lentas que o normal, mesmo sem erro nenhum?), tráfego
   (volume de requisição fora do padrão — pico real de uso ou alguém varrendo o app?), erros
   (taxa de falha subindo, mesmo que cada erro isolado pareça pequeno), saturação (banco/fila/
   função batendo no limite de capacidade). Pra cada um, perguntar se existe ALGUMA forma de
   perceber sem abrir uma tela manualmente (alerta automático, painel do provedor de
   infraestrutura) — os 4 juntos cobrem a maioria dos "algo está errado" antes de virar
   incidente confirmado; faltar um deles é ponto cego específico, não genérico.
5. **Ferramenta terceira que recebe dado do app manda mais do que deveria.** Qualquer serviço
   de monitoramento de erro, analytics, gravação de sessão ou chat de suporte embutido no app
   tem, de fábrica, a opção de capturar dado de mais (IP, cookies, headers inteiros, payload
   cru de request/response) — a configuração "confortável" do fornecedor quase sempre favorece
   captura ampla, não privacidade. Checar, pra cada ferramenta terceira integrada: (a) existe
   uma flag de "mandar tudo" (ex: `sendDefaultPii` do Sentry, ou equivalente de outra lib)
   ligada sem necessidade real? (b) existe uma camada de filtro (`beforeSend` ou equivalente)
   removendo campo sensível — senha, token, documento, chave de pagamento — antes do evento
   sair do app, ou só se confia que ninguém vai passar isso sem querer num `captureException`/
   `setContext` futuro? (c) se o build gera **sourcemap** (mapa que reconstrói o código-fonte
   original a partir do bundle minificado), ele fica público no pacote final entregue ao
   usuário, ou é enviado só pro fornecedor (via plugin dedicado, ex: `@sentry/vite-plugin`) e
   removido do que sai pro ar? Sourcemap público republica o código-fonte inteiro do app pra
   qualquer visitante que abrir o DevTools. Comando genérico pra achar, adaptar por stack:
   grep pelo nome da lib de monitoramento (`Sentry.init`, `LogRocket.init`, `mixpanel.init`,
   etc.) cruzado com nome de campo sensível; `sourcemap` no config de build; e depois de um
   build real, `find dist -name "*.map"` (ou equivalente da stack) pra confirmar que nada ficou
   exposto no pacote final.

## Parte B — Runbook de resposta a incidente (tempo real, não auditoria)

Diferente de todo o resto do `/full-audit`: isto não é seguido numa sessão de revisão calma —
é o que você segue **enquanto** algo real está acontecendo (dado de outra pessoa exposto agora,
abuso confirmado em andamento, risco à segurança física de alguém). Estrutura fixa, adaptar o
detalhe por app, manter o esqueleto:

1. **Classificar em menos de 1 minuto**, porque a classificação decide a velocidade de tudo
   depois: (a) segurança física de pessoa — prioridade máxima, prazo de minutos; (b) dado
   pessoal exposto/vazando agora — prazo de horas, e se for dado pessoal de terceiro checar
   prazo legal de notificação (LGPD/ANPD, quando aplicável); (c) dinheiro sendo perdido nesse
   instante — prazo de horas; (d) abuso de padrão confirmado mas sem urgência de minuto (ex:
   fraude sistemática já detectada, mas contida) — segue ritmo normal, vira item do catálogo.
2. **Congelar primeiro, investigar depois** — a ação que estanca o dano crescendo vem antes de
   entender a causa raiz completa. Em ordem de escopo (menor primeiro, só sobe se não for
   suficiente): suspender a conta/sessão específica → revogar a chave/token específico →
   pausar a função/endpoint específico → só em último caso, algo mais amplo. Nunca pular direto
   pro mais amplo por reflexo — isso também é um custo (usuário legítimo impactado).
3. **Quem aciona e em quanto tempo** — decidir isso **antes** de precisar (não durante):
   o dono decide sozinho, ou existe alguém mais (advogado, suporte, sócio) que precisa ser
   acionado por classificação? Registrar prazo-alvo por categoria da etapa 1.
4. **Comunicação** — nunca admitir culpa antes de revisão jurídica, nunca tentar "resolver por
   dentro" escondendo de quem foi afetado. Se o incidente envolve dado pessoal de terceiro,
   checar a obrigação legal de notificação (no Brasil, LGPD/ANPD tem prazo próprio — não
   assumir que "corrigir rápido" dispensa notificar). **As mensagens-chave são escritas
   ANTES da crise, não durante** — pra cada público que o app tem (ex: 8LOOP: síndico,
   morador, imprensa), um rascunho curto e pré-aprovado do que dizer num incidente real
   (o que aconteceu, o que já foi feito, o que a pessoa precisa fazer, se precisar de algo).
   Escrever a mensagem pela primeira vez em cima da hora, sob pressão, é onde nasce a
   promessa exagerada ou a admissão de culpa precoce que a revisão jurídica depois não
   consegue mais desfazer.
5. **Preservar evidência antes de corrigir** — travar edição/exclusão dos registros envolvidos
   no incidente antes de aplicar a correção técnica; corrigir o bug que causou o problema pode
   apagar, sem querer, a prova de que ele aconteceu (relevante inclusive pra diligência
   jurídica — ver achado do 8LOOP sobre ausência de evidência retida de verificação de
   antecedentes).
6. **Depois do fogo apagado, volta pro método normal** — a causa raiz do incidente entra no
   loop prova-antes-de-corrigir padrão (`/full-audit` seção 2), com o mesmo rigor de qualquer
   outro achado. O runbook acima é só a parte "agora"; não substitui a correção real, e o
   incidente vira entrada nova no catálogo (seção 4) se o padrão puder se repetir em outro app.

**Critério de "pronto" pro runbook (diferente do resto do skill):** não é "documento existe" —
é "alguém já leu/ensaiou isso de verdade nos últimos N meses". Um runbook nunca exercitado tem
o mesmo risco que um extintor de incêndio nunca checado: pode estar vencido sem ninguém saber
até precisar dele. Sugestão de gatilho: reler/ensaiar mentalmente (ou revisar contra um caso
hipotético novo) a cada 6 meses, mesmo prazo sugerido pra reauditoria de `-dados`/`-negocio`/
`-gateway` (orquestrador seção 5.2).

## Confirmado em: 8LOOP, 2026-08-19

- **Cobertura de rate-limit concentrada nos endpoints óbvios** [negócio/dado, moderado] —
  **RESOLVIDO em 8LOOP, 2026-08-25**: reconferido e achado 30 de 34 edge functions com
  proteção (era 5 de ~30 em 19/08) — cresceu organicamente ao longo de várias sessões
  seguintes que foram fechando endpoint por endpoint. Os 4 sem rate-limit são todos isentos
  por motivo real: 2 são webhook de gateway (validação de assinatura criptográfica é proteção
  mais forte que rate-limit), 1 tem rate-limit próprio por IP feito de propósito separado
  (endpoint público sem login), 1 está desativado (sempre `410`). Continua valendo reconferir
  a cada endpoint novo — o padrão do achado original (fricção nasce nos óbvios, endpoint novo
  esquece) ainda é real, só não está mais confirmado neste app hoje.
- **Antifraude desenhado só pro lado que executa a transação** [negócio, moderado] — sistema de
  disputa/alerta (`nivel_alerta`/`disputas_30_dias`) é forte pro lado OA (prestador), sem
  equivalente pro lado QA/morador (cancelamento repetido testando limite, múltiplas contas no
  mesmo apartamento, chargeback abusivo). **Parcialmente resolvido em 8LOOP, 2026-08-25**:
  chargeback (handler novo no webhook do gateway), comprovante de conclusão falso/reusado, e
  cancelamento repetido do QA (mesma vigilância do OA, espelhada, sem penalidade automática —
  só registro + fila de revisão) foram fechados. "Múltiplas contas no mesmo apartamento" segue
  sem cobertura própria pro lado QA (existe só indiretamente via limite de vaga de apartamento
  no cadastro).
- **Runbook de incidente existe só como documento estático, nunca exercitado** [gestão de
  crise] — `INCIDENT_RESPONSE.md` do 8LOOP cobre o desenho (verificação de antecedentes
  criminais do OA implementada), mas falta: categoria de denúncia "Emergência/Segurança física"
  com 190/180 antes do formulário, suspensão automática imediata pra esse tipo de denúncia
  (diferente da regra "≥5 disputas", que é pra fraude financeira, não física), trava contra
  edição/exclusão de evidência após denúncia aberta.
- **Ferramenta terceira (item 5) — checado em 8LOOP, 2026-09-08: sem achado, config correta.**
  Único ponto de `Sentry.init(` (`src/lib/sentry.ts`), `sendDefaultPii: false` explícito e
  comentado como obrigatório, `beforeSend` fazendo scrub de campo sensível (senha/token/
  pix_key/cvv/cartão/cpf) em `extra`/`contexts` + remoção de cookies/headers do request, os 5
  usos de `captureException` no app só passam `tags`/`extra` inofensivos (gateway de pagamento,
  task_id). Sourcemap não configurado no build (Vite não gera em produção por padrão sem ligar
  explícito) — `@sentry/vite-plugin` não instalado, consistente. `find dist -name "*.map"` após
  build real: vazio. Achado por pedido cruzado de outra sessão (Kurax, auditoria de outro
  sistema) que motivou promover isso de checklist ad-hoc pra item formal desta camada.

## Critério de conclusão

Parte A: cada item está em um de três estados — revisado sem achado, achado promovido pro
catálogo geral com plano de correção, ou risco aceito registrado com motivo. Parte B: runbook
documentado com dono/prazo definidos por categoria, mensagens-chave pré-escritas existindo
por público (não só planejadas pra escrever "quando precisar"), e data da última vez que foi
lido/ensaiado de verdade (não só escrito uma vez).
