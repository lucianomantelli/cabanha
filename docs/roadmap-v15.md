# Plano de entrega — V1.5 até 29/08/2026

> Origem: `docs/roadmap-gestao-crioulo.pdf` (roadmap geral, marco "V1.5 · Consolidação da plataforma ·
> Agosto 2026"). Decisão do Pedro em 2026-08-02: meta é entregar a V1.5 **completa** até dia 29.
> Faltam ~27 dias. Este documento quebra os 6 itens da V1.5 em fases executáveis, na ordem recomendada.

## Panorama antes de começar

`staging` está **53 commits à frente de `main`** — todo o trabalho recente (Reprodutivo v3, redesign,
correção de convite, mobile) só existe em homologação.

**Decisão de 2026-08-02: a promoção `staging→main` (Fase 0) fica pra DEPOIS, não agora.** Thiago (sócio,
admin da Cabanha Santa Adelina) usa a staging pra testar as funcionalidades novas conforme são construídas
— a Cabanha Santa Adelina fica **de propósito** marcada `ambiente_teste=true` (não é um contorno a reverter).
Ordem combinada: construir as Fases 1-4 da V1.5 primeiro, com a `main` estável do jeito que está pros
clientes reais que já usam produção, e só promover tudo de uma vez (com a Fase 0 completa: QA + reverter
flags de teste que façam sentido + deploy) quando o pacote da V1.5 estiver pronto pro lançamento.

## Fases

### Fase 0 — Fundação (QA + promoção pra produção) — **adiada, fazer por último**
- QA de ponta a ponta do Reprodutivo v3 em `mimba-hml.pages.dev` com login/dado reais.
- Revisar (não necessariamente reverter — Santa Adelina fica teste de propósito) quais cabanhas devem
  voltar a `ambiente_teste=false` antes do lançamento.
- Promover `staging` → `main` (skill `deploy`) — junto com o lançamento da V1.5, não antes.

### Fase 1 — Personalização de cores por cabanha ✅ CONCLUÍDA (2026-08-02)
- Campo "Cor da marca" (color picker) na Tela de Conta → aba Cabanha, ao lado de nome/logo — mesmo padrão
  de edição já usado ali.
- Aplicada via CSS variables (`--green`/`--green-d`/`--green-l`, o acento primário usado em botões e estado
  ativo do menu) sobrescritas por inline style em `:root` — vence a cascata sem duplicar regra nenhuma.
  `--green-l` (fundo claro) é calculado misturando a cor com branco (88%), não precisa o usuário escolher 3
  tons. Aplica no login (`_entrarApp`), some no logout (volta ao verde padrão Mimba).
- Backend: `minhas_cabanhas()` **já devolvia** `cor_primaria` (achado — já estava pronto de uma sessão
  anterior, só não era usado). `atualizar_tenant` RPC atualizada pra aceitar e persistir `p_cor_primaria`
  (parâmetro com `default null`, mantém compatibilidade com chamadas antigas).
- Testado via servidor local: cor aplicada no login, carregada certa ao abrir a Tela de Conta, "Restaurar
  padrão" funciona, e o logout limpa a cor customizada (volta ao verde Mimba).

### Fase 2 — Trial automático de 30 dias ✅ CONSTRUÍDA (2026-08-02) — falta 1 passo manual seu
- **Banco**: `tenants.asaas_card_token` (novo). `minhas_cabanhas()` passou a devolver cabanhas com
  `status in ('ativo','trial','bloqueado')` — antes só `'ativo'`, o que deixaria uma cabanha em trial
  **invisível no login** (achado no meio da implementação).
- **`provisionar-cabanha`** (v14): aceita `status`/`asaas_card_token` opcionais (default `'ativo'`,
  compatível com o fluxo de pagamento imediato existente).
- **Nova edge function `criar-checkout-trial`** (pública, chamada pela landing): tokeniza o cartão no Asaas
  (`/creditCard/tokenize`, nada é cobrado nesse passo) e provisiona a cabanha na hora com `status:'trial'`
  — não espera o `asaas-webhook`, porque não há pagamento ainda pra confirmar.
- **Nova edge function `cobrar-trial`** (interna, só `Bearer service_role`): roda 1x/dia via `pg_cron`,
  busca tenants com `status='trial'` e `trial_fim` vencido, cria a assinatura de verdade no Asaas com o
  cartão tokenizado. Sucesso → `status='ativo'`. Falha → `status='bloqueado'` na hora (decisão já tomada:
  sem retry/carência nesta versão).
- **Frontend**: tela de bloqueio (`login-bloqueado`) quando a cabanha está `status='bloqueado'` — barra
  a entrada antes de buscar qualquer dado. Aviso de "X dias restantes" na sidebar quando `status='trial'`.
- **⚠️ Achado da revisão de isolamento, corrigido**: o bloqueio acima era só client-side — a RLS não sabia
  nada sobre `status`, então um JWT válido + membership continuava com acesso total via API direta a uma
  cabanha "bloqueada". Corrigido na raiz: `tem_acesso_tenant()` (usada por **todas** as policies de
  **todos** os schemas `cab_*`) agora exige `tenants.status in ('ativo','trial')` além do membership — uma
  função só, efeito em todo lugar, sem precisar tocar policy por policy. Verificado que os 7 tenants
  provisionados existentes (`status='ativo'`) não foram afetados.
- Demais achados da revisão (payload de `criar-checkout-trial`, loop de `cobrar-trial`, colunas devolvidas
  por `minhas_cabanhas()`) — conferidos direto no código real: sem problema.
- **⚠️ Falta um passo manual seu antes disso rodar de verdade**: agendar o `pg_cron` exige a
  `service_role key` guardada no Vault do banco — só você tem essa chave (Project Settings → API). SQL
  pronto em `docs/migrations/2026-08-02-cron-cobrar-trial.sql`, é só colar a chave e rodar no SQL Editor.
- **Fora do escopo desta sessão**: o formulário de cadastro com trial na landing (`mimba.com.br/assinar`)
  vive no repo `mimba-landing`, separado deste — o backend já está pronto pra receber a chamada
  (`criar-checkout-trial`), mas o formulário em si (campos de cartão/endereço) precisa ser construído numa
  sessão nesse outro repo.

### Fase 3 — Painel admin da plataforma + Dashboard de métricas de uso (mesma audiência, construir junto)
- Público interno Mimba (não o cliente cabanha) — visão de todas as cabanhas ativas, status de
  assinatura, métricas de uso (animais cadastrados, módulos usados, frequência de acesso), logs de
  provisionamento.
- **Decidido — acesso restrito a Pedro e sócio por enquanto.** Reaproveita o login atual (Supabase Auth):
  um flag simples de "staff" (ex.: em `usuarios_master`, hoje com 1 conta única `admin@arandu.app`, ou uma
  tabela nova `mimba_staff` com os `auth.users.id` autorizados) — não precisa de sistema de auth novo, só
  uma policy/RPC que checa esse flag antes de expor dado cross-tenant. Ainda assim é RLS/acesso cross-tenant
  de verdade (não por schema de cabanha) — vale uma revisão rápida com o `revisor-isolamento` antes de
  liberar, mesmo sendo só 2 contas.
- Combinar os dois itens do roadmap geral numa tela só evita duas iniciativas separadas pro mesmo público.

### Fase 4 — Portal do cliente + Relatórios em PDF (cliente-facing, mais isolados entre si)
- **Portal do cliente**: gestão de conta, histórico de faturas, troca de plano, atualização de forma de
  pagamento — depende da integração Asaas já existente (expandir, não recriar) e se beneficia da Fase 2
  (tokenização de cartão) já estar pronta.
- **Relatórios em PDF**: sanidade do plantel, histórico sanitário por animal, relatório de gestações,
  análise de sangues — geração client-side (`jsPDF` ou similar) ou via Edge Function; independente das
  outras fases, pode ser paralelizado por outra frente se houver.

## Risco explícito sobre o prazo

27 dias pra 6 iniciativas (uma delas — trial automático — com decisões de produto ainda pendentes, e outra
— painel admin — tocando um conceito de acesso que não existe hoje) é um sprint pesado pra uma pessoa
construindo sozinha. As Fases 0 e 1 são de baixo risco e alta alavancagem (fazer primeiro). A partir da
Fase 2, cada decisão de produto não resolvida antes de começar a construir consome tempo do sprint. Recomendo
revisar o progresso na Fase 0/1 concluída como checkpoint pra confirmar se o restante do prazo é realista ou
se algo precisa ser recortado — melhor decidir isso cedo do que descobrir no dia 27.

## Pendências de decisão de produto

Resolvidas em 2026-08-02: falha de cobrança no trial (Fase 2) → bloqueio imediato, sem retry/carência.
Acesso ao painel admin (Fase 3) → só Pedro + sócio, via flag de staff reaproveitando o login atual.

Ainda em aberto (não bloqueia o início das Fases 0-3, só precisa ser resolvida antes da Fase 4):
1. Portal do cliente (Fase 4): troca de plano é self-service de verdade (upgrade/downgrade automático) ou
   só visualização + pedido que a Mimba processa manualmente por enquanto?
