# Roadmap — Módulo de Reprodução Equina v2.0

> Origem: spec funcional/técnica do sócio (`spec-reproducao-mimba-v2.docx`, MimbaTech, julho 2026).
> Este documento quebra a spec em fases executáveis, na ordem sugerida pela própria spec (seção 11).
> Cada fase é um refactor/extensão do módulo reprodutivo atual (tabela `coberturas` simples → modelo completo).

## Como usar este documento
- Checkbox por fase e por sub-item. Marcar `[x]` ao concluir.
- Toda tabela nova entra no schema `public` (template) via skill `nova-migration-tenant` — nunca DDL direta em `cab_*`.
- Mudanças em RLS/auth passam pelo `revisor-isolamento` antes de mergear.
- Tenants existentes hoje: `cab_cabanha_pedro_teste`, `cab_cabanha_santa_enoema`, `cab_mae_de_deus`, `cab_qa_isolamento`, `cab_qa_segunda`.

## ⚠️ Armadilha nova descoberta (Fase 1) — anon ganha grant automático em tabela nova
O schema `public` tem `ALTER DEFAULT PRIVILEGES` do Supabase que concede `anon`/`authenticated` automaticamente
em **toda tabela nova** criada por `postgres`/`supabase_admin` — mesmo com RLS habilitada, isso deixa `anon`
com grant (embora RLS sem policy já bloqueie o acesso a dados). Toda tabela nova do template precisa de
`revoke all privileges on public.<tabela> from anon;` logo após o `create table`, senão ela sai do padrão das
tabelas antigas (`animais`, `coberturas`, `usuarios` etc., que não têm grant nenhum para `anon`). Válido para
todas as tabelas novas das Fases 2-8.

## Decisão de migração
A tabela `coberturas` atual (simples: 1 linha por cobertura, sem saldo/ciclo/planejamento) é **substituída** pelo
modelo novo (`fontes_cobertura` + `acasalamentos` + `tentativas` + `gestacoes` + `crias`). Dados existentes em
`coberturas` precisam de migração de linha (fonte "avulsa" retroativa) — tratar isso na Fase 1, não descartar dado.

---

## Fase 1 — Fontes de Cobertura (base) 🔴 em andamento
- [x] Tabela `fontes_cobertura` (tipo, garanhão, saldo, ciclo, vigência, status) — seção 2.3 da spec.
  Criada no template `public` + replicada nos 5 schemas `cab_*` existentes + RPC `provisionar_schema_cabanha`
  atualizada pra tenants novos. Isolamento revisado (`revisor-isolamento` + verificação manual de
  `pg_policies`/grants/RPC): policies corretas por schema, sem grant a `anon` (bug do default privilege do
  Supabase corrigido — ver armadilha abaixo).
- [ ] Enum/regras de saldo: `quantidade_adquirida − confirmadas − negociadas` (calculado on-the-fly ou view — decidir na Fase 2/3 junto com acasalamentos/tentativas, que são quem consome o saldo)
- [x] Migração de dados: linhas existentes em `coberturas` viram fontes "próprio" retroativas (1 unidade cada, ciclo 25/26,
  obs marcando a origem/id original). Rodada em todos os schemas — só `cab_mae_de_deus` (4) e `cab_cabanha_pedro_teste` (2)
  tinham dados; migration idempotente (não duplica se rodar de novo).
- [x] RLS por tenant (`tem_acesso_tenant`), sem diferenciação de perfil ainda (ver achado `rls-permissiva-por-perfil`)
- [x] Tela: lista de cards por garanhão, barra de saldo, filtros, ação "Nova fonte" — implementada como 5ª aba dentro de
  "Gestação" no `index.html` (ver `renderFontesCobertura`/`modal-fonte-cobertura`). Saldo ainda fixo em 100% disponível —
  será alimentado de verdade pelas Fases 2/3 (consumo por confirmação de prenhez).
- [x] RPC `carregar_dados_cabanha` atualizada para devolver `fontes_cobertura` (evita chamada REST extra / preflight CORS,
  mesma lição da Prioridade 1 do roadmap de ajustes menores).
- Dependências: nenhuma. **Base de tudo.**

## Fase 2 — Acasalamentos + Simulação Genética Fase 1 ✅
- [x] Tabela `acasalamentos` (égua × fonte × ciclo, fluxo de status rascunho→simulado→aprovado→em_curso→confirmado/cancelado).
  Criada no template `public` + replicada nos 5 schemas `cab_*` + `provisionar_schema_cabanha` e `carregar_dados_cabanha`
  atualizadas. Isolamento verificado (mesmo padrão da Fase 1, sem grant a `anon`) — RPC de provisionamento agora também
  revoga `anon` de **todas** as tabelas do schema novo, não só uma, corrigindo a raiz do bug achado na Fase 1.
- [x] Painel de simulação: cruzamentos anteriores (via `coberturas2` histórico) e alerta de consanguinidade (comparação de
  `sangues_linhagem.ancestrais_pos`, `gen ≤ 3`, com fallback neutro se SBB ainda não analisado). **Não implementado**:
  concentração de sangue projetada completa (parte mais avançada da Fase 2 original da spec) — cortado deliberadamente,
  fica para revisitar se o sócio pedir.
- [x] Kanban de planejamento (Rascunho/Simulado/Aprovado/Em Curso/Concluído + lista de Cancelados separada).
- [x] Transições de status rascunho→simulado→aprovado implementadas com os botões certos por card; cancelamento sempre
  exige `motivo_cancelamento`. Transição aprovado→em_curso fica para a Fase 3 (é o veterinário que inicia).
- Dependências: Fase 1, análise de sangues existente (`sangues_linhagem`).

## Fase 3 — Tentativas + confirmação de prenhez + saldo ✅
- [x] Tabela `tentativas` (N por acasalamento, sem decremento de saldo até confirmação). Template + 5 schemas `cab_*` +
  `provisionar_schema_cabanha`/`carregar_dados_cabanha` atualizadas. Isolamento verificado (10 colunas em todos os
  schemas, sem grant a `anon`).
- [x] Regra: saldo só decrementa na confirmação; cancelamento sem tentativa devolve saldo, com tentativa frustrada não
  devolve. **Saldo real implementado** (não mais placeholder da Fase 1): `disponivel = quantidade_adquirida − count(
  acasalamentos com status='confirmado' daquela fonte)`, calculado client-side a partir dos arrays já sincronizados —
  sem coluna nova no banco. Devolução automática por design: acasalamento cancelado nunca chega a 'confirmado', então
  nunca é descontado.
- [x] Flag `especialidade_reproducao` em `usuarios` (veterinário reprodutor) — coluna adicionada + exposta em
  `carregar_dados_cabanha`. Seletor de veterinário no registro de tentativa filtra por essa flag, com fallback pra
  todos os usuários se nenhum estiver marcado (evita travar o fluxo em cabanhas que ainda não configuraram o perfil).
- [x] Transição automática aprovado→em_curso na primeira tentativa registrada; em_curso→confirmado ao marcar resultado
  'prenha' de uma tentativa (data_resultado obrigatória). Resultado 'vazia' mantém em_curso, permitindo nova tentativa.
- [x] Seletor de fonte no "+ Novo acasalamento" (Fase 2) agora só mostra fontes com saldo real > 0 e status ativa.
- Dependências: Fase 2, perfil vet reprodutor.

## Fase 4 — Gestações + linha do tempo ✅
- [x] Tabela `gestacoes` (criada automaticamente ao confirmar prenhez — nunca manual, sem UI de criação exposta em
  lugar nenhum). Template + 5 schemas `cab_*` + RPCs atualizadas. Isolamento verificado (13 colunas em todos os
  schemas, sem grant a `anon`).
- [x] Cálculo automático de `parto_previsto` — coluna `GENERATED ALWAYS AS (data_cobertura + 340 dias) STORED` no
  próprio Postgres (não calculado em JS, evita divergência entre telas).
- [x] Status: gestando/parida/abortada/perdida. Auto-criação plugada no mesmo ponto da Fase 3 onde a tentativa com
  resultado 'prenha' confirma o acasalamento (`_confirmarResultadoTentativa`).
- [x] Card visual por trimestre (1º/2º/3º, calculado a partir de `data_cobertura`) + alerta de parto próximo (≤30 dias)
  ou atrasado + timeline expandida reaproveitando as tentativas da Fase 3. Ações: registrar parto/aborto/perda.
- [x] Nova aba "Gestações" coexistindo com a aba antiga "Gestações ativas" (`coberturas2`/tabela `coberturas`) — a
  substituição definitiva fica para depois que todas as fases estiverem prontas (ver Decisão de migração no topo).
- ⚠️ **Não implementado nesta fase (propositalmente)**: criação automática do animal-cria ao registrar parto — isso é
  a Fase 6. O botão "Registrar parto" só marca `status='parida'` + `parto_real`; o ponto de gancho pra Fase 6 está
  comentado no código.
- Dependências: Fase 3.

## Fase 5 — Protocolos + kanban de saúde reprodutivo
- [ ] Tabela `protocolos_reproducao` (templates) + `protocolo_aplicado` (instância por gestação, D0 = confirmação)
- [ ] Estrutura de etapas em JSONB (`dia_relativo`, `tipo`, `descricao`, `obrigatorio`, `obs`)
- [ ] Aba nova no dashboard de Saúde: kanban "Protocolo Reprodutivo" (Em dia/Atenção 7d/Vencido) — mesmo padrão visual de vacinas/exames
- [ ] Inclusão na verificação sanitária automática de eventos ABCCC
- Dependências: Fase 4, módulo Saúde existente.

## Fase 6 — Cria automática no parto
- [ ] Ao registrar parto: cria animal com `status_cadastro = 'rascunho'` (nova coluna em `animais`)
- [ ] Campos herdados: pai (garanhão da fonte), mãe (SBB da égua), ciclo calculado pela data
- [ ] Ao informar SBB real: rascunho → ativo, busca automática ABCCC
- Dependências: Fase 4, módulo Animais.

## Fase 7 — Negociação de coberturas + financeiro
- [ ] Tabela `coberturas_negociadas` (venda/doação, comprador externo ou outro tenant Mimba)
- [ ] Fluxo comprador externo → lançamento de receita automático
- [ ] Fluxo comprador Mimba → notificação cross-tenant, aceite cria fonte automaticamente no tenant comprador
- [ ] Fluxo cadastro manual pelo comprador (sem depender do vendedor ser usuário Mimba)
- Dependências: Fase 1, módulo Financeiro (`lancamentos`), multi-tenant (`tenants`).
- ⚠️ Cross-tenant — **exige revisão obrigatória do `revisor-isolamento`** antes de mergear (spec seção 7 envolve leitura/escrita entre schemas de cabanhas diferentes via control-plane).

## Fase 8 — Job de encerramento de ciclo (automação)
- [ ] Edge Function agendada (cron, 1º de agosto): vence fontes com saldo > 0, cancela acasalamentos `em_curso`,
      recria fontes de garanhão próprio pro ciclo novo, expira cota/direito de uso vencidos
- Dependências: todas as fases anteriores.

---

## Glossário (da spec, seção 12)
Ciclo ABCCC = 1º/ago a 31/jul (notação "26/27"). RM = Registro de Mérito (eleva limite 120→150).
Saldo de cobertura = adquirida − confirmadas − negociadas, decrementado só na confirmação. D0 = data de confirmação de prenhez.

## Referência
Documento fonte completo: `~/Downloads/spec-reproducao-mimba-v2.docx` (não versionado — spec interna do sócio).
