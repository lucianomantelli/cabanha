# Plano de entrega — V1.5 até 29/08/2026

> Origem: `docs/roadmap-gestao-crioulo.pdf` (roadmap geral, marco "V1.5 · Consolidação da plataforma ·
> Agosto 2026"). Decisão do Pedro em 2026-08-02: meta é entregar a V1.5 **completa** até dia 29.
> Faltam ~27 dias. Este documento quebra os 6 itens da V1.5 em fases executáveis, na ordem recomendada.

## Panorama antes de começar

`staging` está **53 commits à frente de `main`** — todo o trabalho recente (Reprodutivo v3, redesign,
correção de convite, mobile) só existe em homologação. Isso é a Fase 0 deste plano: sem promover, cada
feature nova da V1.5 é construída em cima de uma base não publicada, aumentando o risco de acumular ainda
mais defasagem.

## Fases

### Fase 0 — Fundação (destrava tudo, baixo risco, faz primeiro)
- QA de ponta a ponta do Reprodutivo v3 em `mimba-hml.pages.dev` com login/dado reais (pendência já
  registrada no HANDOFF).
- Reverter `ambiente_teste=false` na Cabanha Santa Adelina (cliente real, hoje marcada como teste por um
  contorno temporário).
- Promover `staging` → `main` (skill `deploy`) — publica tudo que já está pronto de uma vez.
- **Sem isso, os itens abaixo nascem em cima de uma base que ainda não é produção de verdade.**

### Fase 1 — Personalização de cores por cabanha (rápido, isolado)
- `tenants.cor_primaria` já existe no banco (não usado). Aplicar como CSS variable no login/casca do app
  (mesmo padrão de `logo_url`, já usado hoje). Ponto de edição: Tela de Conta → aba Cabanha (onde já se
  edita nome/logo).
- Menor esforço de todo o roadmap V1.5 — bom pra abrir o sprint com uma entrega rápida.

### Fase 2 — Trial automático de 30 dias (decisão de produto já tomada: cartão tokenizado)
- Modelo definido: cadastro coleta cartão via Asaas (tokenizado), mas **não cobra na hora** — só ativa a
  cobrança automática no dia 30. Isso muda o fluxo `criar-checkout` atual (que hoje cobra imediatamente) —
  precisa de um novo fluxo "assinar com trial" em paralelo ao checkout imediato existente, não substituindo.
- `tenants.trial_inicio`/`trial_fim` já existem no banco (sem lógica hoje) — usar como base.
- Precisa de: (a) fluxo de cadastro que tokeniza cartão sem cobrar, (b) job periódico (`pg_cron`, já
  instalado no projeto) que verifica `trial_fim` vencido e dispara cobrança via Asaas, (c) tratamento de
  falha de cobrança (cartão recusado no dia 30 → o quê? bloquear? tentar de novo? — **decisão de produto
  ainda em aberto, resolver antes de implementar o job**).
- Maior peça de trabalho do sprint — depende de decisões de produto adicionais no caminho.

### Fase 3 — Painel admin da plataforma + Dashboard de métricas de uso (mesma audiência, construir junto)
- Público interno Mimba (não o cliente cabanha) — visão de todas as cabanhas ativas, status de
  assinatura, métricas de uso (animais cadastrados, módulos usados, frequência de acesso), logs de
  provisionamento.
- Precisa de um conceito de "usuário Mimba/staff" que hoje só existe fragmentado (`usuarios_master`,
  1 conta única `admin@arandu.app`, achado em sessão anterior) — vale revisar com o `arquiteto` antes de
  implementar, já que é RLS/acesso cross-tenant de verdade (não é por schema de cabanha).
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

## Pendências de decisão de produto (bloqueiam Fases 2-4 se não resolvidas antes)
1. Trial (Fase 2): o que acontece se a cobrança do dia 30 falhar (cartão recusado)? Bloqueia a cabanha na
   hora, tenta de novo em X dias, ou dá um prazo de carência?
2. Painel admin (Fase 3): quem tem acesso — só o Pedro/sócio, ou uma equipe maior? Precisa de um novo
   sistema de login separado do login de cabanha, ou reaproveita a mesma auth com um flag de "staff"?
3. Portal do cliente (Fase 4): troca de plano é self-service de verdade (upgrade/downgrade automático) ou
   só visualização + pedido que a Mimba processa manualmente por enquanto?
