---
name: full-audit
description: Orquestrador de auditoria completa de um app em produção (banco/backend/frontend/UI ao vivo) — acha bugs reais, prova o exploit antes de corrigir, corrige, valida, e documenta tudo. Alinha escopo, chama as camadas full-audit-dados/negocio/frontend/fuzzing na ordem certa, e mantém o método compartilhado (loop de prova, catálogo de padrões, fechamento). Use quando o usuário pedir "auditoria 100%", "analisa o app inteiro", "acha e corrige tudo".
disable-model-invocation: true
---

# Full Audit

Auditoria de app em produção, ponta a ponta. Nasceu da auditoria do 8LOOP (2026-07-20/21),
que achou uma cadeia de bugs críticos invisíveis (adulteração de preço, proteção de dado
sensível ausente, XP nunca creditado, feature quebrada desde sempre) só porque cada achado
foi **provado** antes de virar "corrigido".

O princípio central: nunca reportar um achado como bug, nem uma correção como funcionando,
sem prová-lo contra o estado real (banco/API ao vivo) primeiro. "Rodei e não deu erro" não
é prova. Esse princípio e o método das seções abaixo (0, 2, 3, 5) são compartilhados por
todas as camadas — não duplicar em cada skill de camada, sempre voltar pra cá.

**Estrutura (2026-08-13, quebrado em camadas):** este arquivo é o orquestrador — alinha
escopo, define a ordem, guarda o método comum e o catálogo de padrões. Cada camada da
varredura é um skill próprio, invocável sozinho quando o pedido já é específico (ex: "audita
só o RLS desse app" → `/full-audit-dados` direto, sem passar por aqui).

## Mapa da auditoria (Início → Meio → Fim)

Cada rótulo abaixo (0.1, 1.1.1, etc.) é referenciável em qualquer skill da família — **toda
vez que este orquestrador ou uma camada for invocado, comece conferindo esta lista contra o
que já está registrado em memória/arquivo de progresso**, pra saber em que ponto a auditoria
está sem precisar reler tudo de novo. Marcar cada linha como feita (✅), em andamento, ou
não se aplica, ao salvar o estado (seção 5).

**INÍCIO — alinhar escopo (seção 0)**
- [ ] 0.1 De quem é o app
- [ ] 0.2 Literal ou por risco
- [ ] 0.3 Arquivos/áreas travadas
- [ ] 0.4 Autonomia
- [ ] 0.5 Acesso
- [ ] 0.6 Perfil de ameaça
- [ ] 0.7 Quais camadas condicionais entram (decide aqui, usando a tabela da seção 1.2)
- [ ] 0.8 Varredura barata inicial (OBRIGATÓRIA: segredo exposto + `npm audit` + advisor
      da plataforma gerenciada — bloqueia entrada na seção 1)

**MEIO — executar as camadas, sempre pelo método da seção 2**
- [ ] 1.1.1 `full-audit-dados`
- [ ] 1.1.2 `full-audit-negocio` (+ `-gateway` se o app tiver pagamento)
- [ ] 1.1.3 `full-audit-frontend`
- [ ] 1.1.4 `full-audit-fuzzing`
- [ ] 1.1.5 Arquivos/áreas travadas (só se 0.3 destravou)
- [ ] 1.2.* Camadas condicionais decididas em 0.7 — uma linha por camada aplicável (ex:
      "1.2.3 `full-audit-dominio` ✅")

**FIM — validar e fechar**
- [ ] 3. Validação final (suíte automatizada + fluxo E2E real, se existir)
- [ ] 3.1 Revisão independente (OBRIGATÓRIA pra achado de segurança/pagamento/dado sensível —
      subagente sem contexto, no mínimo)
- [ ] 4.5 Cobertura sistemática (OWASP Top 10:2025) atualizada com o que esta rodada checou
- [ ] 5. Fechar a sessão (segredo removido, resumo por estado, salvo em memória)
- [ ] 5.2 Gatilho de reauditoria registrado

(Seção 4 — catálogo de padrões — não é uma etapa própria, é referência consultada durante
todo o MEIO e alimentada durante todo o FIM; não faz parte deste checklist de progresso.)

## 0. Alinhar escopo antes de começar

"100%" é ambíguo — pode significar cobertura literal (linha por linha, todo arquivo) ou
cobertura por risco (para quando o retorno marginal cai). Rode uma sessão curta no estilo
`grilling` para fechar, na ordem:

### 0.1 De quem é o app

Seu ou de terceiro? Se for de terceiro: (a) confirmar autorização
por escrito pra testar, mesmo que informal (e-mail/mensagem já basta), antes de tocar em
produção; (b) confirmar que você realmente tem o direito de estar olhando esse código/
banco — "fui convidado a olhar" não é o mesmo que "posso auditar e cobrar por isso", não
assumir propriedade sem checar; (c) travar a regra abaixo, que vale em **toda** camada
(1.1.1-1.1.4), não só na de fuzzing: qualquer teste cross-tenant (uma conta vendo/afetando
dado de outra) usa sempre **duas contas de teste suas, descartáveis** — nunca dado real de
um cliente do dono do app. Provar um IDOR trocando o ID por um registro real de terceiro é
expor dado real de outra pessoa, não um teste. Se for seu próprio app, (a) e (b) não se
aplicam, mas (c) continua valendo.

### 0.2 Literal ou por risco

Se por risco, qual critério de parada (lista de itens fechada, ou "até o dono decidir")?

### 0.3 Arquivos/áreas travadas

Existe lista de "não mexa"? Pedir autorização explícita antes de abrir esses arquivos,
mesmo só pra ler-e-reportar.

### 0.4 Autonomia

O dono quer aprovar achado por achado, ou autoriza corrigir e só reportar depois? Isso muda
o ritmo do resto da auditoria inteira.

### 0.5 Acesso

Chave de serviço/banco disponível nesta sessão? Nunca pedir de novo sem necessidade; se for
fornecida em texto puro, nunca ecoar o valor em log/saída, e remover do ambiente ao final.
Registrar também se existe acesso a **transação de banco (Postgres/MySQL)** ou não — muda o
método da seção 2 antes de começar, não no meio.

### 0.6 Perfil de ameaça

Quem tende a atacar esse app, especificamente? Não genérico ("hackers") — listar os perfis
plausíveis desse contexto real (ex: 8LOOP: morador mal-intencionado, prestador golpista,
ex-funcionário, concorrente tentando raspar dado; Kurax em modo pentest: perguntar isso ao
cliente, não presumir). Isso muda a prioridade das camadas seguintes: um app onde o perfil
mais provável é "vizinho oportunista" pesa mais pra `-negocio`/`-dominio` (abuso de regra de
negócio, contorno de processo) do que pra exploit técnico sofisticado; um app exposto a
atacante profissional pesa mais pra `-dados`/`-fuzzing`. Registrar a lista — ela guia onde
investir tempo primeiro dentro da seção 1, não é só formalidade.

Não prossiga pras camadas até 0.1-0.6 estarem registradas (em memória ou num arquivo de
progresso) — sem isso, a auditoria não tem como saber quando está "pronta".

**Exceção que fura a fila (independe da resposta de 0.4):** um achado **crítico ativo** —
dado de outro usuário legível agora, segredo exposto, dinheiro sendo perdido nesse instante —
é reportado assim que confirmado, não guardado pro relatório final. "Crítico ativo" é
diferente de "achado grave mas estático" (ex: uma policy errada que *permitiria* o exploit,
mas ninguém está explorando agora) — esse segundo caso segue o ritmo normal de 0.4.

**Critério de conclusão de 0.1-0.6:** as 6 perguntas têm resposta explícita do dono,
registrada por escrito.

### 0.7 Quais camadas condicionais entram

Com 0.1-0.6 respondidas (principalmente 0.6, perfil de ameaça, e a natureza do app), decidir
**agora, de uma vez**, quais camadas condicionais da tabela em 1.2 se aplicam — não descobrir
isso no meio da execução das camadas obrigatórias (1.1), o que forçaria reabrir contexto já
fechado. Registrar a lista decidida junto das respostas de 0.1-0.6.

**Critério de conclusão:** cada camada condicional da tabela 1.2 tem uma decisão explícita —
"entra" (com a posição na sequência) ou "não se aplica" (com o motivo).

### 0.8 Varredura barata inicial (OBRIGATÓRIA, bloqueia entrada na seção 1)

Antes de entrar nas camadas pesadas, dois minutos de checagem barata que costumam achar o
achado mais fácil e mais grave primeiro — não deixar escondido atrás de bugs sutis de lógica.
**Isto não é opcional nem "boa prática" — é a diferença entre auditoria manual sozinha e o
padrão real de mercado (scanner automatizado primeiro, análise manual depois). Nenhuma
camada da seção 1 começa antes dos itens abaixo rodarem e o resultado ser registrado**, sem
exceção mesmo em auditoria rápida/parcial:

- **Segredo exposto** — grep por padrão de chave/token no histórico do git (`git log -p` ou
  `git log -S`) e no bundle JS gerado do frontend (chave que deveria ser só de backend
  vazando pro client). Achado aqui vira item crítico ativo (ver exceção acima), não espera o
  relatório final.
- **Dependências com CVE conhecida** — rodar `npm audit`/`pip-audit`/equivalente da stack
  pelo menos uma vez. Detalhe completo (o que fazer com o resultado) fica em
  `/full-audit-infra`.
- **Advisor nativo da plataforma gerenciada** — se o app usa Supabase (ou equivalente com
  advisor próprio), rodar `mcp__plugin_supabase_supabase__get_advisors` (categorias
  `security` e `performance`) antes de qualquer leitura manual de RLS/policy — ele já varre
  determinísticamente padrões conhecidos (RLS ausente, `SECURITY DEFINER` sem
  `search_path`, índice faltando em FK de tabela grande) que seria desperdício redescobrir
  lendo migration por migration. Achado real: view sem RLS discutida na seção de catálogo
  (4) foi achada assim, não por leitura manual.
- **Função/bloco copiado entre arquivos, com hash divergente** — grep por definições de
  função que aparecem em 3+ arquivos (comum em backend sem pasta compartilhada, ex: CORS
  repetido por edge function) e comparar hash de cada cópia. Qualquer cópia com hash
  diferente da maioria é uma cópia já desviada silenciosamente — achado confirmado só pela
  leitura de código, sem precisar de prova ao vivo. Barato e já achou bug real (ver catálogo,
  seção 4). Detalhe completo em `/full-audit-duplicacao`.
- **Superfície viva vs. superfície versionada** — toda auditoria até aqui parte de ler o repo,
  o que tem um ponto cego óbvio: código/tabela implantado direto em produção sem nunca ter
  sido commitado é **invisível** pra qualquer camada que só lê arquivo. Comparar a lista viva
  contra o repo antes de confiar que "o repo é a verdade": `supabase functions list -o json`
  (ou equivalente da plataforma) vs. `ls supabase/functions/`, e a lista de tabelas do banco
  (`information_schema.tables`) vs. `CREATE TABLE` nas migrations. Qualquer função/tabela só
  no lado vivo é achado confirmado (não precisa prova ao vivo, é comparação determinística) —
  baixar o código real antes de julgar severidade (pode estar com RLS/auth corretos mesmo sem
  estar versionado, o problema é a ausência de trilha de revisão, não necessariamente um
  exploit). Achado real: `send-engagement` (8LOOP) rodando em produção desde 2026-06-12, zero
  histórico em qualquer branch — ver catálogo seção 4.

Registrar resultado (limpo, ou achado promovido) antes de seguir pra seção 1.

## 1. Ordem das camadas

### 1.1 Camadas obrigatórias (nesta ordem, sempre)

Cada camada expõe suposições que a próxima usa sem verificar — seguir nesta ordem:

1.1.1. **`/full-audit-dados`** — RLS/policies, autenticação, colunas sensíveis.
1.1.2. **`/full-audit-negocio`** — lógica de backend/funções que envolvem dinheiro ou dado
   sensível. Complementar com `/full-audit-gateway` se o app tiver gateway de pagamento.
1.1.3. **`/full-audit-frontend`** — touchpoints de escrita nas telas (insert/update/RPC).
1.1.4. **`/full-audit-fuzzing`** — UI ao vivo, teste exploratório adversarial via navegador.
1.1.5. **Arquivos/áreas travadas** — só depois de autorização explícita (item 0.3). Não é
   uma camada própria — reaplica o loop de prova (seção 2) e o padrão da camada relevante
   (1.1.1-1.1.4) sobre esses arquivos, sem relaxar o rigor por já ter permissão.

### 1.2 Camadas condicionais (decidir em 0.7, não no meio da execução)

Cada linha entra na sequência acima quando o gatilho da coluna 2 se aplica ao app. Decisão
registrada em 0.7 — não descoberta ad-hoc enquanto 1.1 já está em andamento.

| # | Camada | Gatilho | Onde entra |
|---|---|---|---|
| 1.2.1 | `/full-audit-api` | API própria fora do padrão Supabase | Junto com 1.1.2-1.1.3 |
| 1.2.2 | `/full-audit-infra` | Pipeline de deploy (quase sempre relevante) | Depois de 1.1.4 |
| 1.2.3 | `/full-audit-dominio` | Produto modela contexto físico/social real (condomínio, bairro, comunidade — usuário vive uma vida fora da tela) | Depois de 1.1.2 |
| 1.2.4 | `/full-audit-duplicacao` | Pedido explícito de "enxugar"/reduzir gordura/achar duplicação | Independente, qualquer momento |
| 1.2.5 | `/full-audit-security` | Pedido sobre postura defensiva contínua ou gestão de crise/resposta a incidente | Depois de 1.1.4 |
| 1.2.6 | `/full-audit-conteudo` | App tem chat/mensagem em tempo real, mapa/geolocalização, foto/vídeo ou agenda/calendário | Junto com 1.1.3 |
| 1.2.7 | `/full-audit-juridico` | Pedido sobre redução de risco jurídico (trabalhista/civil/criminal/consumidor) ou conformidade | Depois de 1.1.2 |
| 1.2.8 | `/full-audit-pentest` | App auditado **não é seu** — cliente pagando pelo serviço | **Substitui a seção 0 inteira** como modo de entrada (autorização formal por escrito, escopo por pacote/preço, confidencialidade do achado do cliente) |

Cada eixo condicional é diferente dos demais — não é "mais do mesmo tipo de bug", é uma
lente própria (ex: `-dominio` não caça bug de código, caça lacuna entre o que o produto
representa e o mundo real; `-juridico` não caça bug nem manutenibilidade, caça "isso vira
evidência contra o app numa ação real?"). Detalhe de cada uma no próprio skill.

Cada skill de camada assume que 0.1-0.8 acima já rodaram nesta sessão.

**Critério de conclusão de uma camada:** todo item do escopo dela está em um de três
estados — revisado sem achado, achado com correção provada, ou achado registrado como risco
aceito com justificativa escrita. Nenhum item fica "pendente" sem essa etiqueta.

## 2. O loop prova-antes-de-corrigir (método compartilhado)

Para cada suspeita de bug (camadas 1-3):

1. **Hipótese** — o que a leitura do código sugere que está errado.
2. **Provar o exploit ao vivo, sem persistir** — simular a sessão real (papel/token do
   usuário, não superusuário) dentro de uma transação que termina em ROLLBACK. Só conta como
   provado se o efeito indesejado realmente aconteceu nessa simulação.
3. **Aplicar a correção como instrução própria, fora de qualquer transação de teste** —
   nunca dentro do mesmo bloco que vai sofrer ROLLBACK; a correção precisa sobreviver
   sozinha. (Erro fácil de cometer: testar `CREATE OR REPLACE FUNCTION` dentro do bloco de
   prova e descobrir depois, numa consulta separada, que a versão antiga continua ativa
   porque o ROLLBACK desfez a correção junto.)
4. **Reprovar os dois lados** — repetir o passo 2 (exploit deve falhar agora) e confirmar
   que o uso legítimo continua funcionando.
5. **Documentar o achado no artefato da correção** (migration/commit/PR): o que estava
   errado, como o exploit foi confirmado, o que foi mudado, o que foi validado — não só "fix
   bug X".

**Critério de conclusão do loop:** passos 2 e 4 têm resultado observado (não assumido) para
aquele item específico.

**Cuidado com efeito fora do banco:** ROLLBACK desfaz linhas do Postgres, não desfaz uma
chamada real a um serviço externo (gateway de pagamento, envio de e-mail/SMS, webhook de
terceiro). Se a função sob teste chama uma API externa de verdade, provar "dentro da
transação" não é suficiente — a chamada externa já aconteceu antes do ROLLBACK rodar.

**Variante offline/sandbox (quando a ação sob teste chama um serviço externo de verdade):**
o princípio é sempre o mesmo, não é exclusivo de gateway — usar a **credencial/ambiente de
teste do próprio provedor** (chave sandbox, número de teste, webhook de staging), nunca
disparar a chamada real contra a conta de produção só pra provar um exploit:

1. Trocar a credencial usada no teste pela de sandbox/teste do provedor (Stripe/MP têm
   modo test explícito; Resend/Twilio-like normalmente têm domínio/número de teste ou
   endpoint que não entrega de verdade — checar a doc do provedor específico antes de
   assumir que existe).
2. Rodar a ação suspeita contra esse modo de teste — o loop de prova continua igual (passos
   1, 2, 4, 5 da seção 2), só o passo 2 usa a credencial de teste em vez de transação.
3. Se o app **não distingue** credencial de teste de credencial real na função sob teste
   (ou se o bug é justamente nisso — ex: chave sandbox aceita em produção sem aviso),
   documentar isso como achado à parte antes de prosseguir — é pior que o bug original.
4. Se o provedor específico não tiver modo sandbox nenhum (raro, mas existe), cair pra
   variante "sem transação de banco" abaixo, com o cuidado extra de que o passo 5
   (reverter manualmente) ali não desfaz o efeito externo já disparado — só documentar.

`/full-audit-gateway` é a primeira instância concreta desta variante (pagamento), não a
definição dela — qualquer camada nova que envolva serviço externo com efeito real (e-mail,
SMS, push, webhook de terceiro) usa o mesmo princípio, mesmo sem skill de camada própria
ainda escrita pra ela.

A camada `/full-audit-fuzzing` (UI ao vivo) usa uma **variante própria** desse loop — não dá
pra usar transação+ROLLBACK contra um navegador renderizado de verdade. Ver o loop completo
lá.

**Variante sem transação de banco (app não-Postgres, ou só chave de API/staging sem acesso a
banco):** quando a resposta de 0.5 for "sem acesso a transação", os passos 2-4
mudam:

1. Criar/usar uma conta de teste descartável (nunca conta real de terceiro).
2. Capturar o estado **antes** da ação (screenshot, resposta de GET/consulta via API, ou
   export do registro) — sem esse "antes", não dá pra provar que a ação mudou algo.
3. Executar a ação suspeita normalmente (sem transação envolvida).
4. Capturar o estado **depois** pela mesma via, e comparar com o "antes" — essa comparação é
   a prova, no lugar do ROLLBACK confirmando o efeito.
5. **Reverter manualmente** (delete via API/admin, ou instrução SQL separada se houver algum
   acesso de escrita mesmo sem transação) — como não existe desfazer automático, checar
   explicitamente ao final que não sobrou lixo de teste, com o mesmo rigor do passo "limpar
   conta de teste" da camada de fuzzing.

Sem esse passo 5 documentado, dado de teste fica esquecido em produção do jeito que a
transação normalmente evitaria sozinha — é o ponto onde essa variante mais falha se for
feita apressada.

## 3. Validação final

Depois que uma camada inteira estiver marcada (seção 1), rode a suíte de testes automatizada
disponível.

Quando um teste falhar, **antes de tratar como bug do app**, investigue diretamente se a
causa é o teste estar desatualizado — coluna morta, fluxo que mudou, campo de resposta que a
API nunca prometeu devolver. As duas hipóteses ("app está errado" e "teste está errado")
merecem o mesmo nível de investigação; não vale assumir a primeira só porque é mais
interessante de corrigir. Corrija o que estiver realmente errado (app ou teste) e rode de
novo até a suíte refletir o comportamento real.

Se houver um fluxo real ponta-a-ponta que só roda sob flag explícita (ex: cria cobrança real
num provedor de pagamento) e ele estiver disponível, rode-o pelo menos uma vez antes de
fechar a auditoria — é a única forma de provar que a cadeia completa (não só peças isoladas)
funciona.

**Planejar o teste E2E é parte da prova, não só executá-lo** (achado 8LOOP, 2026-08-25):
ao desenhar como testar de verdade uma correção anti-fraude baseada em "cruzar campo X entre
duas contas", ficou claro que uma constraint de unicidade no banco tornava o próprio cenário
de teste impossível de montar (duas contas nunca podem ter o mesmo valor naquele campo,
então a proteção nova nunca vai disparar na prática) — a correção continuava "certa" em
isolamento, mas cobria bem menos do caminho real de burla do que o relatório original
descrevia. Isso só apareceu tentando montar o teste, não relendo o código de novo. Se ao
planejar a prova de um achado o cenário de teste parecer estranhamente impossível de montar
com dado real, não force um substituto artificial — pare e pergunte por que, é sinal de que
o entendimento original do achado (ou da correção) estava incompleto.

**Critério de conclusão:** suíte automatizada rodando limpa, e todo fluxo end-to-end real
disponível foi exercitado pelo menos uma vez com resultado observado.

### 3.1 Revisão independente (olho de fora) — OBRIGATÓRIA pra achado de segurança/pagamento/dado sensível

Quem construiu (ou corrigiu) o código tem viés — aceita as próprias premissas de design sem
perceber. **Toda correção que mexeu em RLS, trigger de proteção, fluxo de pagamento, ou dado
sensível (CPF/PIX/documento) precisa passar por um segundo par de olhos antes de considerar
a auditoria fechada**, mesmo quando o loop de prova (seção 2) já confirmou o exploit fechado:

1. **Revisão via subagente sem contexto prévio** — depois de aplicar e provar a correção,
   invocar um agente novo (`Agent` tool, tipo `general-purpose`, **nunca** `fork` — `fork`
   herda todo o contexto desta sessão, o que anula o propósito de olho de fora) passando só
   o diff/arquivo final, sem a hipótese nem o histórico de como foi achado. Pedir
   explicitamente: "revise este código em busca de problema de segurança, sem saber o que eu
   já testei — não confirme minha conclusão, ache a sua própria." Se ele apontar algo novo,
   voltar ao passo 2 da seção 2 antes de fechar. Se não apontar nada, registrar isso também
   (é sinal, não é nada).
2. **Segunda ferramenta/modelo diferente, quando disponível** — se houver acesso a uma IA de
   procedência diferente rodando em paralelo (ex: outro CLI/modelo, não só outra sessão do
   mesmo Claude Code), rodá-la sobre o mesmo escopo antes de fechar é a camada de calibração
   mais forte disponível sem custo externo — modelos diferentes erram em padrões diferentes.
   Cada achado dela entra pelo mesmo loop de prova (seção 2) antes de confiar na descrição
   literal — ela pode apontar a área certa com o detalhe impreciso.
3. **Pentest humano externo, periódico** — nenhuma das duas camadas acima substitui uma
   revisão humana de fato independente (sem vínculo com quem constrói o app) — ver
   `/full-audit-pentest` pra pacote/preço de mercado. Não é obrigatório em toda auditoria
   (custa dinheiro e tempo de terceiro), mas convém repetir periodicamente (ex: 1x/ano) — é o
   único item desta lista que calibra se os pontos cegos das camadas 1-2 acima estão se
   repetindo sem que ninguém perceba.

**Critério de conclusão:** todo achado de segurança/pagamento/dado sensível desta rodada
passou pelo menos pelo passo 1 antes de ser marcado como fechado.

## 4. Catálogo de padrões conhecidos (cresce a cada auditoria, compartilhado entre camadas)

Antes de começar qualquer camada, releia esta lista e procure ativamente por cada padrão —
não espere tropeçar neles por acaso. Depois de fechar uma auditoria com achado novo que não
está aqui, **adicione o padrão nesta lista antes de encerrar a sessão** (ou no skill da
camada correspondente, se o padrão for específico dela) — é assim que auditorias futuras
(neste projeto ou em qualquer outro) ficam mais rápidas em achar o mesmo tipo de bug.

**Catálogo é piso, não teto.** Toda entrada abaixo nasceu de um app real, mas de uma fonte
concentrada — não presuma cobertura completa só porque nenhum item da lista bateu. Isso
importa mais em duas situações: (1) app de stack diferente da que gerou o catálogo (hoje,
majoritariamente Postgres/Supabase/TypeScript) — riscos típicos de outra stack (ex: mass
assignment em Rails, regra do Firestore) não têm entrada aqui ainda; (2) app do mesmo motor/
CORE que já gerou entrada aqui — mesmo compartilhando código, pode ter bug novo que a fonte
original nunca expôs. Reler o catálogo é o ponto de partida, nunca o critério de parada —
sempre gerar hipótese nova a partir do código real em auditoria, além dele.

Formato de cada entrada: **nome curto** [severidade: dinheiro/dado/feature/estético] —
como reconhecer · como confirmar rápido · [camada] · confirmado em: app, data. A severidade
e o "confirmado em" existem pra o catálogo continuar navegável conforme cresce — sem isso,
depois de 20+ entradas fica difícil saber o que priorizar reler primeiro numa auditoria nova.
Entradas já existentes abaixo (anteriores a esta regra) podem ficar sem a tag até serem
tocadas de novo; não é preciso retroagir só por causa disso.

- **Guarda que nunca dispara na prática** [negócio] — uma condição de proteção (`IF x IS NOT
  NULL THEN RETURN`, early-exit, etc.) que parece certa lendo isolada, mas o valor que ela
  testa é preenchido em 100% das chamadas reais do app — a proteção nunca roda de verdade, é
  código morto disfarçado de código ativo. Como achar: pra cada trigger/função de validação,
  grep no frontend pra ver se o campo testado na condição de saída é sempre preenchido nas
  telas reais (não só nas de teste/dev). Confirmar com prova ao vivo simulando exatamente o
  payload que a tela real manda — nunca um payload "limpo" inventado.
- **Lógica duplicada em gêmeos divergindo** [negócio] — a mesma regra de negócio (ex:
  aprovação manual vs. automática por cron, cálculo de preço em duas telas) implementada em
  dois lugares. Corrigir um e esquecer o outro é o erro mais comum desse padrão — sempre que
  achar um bug numa função, perguntar "essa mesma lógica existe em algum outro arquivo?"
  antes de marcar como resolvido. Grep pelo nome da tabela/coluna afetada em todo o repo,
  não só no arquivo onde achou o bug.
- **RLS por linha sem RLS por coluna** [dados] — uma policy de UPDATE que autoriza o dono a
  mexer na própria linha, mas sem trigger separado protegendo colunas sensíveis daquela
  mesma linha (ex: status de aprovação, flags de pagamento). O dono da linha consegue então
  editar campos que só deveriam ser alterados por admin/sistema. Como achar: pra cada tabela
  com policy de UPDATE tipo `auth.uid() = user_id` sem WITH CHECK restringindo colunas,
  listar as colunas e perguntar "alguma dessas decide algo importante sozinha?" — se sim,
  checar se existe trigger `protect_*` cobrindo especificamente essas colunas. **Técnica pra
  achar mais rápido**: quando existir um trigger `protect_*` numa tabela, listar TODAS as
  colunas dela e perguntar, pra cada uma que NÃO está na allowlist do trigger, "essa coluna
  tem uma irmã que JÁ está protegida?" (ex: `rating` protegido mas `rating_count` não,
  `queue_penalty_until` protegido mas `available_after` não) — colunas que compõem o mesmo
  conceito de negócio (uma métrica e seu contador, um valor e seu timestamp de expiração)
  quase sempre precisam da mesma proteção, e é comum uma allowlist cobrir só a mais óbvia das
  duas. Confirmado em: 8LOOP, 2026-09-09 (`rating_count` e `available_after` graváveis direto
  pelo cliente, apesar dos campos irmãos já estarem protegidos há tempo).
- **Trigger de proteção bloqueia silenciosamente o sync legítimo de outra tabela** [dados,
  severidade: dado — corrompe informação real, sem precisar de atacante] — um trigger
  `protect_*` (BEFORE UPDATE, allowlist de colunas por `jwt_role`) e um trigger de sync
  legítimo `SECURITY DEFINER` (ex: `AFTER INSERT` numa tabela filha, recalculando uma média/
  contador na tabela pai) parecem independentes, mas não são: `SECURITY DEFINER` só eleva
  permissão de GRANT — **não muda o GUC `request.jwt.claims` da sessão**. Se o trigger de
  sync faz um `UPDATE` numa coluna que o `protect_*` só libera pra `service_role`/admin, e
  quem disparou a cadeia (ex: quem inseriu a avaliação) é um usuário comum, o `UPDATE` do
  sync é revertido pro valor antigo **toda vez**, sem erro, sem exceção — a feature parece
  funcionar (nenhum log, nenhuma falha visível) mas o dado nunca atualiza de verdade. Como
  achar: pra cada trigger `SECURITY DEFINER` que escreve em OUTRA tabela, checar se essa
  tabela de destino tem um `protect_*`/`guard_*` na mesma coluna — se sim, testar com
  `BEGIN; SET LOCAL role = authenticated; SET LOCAL request.jwt.claims = '...'; <ação que
  dispara o trigger de sync>; SELECT <coluna>; ROLLBACK;` e comparar o valor esperado (o que
  o sync deveria ter calculado) contra o valor real — se divergir, é isso. Correção: nunca
  remover a proteção; liberar via GUC de sessão que o PRÓPRIO trigger de sync liga só pra sua
  chamada (`PERFORM set_config('app.<nome>_rpc', 'on', true);` antes do UPDATE, e o
  `protect_*` checa `current_setting('app.<nome>_rpc', true) IS DISTINCT FROM 'on'` antes de
  reverter) — mesmo padrão de "RPC flag" já usado no projeto pra liberações pontuais
  equivalentes. Depois de corrigir, rodar backfill (`UPDATE` com o GUC de service_role
  simulado) nas linhas que já ficaram com dado congelado/errado antes da correção — a
  proteção nova não corrige retroativamente o que já gravou torto. [camada:
  full-audit-dados] · confirmado em: 8LOOP, 2026-09-09 (`sync_user_rating()` sendo revertido
  por `protect_user_sensitive_fields()` — sistema de reputação inteiro congelado, um OA com
  1 avaliação de 1 estrela mostrando nota 5.0 pros usuários).
- **RETURNING exige SELECT policy** [dados] — em Postgres, `INSERT/UPDATE ... RETURNING`
  falha com "new row violates row-level security policy" se não existir uma policy de SELECT
  que também autorize ver aquela linha — mesmo que o INSERT/UPDATE em si estivesse correto.
  Ao testar um exploit e receber esse erro, não assumir que a escrita foi bloqueada — pode
  ser só essa pegadinha. Confirmar rodando de novo sem RETURNING, ou lendo o resultado numa
  query separada fora da RLS (RESET role, ou row_security=off como superusuário).
- **Deploy de função reseta verify_jwt** [negócio] — `supabase functions deploy <nome>` sem
  flag explícita volta `verify_jwt` pro padrão (`true`). Funções que usam autenticação
  própria (CRON_SECRET, webhook signature) precisam de `--no-verify-jwt` — conferir o valor
  ATUAL (`supabase functions list -o json`) antes de todo deploy, nunca assumir que vai
  manter o que já estava.
- **Correção em um caminho de dinheiro sem replicar no caminho gêmeo** [negócio] — ao
  corrigir um bug financeiro (ex: "paga mesmo se a cobrança falhar"), procurar imediatamente
  por outras funções que fazem a mesma sequência captura→repasse (cron de auto-aprovação,
  endpoint de admin, webhook) — é o mesmo padrão de "lógica duplicada" acima, mas específico
  o bastante pra merecer entrada própria: bugs financeiros em código duplicado são os que
  mais importa não deixar passar. **Variante específica de app multi-gateway**: quando o app
  migra de gateway principal (ex: Stripe → Mercado Pago) mas mantém o antigo como fallback,
  um recurso construído só pro gateway original (aqui: handler de chargeback/contestação no
  webhook) pode nunca ter sido replicado pro novo gateway principal — não é "esqueceram de
  atualizar os dois", é "só existiu pra um desde o início", ainda mais fácil de passar batido
  porque o código do gateway antigo continua correto e revisável, só o do novo é que nunca
  existiu. Como achar: pra cada tipo de evento que o webhook do gateway ANTIGO trata (não só
  pagamento aprovado — disputa, reembolso, estorno), confirmar que o webhook do gateway NOVO
  trata o mesmo conjunto de eventos, não só o "caminho feliz" de pagamento. Confirmado em:
  8LOOP, 2026-08-25 (`full-audit-negocio`) — `stripe-webhook` tratava `charge.dispute.created`
  desde sempre; `mp-webhook` (virou gateway padrão numa migração anterior) só tratava
  `type==="payment"`, qualquer outro tipo (incluindo chargeback) caía num early-return "ok"
  em silêncio — zero proteção contra contestação pós-repasse no caminho que já era o
  principal.
- **Coluna lida sem estar no SELECT — condição sempre falsa por engano de shape, não de
  lógica** [frontend] — em código Supabase-js (ou qualquer client que só devolve as colunas
  pedidas), uma condição tipo `if (row.campo === "x")` é código morto se `campo` não estiver
  na string do `.select(...)` — o objeto retornado nunca tem essa chave, então a comparação é
  sempre `false`, silenciosamente, sem erro nenhum em lugar nenhum. É fácil de escrever esse
  bug ao adicionar um branch novo (ex: um método de pagamento novo) que depende de um campo
  que o `.select()` original nunca precisou antes. Como achar: pra cada condição que
  referencia `row.<campo>`, grep pro `.select(` mais próximo acima na mesma função e conferir
  se `<campo>` está literalmente na lista — não vale "deveria estar", tem que estar escrito.
  Confirmar é determinístico (não precisa de transação/ROLLBACK): o comportamento do client
  garante que o campo ausente do select é sempre `undefined`, então a prova é a leitura do
  código, não uma simulação ao vivo. Achado real: `cancel-task` tinha exatamente esse bug em
  produção (campo `payment_method` usado numa condição, nunca selecionado) — corrigido e
  confirmado por download do código realmente deployado antes e depois do fix.
- **Erro de validação sem `data-has-error` — página não rola até o erro** [fuzzing] — ver
  detalhe completo em `/full-audit-fuzzing`.
- **Cópia de função com hash divergente** [negócio: baixo/médio, varia] — mesma função (ex:
  resolução de CORS, validação repetida) colada em vários arquivos por falta de módulo
  compartilhado; uma das cópias evolui separado das outras sem ninguém perceber, porque
  corrigir "a função" nunca corrige todas as cópias de uma vez. Como achar: grep pra achar
  toda função que se repete em 3+ arquivos, comparar hash de cada bloco — cópia com hash
  diferente da maioria é achado confirmado, sem precisar de prova ao vivo (determinístico,
  igual "coluna lida sem estar no SELECT" acima). Detalhe completo e critério de severidade em
  `/full-audit-duplicacao`. Achado real: `corsOrigin()` do 8LOOP copiada em 20 edge functions,
  19 idênticas, `cancel-task` desviada (não libera domínio de staging) — confirmado em: 8LOOP,
  2026-08-18.
- **Antifraude desenhado só pro lado que executa a transação** [negócio, moderado] — em app com
  2+ papéis interagindo (comprador/vendedor, prestador/cliente), o sistema antifraude tende a
  nascer protegendo só o lado que **executa** o trabalho (gera disputa/avaliação, então é mais
  fácil de instrumentar primeiro), sem equivalente pro lado que **inicia/paga** (cancelamento
  repetido testando limite, múltiplas contas com mesmo dado real, chargeback abusivo). Como
  achar: pra cada papel do app, listar os sinais de abuso já vigiados e perguntar "isso existe
  pro papel oposto também?". Detalhe completo em `/full-audit-security`. Confirmado em: 8LOOP,
  2026-08-19 (`nivel_alerta`/`disputas_30_dias` cobre só o OA, nada equivalente pro QA).
- **Rate-limit concentrado nos endpoints óbvios** [negócio/dado, moderado] — proteção contra
  padrão repetitivo nasce nos 3-5 endpoints mais discutidos no design (login, pagamento) e não
  é revisitada quando endpoints novos de escrita sensível entram depois. Como achar: inventariar
  todos os endpoints que escrevem dinheiro/dado sensível/estado importante e comparar contra os
  que já têm alguma trava — a lista quase sempre é maior que a protegida. Detalhe completo em
  `/full-audit-security`. Confirmado em: 8LOOP, 2026-08-19 (5 de ~30 edge functions).
- **Função/tabela viva em produção sem nunca ter sido commitada** [governança, moderado a
  alto — depende do que a função toca] — código implantado direto na plataforma, sem passar
  pelo repo, é invisível pra qualquer auditoria baseada em leitura de código (todas as demais
  camadas deste skill partem dessa suposição). Como achar: seção 0.8 acima (comparação lista
  viva vs. repo). Não é automaticamente um exploit — checar RLS/auth do que foi achado antes
  de classificar severidade, mas é sempre no mínimo um achado de governança (2+ meses sem
  trilha de revisão de código). Confirmado em: 8LOOP, 2026-08-19 — `send-engagement` (push de
  reengajamento 2x/dia) rodando desde 2026-06-12, RLS das tabelas envolvidas conferida e
  correta ao vivo, mas zero histórico em `git log --all` de qualquer branch. **Escala real
  maior do que uma função isolada, confirmado em: 8LOOP, 2026-08-25 (`full-audit-infra`)** —
  script comparando as 80 funções vivas do schema contra `grep -E "CREATE (OR REPLACE )?
  FUNCTION"` em todas as migrations achou **19 órfãs de uma vez**, incluindo o próprio trigger
  de criação de conta no cadastro (`handle_new_user`/`on_auth_user_created` em `auth.users`,
  nunca teve migration) e `current_user_is_admin()` (usada em 20+ policies de RLS). Lição:
  não presumir que só 1-2 funções escapam — rodar a comparação completa (script, não grep
  manual função por função) fecha o achado de uma vez em vez de descobrir aos poucos ao longo
  de sessões diferentes. Correção é sempre segura (captura pura via `pg_get_functiondef()`,
  `CREATE OR REPLACE` com o texto exato já rodando, zero mudança de comportamento) — mesmo
  padrão das migrations `recover_*_schema.sql` já usadas nesse projeto. **Reauditado e
  confirmado ZERO órfã em: 8LOOP, 2026-09-02** (pentest white-box) — mesma comparação
  determinística (82 funções vivas × 90 nomes únicos já versionados, 55 tabelas vivas × 58
  versionadas, toda diferença explicada por `DROP` de algo que só existiu numa migration
  antiga, nunca o contrário) reconfirma que as 19 órfãs de 2026-08-25 continuam corrigidas e
  nada novo vazou pra produção sem migration desde então — inclusive a tabela criada na própria
  sessão de hoje (`security_certificates`) já nasceu presente nos dois lados. Gatilho de
  reauditoria (seção 5.2) cumprido para esta área nesta data; próxima checagem sugerida por
  tempo (~6 meses, ~2027-03) ou antes disso se algum deploy tocar função/tabela fora do fluxo
  normal de migration.
- **Bucket de Storage público com policy sem dono, corrigido pela metade** [dados/LGPD, alto
  se o conteúdo for sensível] — quando um bucket público misturava dois tipos de conteúdo
  (um intencionalmente público, outro sensível) e alguém já corrigiu UM fluxo (migrou pra
  bucket privado + policy com dono via SECURITY DEFINER), o hábito é assumir que "o bucket"
  foi resolvido — mas grep pelo nome do bucket antigo em todo o repo quase sempre acha um
  segundo (ou terceiro) fluxo de upload que ainda escreve nele, esquecido porque não fez
  parte do incidente que motivou o primeiro fix. Sintoma: `storage.buckets.public = true` +
  policy de SELECT em `storage.objects` que só checa `bucket_id = 'x'`, sem `owner`/path
  scoping — nesse caso a policy nem importa, porque bucket público serve o objeto direto via
  `/storage/v1/object/public/...` sem consultar RLS nenhuma. Como achar: `SELECT id, public
  FROM storage.buckets`; pra cada bucket `public=true`, `SELECT DISTINCT split_part(name,
  '/',1) FROM storage.objects WHERE bucket_id=X` pra ver os "tipos" de conteúdo que vivem lá
  — se mais de um tipo aparece, perguntar se todos são realmente ok pra internet inteira ver
  pra sempre sem login. Confirmar rodando `grep -rn "'<bucket>'" src supabase/functions` —
  cada import é um fluxo separado que precisa da mesma decisão. Correção usa o mesmo padrão
  já estabelecido (bucket privado novo, função `can_read_*`/`can_write_*` SECURITY DEFINER,
  URL assinada via edge function só se a regra de visibilidade for mais ampla que dono/par —
  ex: "qualquer morador do mesmo condomínio enquanto a tarefa está aberta" não cabe numa RLS
  simples, fica melhor decidida em código de servidor). Achado real: 8LOOP tinha corrigido
  `task-photos` (bucket público) só pro fluxo de foto de conclusão de tarefa (migrado pra
  `task-proofs`, privado) um dia antes — o fluxo de foto de PEDIDO (upload na criação da
  tarefa) continuou no bucket público, 124 arquivos reais expostos sem login, o mais recente
  de 1 dia antes da auditoria achar. Confirmado em: 8LOOP, 2026-08-25 (`full-audit-dados`).
- **Deploy de frontend feito de dentro de um git worktree sem copiar o `.env` primeiro**
  [infra, alto — derruba o app inteiro] — `.env` é gitignored de propósito (tem chave), então
  um worktree novo nunca vem com ele. O build de frameworks tipo Vite não FALHA sem a env var
  — só embute `undefined` no bundle final, "build com sucesso" normal — e o app quebra 100%
  em silêncio só na hora de rodar no navegador do usuário real (ex: cliente que instancia com
  a URL/chave ausente lança exceção síncrona no carregamento do módulo, antes de qualquer
  render). Isso é invisível pra `curl`/checagem de HTTP status (HTML e até o bundle JS isolado
  voltam 200 normal) — só aparece abrindo de verdade num navegador e lendo o console. Como
  achar: antes de qualquer `build`/`deploy` rodado de um worktree, confirmar que as env vars
  client-side críticas (as que instanciam algo no topo de um módulo carregado cedo — cliente
  de banco, SDK de auth) estão presentes E que o valor aparece de verdade no arquivo final
  gerado (`grep` pelo valor esperado no bundle), não só que a variável existia no ambiente do
  build. Prevenção estrutural (não só checklist manual): um script de verificação em 2 fases
  (antes do build: falha rápido se a var crítica estiver vazia; depois do build: confirma que
  o valor está de verdade no bundle) ligado direto no comando de deploy via `&&`, pra não
  depender de alguém lembrar de rodar manualmente. Confirmado em: 8LOOP, 2026-08-25
  (`full-audit-infra`) — app ficou em branco em produção por ~45min até o dono reportar.

**Técnica: trava anti-regressão via teste-travado (snapshot), pra regra que já perdeu-e-
recuperou 2+ vezes.** Diferente do resto do catálogo (que é "ache e corrija"), isso é
prevenção estrutural pra um achado que já se repetiu — quando uma policy/regra de negócio
crítica já foi reescrita por engano mais de uma vez em sessões diferentes sem ninguém
perceber que uma exceção/condição sumiu, achar e corrigir de novo não impede a 3ª vez. A
técnica: um teste automatizado que compara a definição ATUAL exata (ex: `with_check` de uma
policy, via `pg_get_functiondef`/`pg_policies`) contra um valor travado no próprio teste —
qualquer divergência (deliberada ou regressão) quebra o teste na hora, forçando quem mudou a
atualizar o valor travado conscientemente em vez de a mudança passar despercebida. Só vale a
pena pra regra que já demonstrou esse padrão de "sumir sem ninguém notar" — não é prática
padrão pra toda policy do app. Confirmado em: 8LOOP, 2026-08-25 — `tasks_insert_own` tinha
perdido a mesma exceção de categoria 3 vezes (05/07, 20/07, 13/08) antes do teste-travado ser
criado.

- **`SET LOCAL request.jwt.claims` sobrescreve o objeto inteiro, apaga o "sub"** [dados] —
  trocar `request.jwt.claims` pra simular `service_role` dentro de uma função (bypass de
  trigger `protect_*`/`guard_*`) substitui o JSON inteiro — inclusive o campo `"sub"` que
  `auth.uid()` lê. Qualquer `WHERE id = auth.uid()` chamado DEPOIS dessa troca, na mesma
  função, vira `WHERE id = NULL`: não dá erro nenhum, só zero linhas afetadas — a função
  continua retornando sucesso. Como achar: grep por `SET LOCAL request.jwt.claims` seguido de
  `auth.uid()` usado depois na mesma function body — o certo é capturar `auth.uid()` numa
  variável ANTES da troca e usar a variável dali em diante, nunca chamar `auth.uid()` de novo.
  Como confirmar rápido: BEGIN/ROLLBACK combinando SELECT de escrita + SELECT de leitura no
  mesmo texto de query pode mascarar o bug (snapshot/visibilidade entre statements de um MCP
  tool nem sempre é confiável) — testar com COMMIT real (ou 2 chamadas separadas, cada uma sua
  própria transação, como PostgREST realmente invoca) e uma leitura verdadeiramente
  independente depois é o que expõe o problema sem ambiguidade. Confirmado em: 8LOOP,
  `upgrade_to_oa_pro`, 2026-08-31.
- **RPC sempre "sucesso" que nunca persiste — bypass de proteção de coluna esquecido**
  [negócio] — uma função `SECURITY DEFINER` faz `UPDATE` numa coluna protegida por trigger
  `protect_*`/`guard_*`, mas nunca aciona o bypass que esse trigger reconhece (ex: `SET LOCAL
  request.jwt.claims` pra `service_role`) — o `UPDATE` roda sem erro, a função retorna
  `{"ok":true}`, e o trigger reverte o valor em silêncio. De fora, a feature "funciona" (sem
  exceção, sem log de erro) mas nunca fez efeito nenhum — pode ficar assim desde a criação sem
  ninguém perceber. Como achar: pra cada função `SECURITY DEFINER` que faz `UPDATE`, listar as
  colunas que ela tenta mudar e comparar contra a lista protegida de CADA trigger
  `protect_*`/`guard_*` daquela tabela (pode ter mais de um, ver entrada seguinte) — se
  sobrepõe, checar se a função tem o bypass. Confirmar chamando a função de verdade (commit
  real) e lendo o valor PERSISTIDO depois, nunca só o retorno JSON da própria função — o
  retorno mentiu desde o início. Confirmado em: 8LOOP, `upgrade_to_oa_pro`, 2026-08-31 (função
  quebrada desde que foi criada — 0 usuários tinham o valor que ela deveria setar).
- **Dois triggers de proteção de coluna com listas diferentes na mesma tabela** [dados] — mais
  de um trigger `BEFORE UPDATE` protegendo colunas sensíveis contra auto-edição na mesma
  tabela, cada um com sua própria lista de campos (ex: um cobre `pix_key`/`stripe_*`, outro
  cobre `xp`/`streak`/`rating` — nomes parecidos, `protect_*` e `guard_*`). Fácil auditar só
  um, concluir "está tudo protegido" e nunca perceber que o outro existe — ou, o oposto, achar
  um campo "desprotegido" no trigger errado quando na verdade está protegido pelo outro. Como
  achar: `SELECT tgname, pg_get_triggerdef(oid) FROM pg_trigger WHERE tgrelid =
  '<tabela>'::regclass AND NOT tgisinternal` — se mais de um trigger `BEFORE UPDATE` aparecer,
  ler as duas listas de campo lado a lado antes de declarar qualquer coluna "protegida" ou
  "desprotegida". Confirmado em: 8LOOP, `users` (`protect_user_sensitive_fields` +
  `guard_user_sensitive_fields`), 2026-08-31.
- **Validação de dígito verificador prova formato, não posse** [negócio] — validar CPF/CNPJ/
  outro documento só pelo algoritmo de checksum (dígito verificador) prova que o NÚMERO é
  matematicamente possível — não prova que existe de verdade, nem que pertence a quem está
  enviando. O algoritmo é público; qualquer um gera quantos números "válidos" quiser sem
  possuir nada real. Como achar: grep por validação de documento que só calcula dígito
  verificador, sem chamada a registro externo (Receita Federal, etc.), E cujo resultado libera
  algo de valor (tier, desconto, verificação, limite maior). Mitigação mínima sem integração
  externa: `UNIQUE constraint` pra impedir reusar o MESMO número (real ou inventado) em várias
  contas — não resolve a posse, mas fecha o abuso "gerar quantas contas quiser com o mesmo
  documento fake". Confirmado em: 8LOOP, `upgrade_to_oa_pro` (CNPJ→tier 'pro'), 2026-08-31.
- **Ciclo de recursão entre policies RLS de tabelas diferentes** [dados] — policy A (tabela X)
  faz `EXISTS`/subquery direto numa tabela Y que também tem RLS, e uma policy de Y consulta de
  volta X (direto ou via função sem `SECURITY DEFINER`) — cria recursão infinita
  ("infinite recursion detected in policy"), que derruba a leitura de AMBAS as tabelas pra
  TODO MUNDO, não só pra quem a policy tentava restringir. `EXISTS` numa tabela RLS sozinho é
  normal e seguro — só o CICLO (ida e volta) é perigoso. Como achar/confirmar (determinístico,
  sem simulação): extrair toda policy com `EXISTS`/`FROM` referenciando outra tabela RLS
  (`pg_policy` + `pg_get_expr(polqual/polwithcheck)` + regex por nome de tabela), montar um
  grafo simples com os pares (tabela_origem, tabela_referenciada), e buscar por qualquer par
  onde A→B e B→A ao mesmo tempo — 1 query SQL faz a checagem inteira, boa candidata a rodar
  como teste automatizado recorrente, não só auditoria pontual. Função `SECURITY DEFINER` na
  cadeia quebra o ciclo (ela roda com privilégio próprio, não reaplica a RLS da tabela que
  consulta) — é o motivo do padrão estabelecido "usar SECURITY DEFINER sempre" pra esse tipo de
  helper funcionar de verdade. Confirmado em: 8LOOP, `users`↔`tasks`
  (`users_select_task_partners`↔`tasks_select`), 2026-07-20 — outage real, derrubou leitura de
  `users` pra todo mundo; reauditado e confirmado sem ciclo ativo em 2026-08-31 (3 candidatos
  com `EXISTS` cruzado revisados, nenhum forma ciclo, os 2 que precisavam já usam
  `SECURITY DEFINER` corretamente).

- **View `SECURITY DEFINER` auto-atualizável com GRANT amplo demais** [dados, alto] — uma
  view criada como atalho de leitura (ex: "mostrar só 2-3 campos não sensíveis pro outro lado
  de uma transação") definida como `SECURITY DEFINER` (ignora RLS da tabela de baixo) e, sem
  ninguém perceber, o Postgres a trata como view **auto-atualizável** (FROM de tabela única,
  sem agregação/JOIN/DISTINCT) — o `GRANT` default de uma view nova no Supabase inclui
  INSERT/UPDATE/DELETE, não só SELECT. Resultado: qualquer usuário (`anon`/`authenticated`)
  com privilégio na view ganha escrita real na tabela de baixo, contornando toda RLS de
  escrita da tabela original — inclusive `DELETE`, mesmo quando a tabela não tem policy
  nenhuma de DELETE pra ninguém (nem pro dono da própria linha). Como achar: rodar o advisor
  nativo do Supabase (`security_definer_view`, nível ERROR) e, pra cada view marcada, checar
  `information_schema.views.is_updatable`/`is_insertable_into` (`YES` é o sinal de alarme) e
  os `GRANT`s reais em `information_schema.role_table_grants` pra `anon`/`authenticated` — se
  tiver qualquer coisa além de `SELECT`, é achado confirmado sem precisar de prova ao vivo
  (view de tabela única sem `WITH CHECK OPTION` é suficiente pra saber que `INSERT` ignora a
  condição do WHERE, e `UPDATE`/`DELETE` só usam o WHERE pra decidir a LINHA afetada, não pra
  limitar os valores novos). Correção segura, sem quebrar a leitura nem tocar na RLS da tabela
  de baixo: `REVOKE INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER ON <view> FROM anon,
  authenticated;` (mantém só SELECT). Confirmado em: 8LOOP, `task_partner_preview` (view criada
  pra mostrar torre/andar do parceiro de tarefa ativa — mesmo propósito que a policy
  `users_select_task_partners` já cobria de forma segura, só que essa view virou uma porta de
  escrita sem ninguém pretender isso), 2026-09-02 (pentest solicitado pelo dono, fora do fluxo
  normal do `/full-audit`, achado via advisor do Supabase).
- **Comentário no código descreve proteção que não existe de verdade** [dinheiro/dados, alto]
  — uma checagem de servidor (trigger, RLS) é pulada de propósito pra um caso específico, com
  comentário explicando que a proteção "já existe em outro lugar" (RLS, outro trigger, outra
  camada) — mas ninguém confirma isso lendo o outro lugar de verdade, só confia no comentário.
  Passa despercebido justamente porque o código *parece* documentado e intencional, não
  esquecido — reler o comentário dá a sensação de "já foi pensado". Como achar: toda vez que
  um `RETURN`/early-exit tiver comentário do tipo "já garantido por X", ler X de verdade
  (`pg_policies`, outro trigger, o schema) e confirmar que a condição citada realmente cobre o
  caso — não aceitar a alegação do comentário como prova. Confirmar com prova ao vivo
  (transação + ROLLBACK): tentar exatamente o caso que o comentário diz estar protegido.
  Confirmado em: 8LOOP, `validate_fixed_price_task`, 2026-09-04 (`full-audit-frontend`) —
  comentário dizia que o piso de R$200/R$100 das categorias de profissional "já é garantido
  pela RLS", mas a policy real (`tasks_insert_own`) só checava o intervalo geral R$5-R$1.000;
  `INSERT` com `service_type='prof_certificado'` e `value=10` foi aceito sem erro até o fix.
- **Teto/piso de preço com `ELSE NULL` — categoria fora da lista fica sem checagem nenhuma**
  [dinheiro, alto] — validação de preço por categoria feita com `CASE WHEN categoria = 'X'
  THEN teto ... ELSE NULL END` é fail-open: qualquer categoria que não bate exatamente com a
  lista (nome novo ainda não cadastrado ali, variação de texto, chamada direta à API pulando
  a UI) cai no `ELSE NULL`, e a checagem seguinte (`IF teto IS NOT NULL AND valor > teto`)
  simplesmente não dispara — sobra só o limite geral da tabela, se existir algum. Como achar:
  em toda função de validação de preço por categoria/tipo, procurar o `ELSE` do CASE — se for
  `NULL` (ou ausente, o que em SQL também vira NULL), é fail-open por padrão. Trocar por um
  valor conservador (o teto mais comum entre as categorias "simples" do app) fecha o buraco
  sem quebrar as categorias já listadas. Confirmar com prova ao vivo: inserir uma categoria
  inventada, não usada em nenhuma tela real, com valor acima do que qualquer categoria real
  deveria aceitar. Confirmado em: 8LOOP, `validate_fixed_price_task`, 2026-09-04
  (`full-audit-frontend`) — `category='Retirar Portaria Extra'` (variação de 'Retirar
  Portaria', teto real R$30) com `value=999` foi aceito sem erro até o fix (`ELSE NULL` →
  `ELSE 30`).

## 4.5 Cobertura sistemática — OWASP Top 10:2025

O catálogo (seção 4) cresce do que **já apareceu** neste app — não garante que toda
categoria de risco conhecida foi checada de propósito. Esta tabela existe pra fechar essa
lacuna: cruza o [OWASP Top 10:2025](https://owasp.org/Top10/2025/) (lista estável mais
recente, publicada nov/2025–jan/2026, sem revisão prevista antes de ~2028) com as camadas
deste orquestrador. **Atualizar a cada rodada de auditoria** — mudar o status quando uma
categoria for checada de propósito, nunca deixar “nunca testado” só porque ninguém perguntou.

Status possíveis: ✅ testado, sem achado · 🔧 testado, achado corrigido · ➖ não se aplica
nesta stack/produto · ⬜ nunca testado de propósito (ainda).

| # | Categoria (OWASP 2025) | Camada responsável | Status (8LOOP, atualizado 2026-09-09) |
|---|---|---|---|
| A01 | Broken Access Control (inclui SSRF) | `full-audit-dados` | 🔧 — RLS/trigger de coluna, achados reais (ver catálogo) |
| A02 | Security Misconfiguration | `full-audit-infra` | 🔧 — grants de TRUNCATE, bucket público (`AUDITORIA_SEGURANCA_2026-08-24`) |
| A03 | Software Supply Chain Failures | `full-audit-infra` (0.8) | ⬜ — só dependência (`npm audit`) checada; pipeline CI/CD e integridade de build **não** foram auditados de propósito |
| A04 | Cryptographic Failures | `full-audit-dados` | ⬜ — nunca confirmado se PIX/CPF têm proteção além de RLS (RLS não é criptografia) |
| A05 | Injection | `full-audit-negocio` + `full-audit-frontend` | ✅ — sem concatenação de SQL (client parametrizado), único `dangerouslySetInnerHTML` é conteúdo estático |
| A06 | Insecure Design | `full-audit-negocio` + `full-audit-juridico` | 🔧 — vários achados de lógica de negócio (fail-open de preço, guarda que nunca dispara, achado A1/A2 de 2026-09-09) |
| A07 | Authentication Failures | `full-audit-dados` | ⬜ — OTP existe, mas rate-limit/força-bruta de código nunca teve rodada dedicada |
| A08 | Software/Data Integrity Failures | `full-audit-gateway` | 🔧 — webhook HMAC fail-closed confirmado; integridade de update/deploy não testada de propósito |
| A09 | Security Logging and Alerting Failures | `full-audit-negocio` | 🔧 — `audit_logs`/Sentry existem e têm achado real (falha de reconciliação de pagamento), mas sem rodada dedicada a cobertura de log |
| A10 | Mishandling of Exceptional Conditions | `full-audit-negocio` (0.8) | 🔧 — 4 edge functions sem try/catch já achadas/corrigidas; não é varredura exaustiva de toda função |

**Leitura honesta desta tabela:** metade das categorias está `⬜`/parcial — isso não é falha
de processo, é o estado real hoje. O valor da tabela não é "provar que já cobrimos tudo", é
**tornar visível, de forma específica, o que ainda não foi olhado de propósito** — pra virar
prioridade concreta de próxima rodada em vez de suposição vaga de "acho que está tudo bem".

## 5. Fechar a sessão

Ao final (ou em qualquer pausa longa):

- Remover do ambiente qualquer segredo carregado só para a auditoria.
- Resumir o que foi feito separando claramente: corrigido e provado / corrigido mas não
  testado ao vivo pelo usuário ainda / risco aceito conscientemente (com motivo) / fora de
  escopo por decisão explícita.
- Registrar em memória (se o projeto usa) o estado de fechamento — quais itens da lista de
  escopo (seção 0) foram cobertos, para uma sessão futura não precisar reconstruir isso lendo
  tudo de novo.

### 5.1 Traduzir pra quem não lê código

O relatório final é pro dono do produto, não pra outro engenheiro — cada achado precisa
responder 3 perguntas em português simples antes de qualquer detalhe técnico: **o que
acontecia** (cenário real: "um morador conseguia..."), **o que isso custava** (dinheiro,
dado vazado, confiança — não "severidade: alta"), **o que mudou** (uma frase, sem jargão).
Detalhe técnico (arquivo, linha, query) vem depois, como referência, não como abertura.

Errado: "RLS ausente em `oa_receivables` permitia leitura cross-tenant via policy
mal-configurada." Certo: "Qualquer prestador logado conseguia ver quanto os outros
prestadores iam receber — bastava trocar um número na tela. Corrigido: agora cada um só
vê o próprio valor."

Isso vale pra toda camada, não só pra quem já usa linguagem simples por padrão — inclusive
achados de `/full-audit-gateway`/`-api`/`-infra`, que tendem a ser mais técnicos por
natureza (idempotência, rate limit) e por isso precisam do exemplo concreto ainda mais.

Quando houver mais de um achado, a lista final vem **ordenada por impacto**, não pela ordem
em que foram encontrados: dinheiro (perda ou desvio real) primeiro, depois dado de outra
pessoa exposto, depois feature quebrada, depois estético/cosmético. O dono decide por onde
começar a corrigir — mas a ordem da lista já indica onde a atenção importa mais.

### 5.2 Gatilho de reauditoria (uma auditoria fechada não é garantia permanente)

Auditoria completa é uma foto do código naquele momento — não garante nada sobre o que vem
depois. Sem um gatilho explícito de quando revisitar, "auditoria completa" vira sinônimo de
"nunca mais vou olhar aqui", que é o próprio erro que a seção 3 (validação final) já tenta
evitar em outra escala. Ao fechar, registrar:

- **Por mudança:** qualquer deploy futuro que toque arquivo/tabela/função já coberto por uma
  camada concluída merece, no mínimo, reler o catálogo de padrões (seção 4) contra o diff
  antes de mergear — não precisa reabrir a camada inteira, mas o hábito de checar não pode
  se perder só porque a auditoria "já foi feita" uma vez.
- **Por tempo, sem mudança de código conhecida:** áreas de dinheiro ou dado sensível
  (`-dados`, `-negocio`, `-gateway`) merecem um prazo de revisão sugerido (ex: 6 meses) —
  bug de terceiro (dependência desatualizada, provedor de gateway mudando comportamento sem
  avisar) não aparece só quando você mexe no seu próprio código.

Registrar esse gatilho junto do resumo de fechamento (seção 5 acima), como parte do estado
salvo em memória — sem isso, uma sessão futura não tem como saber que essa área "já foi
auditada, mas faz N meses" e pode tanto reauditar sem necessidade quanto deixar passar tempo
demais sem perceber.
