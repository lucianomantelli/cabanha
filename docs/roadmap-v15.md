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
- **`pg_cron` agendado** (2026-08-02, rodado pelo Pedro): job `cobrar-trials-vencidos` ativo, todo dia às
  3h (`jobid=2`). Fase 2 fechada de ponta a ponta.
- **Fora do escopo desta sessão**: o formulário de cadastro com trial na landing (`mimba.com.br/assinar`)
  vive no repo `mimba-landing`, separado deste — o backend já está pronto pra receber a chamada
  (`criar-checkout-trial`), mas o formulário em si (campos de cartão/endereço) precisa ser construído numa
  sessão nesse outro repo.

### Fase 3 — Painel admin da plataforma + Dashboard de métricas de uso ✅ CONSTRUÍDA (2026-08-02)
- **`mimba_staff`** (tabela nova, RLS habilitada deny-all + grants revogados de anon/authenticated — só
  acessível via função `SECURITY DEFINER`) substitui a ideia de reaproveitar `usuarios_master` (achado:
  essa tabela é legada, com seu próprio `senha_hash`, **desconectada** do Supabase Auth atual — não dava
  pra reaproveitar sem refatorar o que já funciona). Hoje tem 2 linhas: Pedro e Thiago.
- **`sou_staff_mimba()`** — RPC que checa o flag. **`admin_listar_cabanhas()`** — RPC única que junta os 2
  itens do roadmap geral (painel + métricas): itera todos os tenants, agrega `animais_count`/
  `usuarios_count` por cabanha (dado agregado, nunca linha individual), devolve status/plano/trial/
  provisionamento. Ambas checam `sou_staff_mimba()` internamente — nega antes de tocar em qualquer dado se
  quem chama não é staff.
- **Frontend**: botão "Painel Mimba" na sidebar (só visível pra staff, checado a cada login) abre uma tela
  cheia separada — cross-tenant de verdade, não define `TENANT_SCHEMA` nenhum. Métricas no topo (cabanhas
  ativas/em teste/bloqueadas, total de animais na plataforma) + tabela por cabanha.
- **Revisão de isolamento**: aprovada. Gate atômico (sem race condition possível dentro de uma única
  invocação SQL), `schema_name` usado em `execute format()` vem só do provisionamento (não é input de
  usuário) e os 7 schemas reais batem 1:1 com `tenants.schema_name` — sem vetor de SQL injection. Único
  ajuste sugerido (não bloqueante, aplicado): `mimba_staff` ganhou `ENABLE ROW LEVEL SECURITY` como defesa
  em profundidade (mesmo sem grant hoje, protege contra um `GRANT` futuro concedido por engano).
- Testado via servidor local com `_rpc` simulado: botão aparece só pra staff, painel abre/fecha, métricas e
  tabela renderizam a partir do retorno da RPC, estado de erro (sem acesso/RPC falhou) cai num aviso
  amigável em vez de quebrar a tela.
- **Não incluído nesta fase** (ficou de fora do roadmap original e não foi pedido): logs de provisionamento
  detalhados por cabanha (`provision_log` já existe no banco, dá pra expor depois se fizer falta) e
  frequência de acesso por usuário (precisaria de tracking novo, não existe hoje).
- Combinar os dois itens do roadmap geral numa tela só evita duas iniciativas separadas pro mesmo público.

### Fase 4 — Portal do cliente + Relatórios em PDF (cliente-facing, mais isolados entre si)
- **Portal do cliente**: gestão de conta, histórico de faturas, troca de plano, atualização de forma de
  pagamento — depende da integração Asaas já existente (expandir, não recriar) e se beneficia da Fase 2
  (tokenização de cartão) já estar pronta.
  - **Decidido (2026-08-02): troca de plano é self-service de verdade**, não só pedido manual. Mas com
    validação — se o plano novo tiver limite que a cabanha já estoura (ex.: downgrade com mais animais
    cadastrados do que o plano novo permite), a troca **bloqueia** com uma mensagem explicando exatamente
    o que precisa ajustar antes (ex.: "reduza pra X animais ativos"), não deixa trocar e só regularizar
    depois.
  - **🚫 BLOQUEADO — aguardando o Pedro**: ainda não existe a lista de recursos/limites de cada plano
    (Potro/Arreio/Tropilha). Sem isso não dá pra escrever a validação (não sei o que checar). **Não
    implementar a troca de plano até essa lista chegar** — combinado explicitamente, não é esquecimento.
  - O resto do Portal do cliente (histórico de faturas, atualização de forma de pagamento) não depende
    dessa lista — pode ser construído em paralelo.
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
