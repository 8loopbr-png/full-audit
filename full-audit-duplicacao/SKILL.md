---
name: full-audit-duplicacao
description: Checklist do /full-audit pra duplicação de código e "gordura" de manutenibilidade — funções/blocos copiados entre arquivos que podem desviar silenciosamente (achado real, 8LOOP: CORS divergente entre edge functions), e volume de repetição (ex: estilo inline) que infla o código sem mudar comportamento. Eixo diferente das demais camadas: o achado pode ser risco de bug latente (cópia já desviada) OU custo puro de manutenção (mais lugares pra errar no futuro), raramente um exploit provável na hora. Use quando o usuário pedir pra "enxugar", "reduzir gordura", achar código duplicado, ou avaliar manutenibilidade.
disable-model-invocation: true
---

# Full Audit — Duplicação e manutenibilidade (checklist)

Complementa as demais camadas do `/full-audit`. **Diferente das camadas de bug/segurança**
(`-dados`, `-negocio`, `-frontend`), aqui o achado nem sempre é "isso quebra agora" — pode ser
"isso já quebrou silenciosamente numa cópia" (grave, trata como bug) ou "isso custa caro pra
manter, mas ainda funciona igual em todo lugar" (custo de engenharia, não bug). Classificar
qual dos dois é o primeiro passo de cada achado, porque muda o rigor exigido pra corrigir.

**Origem:** achado real na auditoria de manutenibilidade do 8LOOP (2026-08-18) — mapeamento de
"gordura" pedido pelo dono achou uma função `corsOrigin()` copiada em 20 edge functions, 19
idênticas e **1 já desviada silenciosamente** (`cancel-task` não libera o domínio de staging,
diferente de todas as outras). Duplicação não é só estética: é onde um bug já corrigido em um
lugar continua vivo nos outros, sem ninguém perceber.

## Como achar (por ordem de gravidade)

1. **Função/bloco inteiro copiado entre arquivos — o mais grave, e o mais barato de provar.**
   Grep por definições de função repetidas em mais de um arquivo (ex: `grep -rn "^function
   nome\|^const nome ="` pra cada nome suspeito, ou `grep -rl "nome_da_funcao"` pra achar em
   quantos arquivos aparece). Pra cada grupo de cópias, extrair o bloco de cada uma e comparar
   hash (`md5sum` ou equivalente) — **qualquer cópia com hash diferente da maioria é uma cópia
   já desviada**, é achado confirmado por leitura de código, não precisa de transação/ROLLBACK
   nem prova ao vivo (mesmo padrão determinístico da entrada "Coluna lida sem estar no SELECT"
   no catálogo geral, `/full-audit` seção 4). Promover direto: qual arquivo desviou, o que a
   diferença muda no comportamento real, desde quando (checar `git log` do arquivo).

2. **Checagem de segurança/negócio duplicada across arquivos (não necessariamente idêntica
   char a char, mas a mesma regra reimplementada).** Isso já tem entrada própria no catálogo
   geral — "Lógica duplicada em gêmeos divergindo" e "Correção em caminho de dinheiro sem
   replicar no caminho gêmeo" (`/full-audit` seção 4). Não duplicar aqui — só lembrar de rodar
   esse grep também quando o pedido for "mapear gordura", porque é o mesmo mecanismo de causa
   (código colado em vez de compartilhado) com consequência mais grave (dinheiro/dado).

3. **Ausência de módulo compartilhado quando múltiplos arquivos resolvem o mesmo problema.**
   Mesmo que todas as cópias estejam **hoje** sincronizadas (sem desvio ainda), a ausência de
   um lugar único (`_shared/`, `lib/`, `utils/`) pra lógica repetida (CORS, formatação de erro,
   validação de campo comum) já é o achado — é o motivo estrutural pelo qual o item 1 acontece
   mais cedo ou mais tarde. Como achar: pra cada função/constante que aparece em 3+ arquivos,
   perguntar "existe uma pasta compartilhada nessa stack, e essa função está fora dela por
   quê?". Registrar como risco (não bug ainda) se as cópias estiverem sincronizadas hoje.

4. **Volume de repetição de forma/estilo (UI) — métrica de manutenibilidade, não bug.**
   Contar ocorrências de bloco de estilo repetido por arquivo (ex: `grep -c 'style={{'` em
   React/inline-style, ou equivalente de CSS-in-JS/classe repetida na stack). Rankear os
   maiores arquivos por linhas totais **e** por densidade de repetição (ocorrências / linhas).
   Sinalizar como candidato a extrair componente/constante quando um arquivo isolado responde
   por uma fatia desproporcional do total do frontend (regra prática: >10% das linhas do
   frontend inteiro num único arquivo, ou média de menos de 10 linhas por bloco de estilo).
   Aqui o achado é sempre "custo de manutenção", nunca "bug" — não precisa do loop de prova da
   seção 2 do orquestrador, precisa é de teste visual antes/depois na hora de corrigir (ver
   regra de ouro abaixo).

## Regra de ouro pra corrigir sem disfarçar mudança de comportamento

Refatoração de duplicação **nunca** muda o resultado observável. Se ao extrair uma função/
componente compartilhado o resultado (visual, pixel a pixel, ou lógico, valor a valor) não sai
idêntico ao "antes", **não é mais um refactor — é mudança de comportamento escondida atrás de
uma limpeza**, e passa a exigir o loop de prova completo (`/full-audit` seção 2), com o mesmo
rigor de qualquer outro achado de bug. Isso vale com mais força ainda pro item 1 (funções já
desviadas): ao unificar as cópias, decidir explicitamente **qual comportamento fica** (o da
maioria, ou o correto de verdade, se forem diferentes) — nunca assumir que a maioria está
certa só por ser maioria.

## Critério de conclusão

Todo grupo de duplicação encontrado está em um de três estados: **cosmético/registrado** (só
custo de manutenção, sem desvio, correção opcional), **corrigido e provado** (cópia unificada,
antes/depois comparado, comportamento idêntico confirmado), ou **desvio real promovido pro
catálogo geral** (`/full-audit` seção 4) com a diferença exata documentada.

## Confirmado em: 8LOOP, 2026-08-18

- **CORS divergente entre edge functions** [negócio, baixo/médio] — função `corsOrigin()`
  copiada em 20 das 31 edge functions do projeto (nenhuma pasta `_shared/` existe no backend).
  19 cópias idênticas byte a byte; `cancel-task` tinha uma versão mais antiga que não libera o
  domínio de preview de staging (`*.8loop-v2.pages.dev`) — cancelamento de tarefa a partir do
  ambiente de staging provavelmente falha por CORS, sem ninguém ter percebido porque o código
  está enterrado numa cópia isolada. Achado por grep + hash-diff, sem necessidade de prova ao
  vivo (determinístico).
- **Concentração de repetição de estilo em um único arquivo** [estético/manutenção] —
  `src/pages/Admin.tsx` tem 4.691 linhas (15% do TypeScript do projeto inteiro) e 804 blocos
  `style={{...}}` inline, sem nenhum componente `<Card>`/`<Badge>` compartilhado pra esse
  padrão visual repetido (existe `src/lib/theme.ts` centralizando cores, mas não o "molde" do
  bloco de estilo em si). Total do projeto: 3.410 blocos de estilo inline espalhados pelas
  páginas principais.
