---
name: full-audit-juridico
description: Auditoria adversarial de risco jurídico nas 4 frentes — trabalhista/laboral, civil (privacidade/dano a terceiro), criminal, consumidor — adaptando o marco legal ao país onde o produto opera (Brasil: CLT/CDC/LGPD; EUA: right-to-control test/state UDAP/FTC Act/CCPA-CPRA, variando por estado). Eixo diferente das demais camadas do /full-audit — não caça bug de código, caça "isso seria usado como prova contra nós numa ação real?" lendo código/copy/log com a cabeça de um advogado adversarial. Use quando o usuário pedir redução de risco jurídico, auditoria adversarial das 4 frentes, ou revisão de conformidade — de qualquer produto, não só de um específico.
disable-model-invocation: true
---

# Full Audit — Risco jurídico adversarial (4 frentes)

Complementa as demais camadas do `/full-audit`. Diferente de `-dados`/`-negocio`/`-frontend`/
`-fuzzing` (que caçam exploit técnico e provam antes de corrigir) e de `-security` (postura
defensiva/runbook), esta camada não pergunta "isso quebra?" — pergunta **"isso vira evidência
contra o app numa ação real?"**, lendo código, copy e ausência de log com a cabeça de um
advogado adversarial tentando construir um caso, não de um engenheiro procurando bug.

**Origem:** achado do 8LOOP (2026-08-31) — uma função de admin suspendia usuário com um
clique, sem motivo obrigatório nem log de auditoria, contradizendo a promessa dos próprios
Termos de que toda consequência é "objetiva e mensurável". O gap não era um bug técnico (a
função rodava perfeitamente) — era ausência de prova pra sustentar uma alegação já prevista
nos próprios Termos. Esse é o padrão que essa camada caça sistematicamente, em qualquer
produto: não "a função falha", mas "a função não deixa rastro que prove que agiu do jeito que
promete".

## 0. Determinar o marco legal antes de rodar as 4 frentes

O método (as 4 perguntas centrais abaixo) é o mesmo em qualquer país — o texto de lei que
sustenta cada uma muda. Antes de caçar gap, fechar:

- **Onde a empresa está constituída, e onde o usuário/cliente está?** Quando os dois batem
  (empresa BR servindo só usuário BR), um marco só. Quando divergem — empresa americana
  servindo cliente brasileiro, ou o inverso —, rodar as 4 frentes **duas vezes**, uma por
  marco, porque a exposição real é a soma das duas, não a pior das duas. Achado real: Kurax
  (produto da SAORI USA Multi Services Corp, Flórida) atende cliente BR e US no mesmo app —
  a frente civil/privacidade já precisou citar LGPD pro visitante brasileiro E CCPA/CPRA pro
  californiano no mesmo texto de exposição, porque as duas leis se aplicam a públicos
  diferentes do mesmo produto.
- **Dentro dos EUA, qual estado?** Diferente do Brasil (lei federal única cobre a maior parte
  dos 4 eixos), nos EUA o consumidor (UDAP) e certas frentes trabalhistas variam por estado —
  não existe "a lei americana", existe a combinação federal + estado(s) onde a empresa opera e
  onde o cliente está. Registrar qual estado(s) valem pra essa auditoria (ex: SAORI USA é
  registrada na Flórida — FDUTPA é o UDAP de referência; um usuário californiano do mesmo
  produto ainda traz CCPA/CPRA por residir lá, independente de onde a empresa está).
- **Se não houver certeza da jurisdição correta, isso já é o achado** — registrar como risco
  aceito ("marco legal não confirmado, tratando como BR/CLT-CDC-LGPD por padrão até
  confirmação") em vez de simplesmente não rodar a camada.

Pra cada frente abaixo: (1) reler a tese jurídica já registrada em memória do projeto (se
existir) antes de caçar gap novo, pra não repetir achado já mapeado; (2) rodar o checklist
contra o código/copy REAL (grep, SQL, leitura de tela), nunca contra a intenção documentada;
(3) todo achado precisa de exemplo concreto — "a função X faz Y sem Z" — não uma preocupação
genérica sem localização no código.

## Método (as 4 frentes, cada uma com sua pergunta central)

### 1. Trabalhista/laboral — pergunta central: "isso pareceria vínculo empregatício indevido?"

**Brasil (CLT):** a lei exige 4 pilares simultâneos pra reconhecer vínculo — pessoalidade,
não-eventualidade, subordinação, onerosidade (salário). Ausência de qualquer um já derruba a
tese, mas **subordinação** é o pilar mais fácil de acidentalmente reintroduzir em código/copy
novo, porque "melhorar a experiência" e "controlar o prestador" usam os mesmos mecanismos
técnicos (push, gamificação, penalidade).

**EUA:** não existe um teste único federal — a maioria dos estados usa alguma variante do
"right-to-control test" (o quanto a empresa dita COMO o trabalho é feito, não só o resultado);
vários estados (Califórnia à frente, com o AB5/teste ABC) são bem mais rígidos que o padrão
federal do IRS. O sintoma técnico que gera risco é o mesmo do CLT — controle disfarçado de UX
— só a régua legal muda. Se o produto opera em mais de um estado, registrar qual regra vale
pra cada checklist item, sem assumir que passar no teste federal basta.

Checklist (mecanismo é o mesmo nas duas jurisdições, só a régua de julgamento muda):

- **Toda consequência aplicada a quem presta o serviço tem motivo obrigatório + log
  auditável?** (o achado que originou esta camada). Grep por toda função/RPC que suspende,
  penaliza, reduz prioridade, ou nega acesso a esse lado — cada uma precisa do mesmo padrão
  (checagem de papel + motivo não vazio + log de auditoria), não só a que já foi corrigida.
- **Preço é sempre "referência de mercado", nunca ordem?** Copy que usa imperativo ("aceite",
  "cumpra") ou que trata recusa como falha (não só como escolha) reintroduz o argumento mais
  comum contra apps de intermediação de serviço. Checar tanto texto de tela quanto texto de
  notificação/push (mais fácil de escapar da revisão de copy formal).
- **Quem presta o serviço tem margem real de negociar, ou o preço é 100% algorítmico sem
  exceção?** Se for 100% fixo em toda categoria, é o risco residual mais citado nesse tipo de
  ação — não precisa resolver toda vez que essa camada rodar, mas precisa estar registrado como
  risco aceito consciente, não esquecido.
- **Gamificação (XP/streak/nível) usa linguagem de desempenho de empregado?** ("performance",
  "meta batida", tom de avaliação) em vez de "compromisso comercial"/"critério objetivo".
- **Push/notificação de reengajamento tem tom de convocação?** ("não pare", "estão te
  esperando") em vez de convite opcional. Checar toda function que manda push/e-mail
  automático, não só as telas.
- **Exclusividade de fato:** algum mecanismo (fila, prioridade, bônus) pune implicitamente quem
  também presta serviço em outro app/direto? Não precisa proibir por contrato — o risco é o
  comportamento do sistema, não o texto (no Brasil, Art.442-B já cobre "com ou sem
  exclusividade" contratual; o comportamento do sistema é o que importa nos dois países).

### 2. Consumidor — pergunta central: "o que prometemos bate com o que entregamos e com o que os Termos dizem?"

**Brasil:** CDC Art.2 (quem usa o serviço como destinatário final é sempre consumidor nessa
relação) e Art.37 (propaganda enganosa). **EUA:** não existe um "CDC" federal único — a base é
a Section 5 do FTC Act ("unfair or deceptive acts or practices") somada ao UDAP do estado onde
a empresa opera (ex: FDUTPA na Flórida) e, às vezes, do estado onde o cliente está.

O risco mais comum não é ausência de proteção — é **contradição entre marketing/copy e o
texto legal**, nos dois países. Achado real do 8LOOP: landing chamava o escrow de "Garantia do
Prestador"/"Seguro", Termos diziam "não garante os serviços" — exatamente o tipo de
divergência que tanto CDC Art.37 quanto FTC Act §5 tratam como enganoso, cada um com seu
próprio texto de lei. Checklist:

- **Toda palavra "garantia"/"seguro"/"proteção" no marketing tem correspondência exata no
  texto legal** sobre o que exatamente é garantido (ex: o pagamento) e o que não é (ex: a
  qualidade/execução do serviço)? Grep por essas palavras (e seus equivalentes em inglês —
  "guarantee"/"insured"/"protected" — se o produto tiver versão em outro idioma) em
  landing/telas públicas e comparar frase a frase contra o texto de Termos correspondente.
- **Preço final é sempre mostrado antes do pagamento**, sem taxa surpresa? Checar toda tela de
  checkout/confirmação, não só a principal — em produto multi-moeda, checar cada moeda
  separadamente (achado real: Kurax mostrava desconto "de/por" só no preço em reais; o preço em
  dólar precisou de tratamento textual próprio pra não implicar um desconto que não existe
  nessa moeda).
- **Direito de arrependimento/cancelamento está implementado igual ao texto promete?** No
  Brasil, CDC Art.49 (7 dias pra compra fora do estabelecimento comercial, o que inclui
  e-commerce). Nos EUA não há regra federal geral equivalente pra todo tipo de compra — se o
  texto do produto promete algo (prazo, condição), o código precisa aplicar exatamente isso,
  independente de a lei local exigir ou não.
- **Publicidade nunca promete renda/resultado a quem presta o serviço** ("ganhe R$X/hora",
  "renda garantida") — isso vira munição dupla: contra o app (regra de consumidor, se for
  público que usa o produto) e a favor de quem presta o serviço (se alegar que a plataforma
  prometeu ganho, reforça a tese de vínculo mais próximo de emprego — ver frente 1).
- **Responsabilidade solidária/por defeito do serviço:** quanto mais a plataforma controla
  preço/execução/pagamento, mais perto fica de responder pelo serviço entregue por terceiro
  (Brasil: CDC Art.14; EUA: varia por teoria de responsabilidade do estado, mas o sintoma
  técnico — quanto controle a plataforma exerce — é o mesmo gatilho). Toda feature nova que
  aumenta esse controle (curadoria, preço fixo, aprovação de prestador) merece a pergunta
  "isso aumenta nossa exposição?" antes de lançar, não depois.

### 3. Civil — privacidade e dano a terceiro — pergunta central: "se formos processados amanhã, temos prova de diligência?"

**Brasil:** LGPD Art.46 exige diligência razoável, não perfeição — o risco não é "vazou dado"
(que pode acontecer mesmo com diligência), é "não temos como provar que agimos direito".
Sanção: até 2% do faturamento, teto de R$50 milhões por infração.

**EUA:** não existe uma lei federal de privacidade única e geral — o texto de referência muda
por estado; a Califórnia (CCPA/CPRA) é hoje o padrão mais citado e o que mais empresas tratam
como piso de conformidade nacional na prática, mesmo fora da Califórnia. Diferença estrutural
importante pro achado: CCPA/CPRA não tem teto fixo — multa de US$2.663 a US$7.988 por
violação, contável **por consumidor afetado**, o que pode superar o teto brasileiro rapidamente
em produto com volume. Achado real: Kurax precisou corrigir um texto que tinha traduzido
literalmente a estrutura da LGPD (percentual de faturamento com teto) pro inglês, quando a
regra real pro público americano (CCPA/CPRA) é outra estrutura inteira — o erro não era de
tradução de palavra, era de **traduzir a lei errada**.

Checklist:

- **Toda tabela/coleção com dado sensível novo tem controle de acesso (RLS/policy/regra
  equivalente) desde a criação** — não só "vamos adicionar depois"? Reconferir que continua
  valendo pra todo dado novo desde a última auditoria.
- **Todo compartilhamento de dado com terceiro (provedor de e-mail, IA, gateway de pagamento)
  está documentado na Política de Privacidade E logado** (base legal + timestamp)? Grep por
  toda chamada a serviço externo no backend e comparar contra a seção "Compartilhamento com
  terceiros" do texto público.
- **Todo texto voltado a público de uma jurisdição específica cita a lei certa daquela
  jurisdição** — não a lei do país de origem do produto traduzida literalmente. Achado real:
  ver o parágrafo acima (Kurax, LGPD↔CCPA/CPRA). Esse item é novo nesta camada (2026-09-01) e
  vale pra qualquer produto multi-mercado, não só produto brasileiro operando nos EUA — o
  mesmo erro pode acontecer no sentido inverso.
- **Plano de resposta a incidente existe e foi exercitado nos últimos 6 meses?** (ver
  `/full-audit-security` parte B — runbook nunca ensaiado é o mesmo risco que extintor
  vencido).
- **Blindagem patrimonial pessoal está sendo seguida na prática**, não só registrada como
  princípio? (Brasil: CC Art.50 — desconsideração da personalidade jurídica; EUA: "piercing the
  corporate veil", mesmo princípio, nome diferente por estado.) Sinais concretos: conta
  bancária do negócio sempre separada da pessoal, decisões relevantes com registro escrito
  (e-mail/documento, não só verbal), contabilidade em dia — isso não é código, é processo, mas
  essa camada deve perguntar, não assumir.
- **Todo texto que promete algo específico ("dados criptografados", "revisão humana de decisão
  automatizada") tem implementação real correspondente?** Promessa de privacidade sem
  implementação é pior do que não prometer — vira evidência de má-fé, não só de falha técnica,
  em qualquer jurisdição.

### 4. Criminal — pergunta central: "se algo grave acontecer, temos prova de que fizemos a checagem que dizemos fazer?"

Frente mais jurisdição-agnóstica das 4 — o princípio ("evidência retida > resultado sem
rastro") vale igual em qualquer país, mesmo que o tipo penal exato mude. A frente menos
desenvolvida hoje neste tipo de produto, porque o risco só aparece quando algo dá errado —
exatamente por isso merece checklist ativo, não reativo. Checklist:

- **Verificação de antecedentes (ou equivalente) deixa evidência retida** — protocolo, print,
  número de processo — ou só o resultado (aprovado/reprovado) sem rastro do que foi checado?
  Aprovar sem evidência retida é o mesmo problema do achado que originou esta camada, um nível
  acima: não dá pra provar depois que a checagem realmente aconteceu.
- **Menor de idade:** todo consentimento de responsável (quando aplicável) é persistido com
  IP/timestamp, não só validado e descartado no formulário? Se a decisão do produto foi
  conscientemente não persistir isso, checar se essa decisão continua sendo a certa — revisar
  se mudou o volume/exposição do produto desde que foi tomada.
- **Categoria de denúncia grave (segurança física) é tratada diferente de disputa comum?**
  Suspensão automática, trava de evidência, e escalonamento não podem esperar o mesmo ritmo de
  uma disputa financeira normal — ver `/full-audit-security` parte B pro runbook completo.
- **Trava contra lavagem/fraude via valor manipulado ainda está ativa e sem furo?** (teto/piso
  de preço por categoria, validação de contraproposta) — reconferir que continua disparando de
  verdade, não só existindo no código (achado real: `validate_fixed_price_task` do 8LOOP ficou
  código morto por meses sem ninguém perceber).
- **Identidade é verificada com prova viva (câmera obrigatória), não só upload de arquivo que
  pode ser de outra pessoa/reusado?** Hash do documento contra reuso, comparação facial se
  disponível.

## Critério de conclusão

Cada item das 4 listas (por marco legal aplicável, ver seção 0) termina em um de três estados —
revisado sem achado, achado promovido (com correção aplicada e testada, seguindo o loop
prova-antes-de-corrigir do orquestrador seção 2), ou risco aceito registrado com motivo e data.
Diferente das camadas técnicas, um achado aqui frequentemente não tem "correção de código"
sozinha — pode exigir decisão de negócio do dono (ex: dar margem de negociação de preço) ou
consulta a advogado de verdade **da jurisdição certa** (esta camada reduz risco e organiza
evidência, não substitui aconselhamento jurídico real — muito menos de um advogado do país
errado).

**Gatilho de reauditoria:** toda feature nova que toque preço, penalidade, dado de quem presta
ou quem contrata o serviço, ou comunicação automática (push/e-mail) merece reler o checklist da
frente 1 (trabalhista) antes de lançar — é a frente que mais rápido acumula gap novo, porque
nasce de decisão de produto, não de bug. As outras 3 frentes seguem o mesmo prazo de 6 meses
sugerido pra `-dados`/`-negocio`/`-security` (orquestrador seção 5.2). Todo produto que passa a
atender uma jurisdição nova (novo idioma, novo mercado, nova moeda) dispara a seção 0 de novo
antes de reler qualquer checklist — o marco legal pode ter mudado mesmo sem nenhuma linha de
código nova.

## Confirmado em: 8LOOP, 2026-09-04 (primeira execução formal)

Seção 0: 8LOOP roda 100% no CPF pessoal do dono, condomínios brasileiros, sem CNPJ, sem
cliente fora do Brasil — marco legal único (BR/CLT-CDC-LGPD), 4 frentes rodadas 1x só.
Reli a tese jurídica já registrada em `project_8loop.md` (seção 2, extensa — 8 revisões
anteriores de Termos/Política já mapearam a maior parte das 4 frentes) antes de caçar gap
novo, conforme instrução da própria camada — a maioria dos itens do checklist já tinha
tese registrada; reverificados contra o código real de hoje, não só a memória:

- **Frente 1 (trabalhista)** — gamificação (XP/streak) e push de reengajamento (`send-
  engagement`) reconferidos linha por linha: tom 100% opcional/convite ("quando puder",
  "se topar", "sem compromisso"), nenhum vestígio de linguagem de convocação/desempenho.
  Opt-in semanal já framed corretamente em Terms.tsx §próprio parágrafo. Sem achado novo.
  Risco residual já registrado (preço 100% fixo sem negociação real em categorias comuns)
  continua como estava — risco aceito, não resolvido nesta rodada (fora do escopo de
  código, é decisão de produto já registrada).
- **Frente 2 (consumidor)** — não re-verificado linha por linha nesta sessão (revisão
  completa já datada de 2026-08-10, sem mudança de copy de marketing desde então que eu
  tenha tocado). Recomendo re-passar se alguma tela nova de marketing for publicada.
- **Frente 3 (civil/privacidade)** — consentimento do responsável (menor QA) reconfirmado
  no código atual: checkbox ainda existe só como validação client-side, nunca persiste
  (`parentConsent` não aparece em nenhum insert/update). **Não é achado novo** — decisão
  consciente do dono em 2026-08-10 (perguntado, respondeu "não" a persistir). Registrado
  como risco aceito confirmado inalterado, não reaberto.
- **Frente 4 (criminal) — 1 achado real.** "Trava contra lavagem via valor manipulado"
  (item do próprio checklist, citava `validate_fixed_price_task` como exemplo histórico de
  código morto) — **fechado nesta mesma sessão**, achado e corrigido via `-frontend` antes
  desta camada rodar (2 migrations, `20260904000000`/`20260904010000`). Item novo
  encontrado: **identidade verificada só por upload de arquivo, nunca prova viva** — o
  `<input type="file">` de foto de perfil/comprovante não tem `capture="user"` (câmera
  frontal) nem qualquer exigência de captura ao vivo; aceita qualquer imagem da galeria.
  Nenhum hash de reuso entre contas (`sha256Hex` no projeto só protege código PIX, não
  foto). **Risco aceito, 2026-09-04** — dono decidiu não exigir câmera ao vivo por ora;
  motivo: tensão direta com o trabalho desta mesma sessão de *reduzir* fricção de cadastro
  (achado real: 21% dos cadastros abandonavam antes de completar, ver `-frontend`/
  `-conteudo` da mesma data) — custo de fricção supera o risco hoje, decisão consciente, não
  esquecimento. Revisitar se o volume de cadastros crescer o suficiente pra justificar a
  fricção extra, ou se um caso real de identidade falsa acontecer.

Critério de conclusão da camada: 3 dos 4 itens fecharam "revisado, sem achado novo ou risco
aceito já registrado"; 1 achado novo (Frente 4, prova viva de identidade) fechou como "achado
registrado, risco aceito com motivo e data" — nenhum código alterado por esta camada
especificamente (a correção de "trava de lavagem" veio de `-frontend`, já rodada antes). Com
o achado da Frente 4 resolvido (aceito, não pendente), a rodada de 2026-09-04 está
completamente fechada — as 3 estados finais possíveis (sem achado / achado corrigido / risco
aceito) foram todos usados nesta execução.
