---
name: full-audit-conteudo
description: Checklist do /full-audit pra risco específico de tipo de conteúdo — chat/mensagem em tempo real, mapa/geolocalização, foto/vídeo, agenda/calendário. Complementa full-audit-dados (RLS/policy genérica, bucket público vs privado) e full-audit-fuzzing (XSS armazenado) com os riscos que só existem por causa da natureza daquele tipo específico de dado. Sem entradas provadas ainda no catálogo geral. Use quando o app tiver qualquer um desses 4 tipos de conteúdo.
disable-model-invocation: true
---

# Full Audit — Conteúdo por tipo (checklist)

Complementa as demais camadas do `/full-audit`. **Não duplica** o que `/full-audit-dados` já
cobre (RLS de tabela/bucket, policy de storage) nem o que `/full-audit-fuzzing` já cobre (XSS
em texto reexibido) — aqui o eixo é o risco que só existe **por causa da natureza do
conteúdo em si**, mesmo com RLS e sanitização de texto perfeitos. Um app pode estar 100%
correto nessas duas camadas e ainda vazar a localização exata de um usuário porque ninguém
pensou em EXIF, ou deixar duas pessoas confirmarem a mesma vaga de agenda ao mesmo tempo.

A maioria dos itens abaixo são pontos de atenção genéricos de mercado, sem entrada provada
ainda — pra a varredura não pular esses ângulos. Só vira entrada do catálogo geral
(`/full-audit` seção 4) depois de confirmado contra um app real, passando pelo loop de
prova-antes-de-corrigir (`/full-audit` seção 2). Nem todo app tem os 4 tipos — checar só os
que existem, registrando os ausentes como "não se aplica" (não pular em silêncio).

## Pontos de atenção

### Chat / mensagens em tempo real

- **Escopo de canal realtime** — quem decide que uma mensagem chega só pra quem participa da
  conversa é o servidor (RLS/policy no canal, ou autorização explícita por subscription), ou
  o filtro é só no cliente? Provar: abrir 2 contas de teste descartáveis, assinar o canal de
  uma conversa que não é sua, ver se a mensagem chega mesmo assim.
- **Rate-limit de envio** — spam/flood de mensagem sem trava. Endpoint de mensagem costuma
  ficar de fora quando rate-limit é aplicado só nos endpoints "óbvios" (login, pagamento) —
  ver entrada correlata no catálogo geral sobre rate-limit concentrado nos endpoints óbvios.
- **Exclusão/edição não é ilusória** — mensagem "apagada" pelo remetente realmente some do
  banco (ou só marca uma flag que a tela esconde, mas segue lá pra quem consulta direto)?
- **Anexo dentro do chat** — foto/arquivo trocado numa conversa herda os riscos da seção
  "Foto/vídeo" abaixo, mais um risco próprio: o link do anexo funciona pra qualquer um que
  tenha a URL, mesmo sem nunca ter feito parte daquela conversa?

### Mapa / geolocalização

- **Precisão além do necessário** — coordenada salva com precisão de metros quando o
  propósito do produto só exige bairro/CEP/torre? Precisão exata de onde alguém mora/está é
  dado sensível desnecessário (LGPD, princípio de minimização) — quanto mais preciso, maior o
  dano se vazar.
- **Spoofing de localização** — a coordenada que o app usa pra decidir algo (ex: "só posso
  fazer isso perto de mim") vem só do cliente (GPS do navegador, forjável via devtools/
  extensão) sem nenhuma checagem do lado do servidor?
- **Chave de API de mapa exposta sem restrição** — chave do Google Maps/Mapbox/equivalente
  no bundle do frontend sem restrição de domínio/referrer e sem teto de cota — qualquer um
  que a extrair do JS pode usar por fora e estourar a fatura do dono.
- **Localização em tempo real vista por quem não devia** — se o app expõe "onde essa pessoa
  está agora", isso é filtrado por RLS/policy pra só quem tem relação ativa com ela, ou
  qualquer usuário autenticado consegue consultar a posição de qualquer outro?

### Foto / vídeo

- **Metadado EXIF não removido** — arquivo servido do jeito que foi recebido, com o GPS
  embutido pela câmera/celular intacto? Isso vaza localização mais precisa do que qualquer
  campo de formulário, mesmo que a tela do app nunca mostre coordenada nenhuma — quem baixa o
  arquivo original ganha a informação de qualquer forma.
- **Validação de tipo por extensão, não por conteúdo** — o backend aceita o arquivo confiando
  no nome (`.jpg`) ou confere de fato o conteúdo (magic bytes/content-type real)? Sem isso,
  dá pra subir um arquivo executável disfarçado de imagem.
- **Upload órfão** — arquivo chega a ser enviado pro storage mas o registro correspondente no
  banco nunca é criado (fluxo abandonado no meio) ou é apagado depois sem apagar o arquivo
  junto — fica pra sempre sem dono, e nenhuma auditoria de RLS pega isso (RLS protege linha
  de tabela, não o objeto do bucket sozinho).
- **Limite de tamanho aplicado só no frontend** — input do formulário limita o tamanho, mas o
  endpoint/edge function aceita qualquer coisa se chamado direto — sem teto real, upload
  gigante trava storage ou gera custo inesperado.

### Agenda / calendário

- **Fuso horário salvo sem contexto** — horário gravado "cru" (sem timezone) em vez de UTC
  convertido na exibição — quebra assim que usuário e servidor (ou dois usuários) estão em
  fusos diferentes.
- **Condição de corrida no agendamento** — dois usuários confirmando o mesmo horário/vaga ao
  mesmo tempo sem constraint/lock no banco — os dois "ganham" o mesmo slot, sem ninguém
  perceber até o conflito acontecer na prática.
- **Vazamento de conteúdo via disponibilidade** — tela de "livre/ocupado" expõe mais do que
  isso (título do compromisso, com quem, onde) pra quem só devia enxergar se a pessoa está
  livre ou ocupada, nada além disso?
- **Evento recorrente editado pela metade** — editar/cancelar 1 ocorrência de uma série
  afeta só aquela ocorrência, ou corrompe/duplica as seguintes por engano?

## Critério de conclusão

Cada tipo de conteúdo presente no app foi checado item a item contra o app real (tipo
ausente = "não se aplica", registrado, nunca pulado em silêncio) — resultado "checado, sem
achado", ou "achado, promovido pro catálogo geral com a prova".

## Confirmado em: 8LOOP, 2026-09-04 (primeira execução formal)

Levantamento dos 4 tipos antes de variar: **aplicável** — Chat (tabela `task_messages` real,
mensagem por tarefa) e Foto/vídeo (upload de foto de tarefa, perfil, verificação, anúncio/
parceiro). **Não se aplica, registrado e não pulado em silêncio** — Mapa/geolocalização (zero
coordenada GPS, zero lib de mapa no bundle) e Agenda/calendário (nenhuma tabela de
agendamento/booking/disponibilidade).

**Chat — checado, sem achado novo.** Escopo de canal realtime é RLS de verdade (`SELECT`
restrita a `sender_id`/`receiver_id`, não só filtro client-side) — Supabase Realtime honra
RLS em `postgres_changes`, confirmado lendo a policy, não só o filtro do client. Rate-limit de
envio já corrigido em auditoria anterior (2026-08-24/25, comentário `T2` no próprio código).
Sem função de apagar mensagem nem anexo de arquivo — os 2 riscos do checklist não se aplicam
por ausência de feature, não por lacuna.

**Foto/vídeo — 1 achado real.** [feature/dados, baixo-médio — upload feito pelo próprio
admin, não por usuário não confiável, mas real] `sanitizePhotoForUpload()` (remove EXIF +
valida magic bytes reais) existe desde 2026-07-21 e está corretamente ligado nos uploads de
foto de tarefa (QAServiceSheet.tsx/Create.tsx) e, por caminho independente (canvas redraw via
`PhotoCropModal`), nos de perfil (Profile.tsx/RegisterCondo.tsx) — mas os 3 uploads de
anúncio/parceiro no Admin.tsx (criar anúncio, editar anúncio, criar parceiro) subiam o `File`
original direto pro bucket **público** `task-photos` (confirmado `public=true` no banco), sem
nenhuma das duas proteções. Corrigido: os 3 pontos passaram a chamar o mesmo helper
compartilhado antes do upload — mesmo padrão de "proteção existe mas não em todo lugar que
precisa dela" já visto na camada `-duplicacao` (função copiada com uma cópia desviada), só que
aqui é helper reaproveitável não reaproveitado, não cópia divergente. Efeito colateral achado
e corrigido junto: nenhum dos 3 handlers tinha try/catch — como o helper pode lançar (arquivo
inválido), isso teria travado o botão em "Criando…"/"Salvando…" pra sempre (mesma classe do
achado já visto em `-frontend`, 2026-09-01). Validado com `tsc --noEmit` + build limpo +
staging conferido em navegador real (zero erro de console). Commit `0c1cab7`.
