# Handoff — Mimba · checkpoint 2026-07-26

> Documento de retomada. Condensa o que já foi construído e o que falta. **Não contém segredos.**

## Como retomar (2 trilhas)
- **App/produto Mimba:** sessão em `projetos/cabanha` → *"lê o HANDOFF.md e vamos continuar"*. Carregam sozinhos: CLAUDE.md, memória (`MEMORY.md`), subagentes (`revisor-isolamento`, `arquiteto`, `engenheiro-frontend`) e skills (`nova-migration-tenant`, `deploy`, `testar-provisionamento`).
- **Landing:** sessão em `projetos/mimba-landing` (repo `mimba-app/mimba-landing`, clonado). O `index.html` é um bundle gerado; as páginas `/assinar` e `/obrigado` são hand-authored (editáveis à vontade).

## ▶️ PRÓXIMO PASSO: e-mail de acesso
É o elo que falta pro self-serve fechar: hoje o cliente paga → a cabanha é provisionada → **mas ninguém recebe como entrar**. Objetivo: ao final do provisionamento, enviar ao admin um email com **email + senha temporária + link `app.mimba.com.br`**.
- **Onde a senha temp está:** a Edge Function `provisionar-cabanha` gera `senha_temp = slug[0:6] + '@' + ano` e a retorna no JSON (`senha_temporaria`), **só quando `identidade_nova=true`**. O `asaas-webhook` chama a `provisionar-cabanha` e recebe esse retorno — hoje **não faz nada** com ele. O passo do email liga aqui: webhook → após provisionar com sucesso → envia o email.
- **Caso multi-cabanha** (`identidade_nova=false`, email já existia): não há senha nova → o email deve dizer "use sua senha atual; a cabanha X foi adicionada à sua conta".
- **Provedor:** vão usar o gmail da empresa (`app.mimba@gmail.com`). Opções: (a) **SMTP do Gmail** com App Password (simples); (b) provedor transacional (Resend/Sendgrid — melhor entregabilidade, precisa verificar domínio). Decidir isso é o 1º passo da etapa.
- **Onde implementar:** nova edge function `enviar-acesso` (ou inline no webhook). Marcar `provision_log.status = 'email_enviado'` (o CHECK já prevê).

## Arquitetura (o que existe e funciona)
- **App:** `index.html` único, sem framework/bundler, GitHub Pages. Em **`app.mimba.com.br`** (HTTPS). Marca Mimba (paleta terra/ouro/campo/creme; DM Sans + Playfair + DM Mono).
- **Backend:** Supabase `fmjfvfufkqswweyasjyp` (Postgres + Edge Functions Deno + Auth). anon key pública (no index.html).
- **Multi-tenant por schema:** cada cabanha = `cab_<slug>`. `public` é o **template** (tabelas vazias). Control-plane em `public`.
- **Login identity-first:** Supabase Auth (email+senha, JWT). `tenant_memberships` (identidade → N cabanhas + perfil). `minhas_cabanhas()` → 1 entra direto, N abre seletor. App usa o **JWT do usuário** nas queries.
- **Isolamento (RLS):** policies só `authenticated` via `tem_acesso_tenant(<tenant_id>)` em `(select ...)`. `anon` sem acesso aos schemas de cabanha (→ 401). Cabanha nova nasce isolada.

## Fluxo de contratação (self-serve) — construído e testado no sandbox
```
mimba.com.br/assinar (form) → Edge Function criar-checkout → Asaas (cliente + assinatura mensal)
   → insere signup → devolve invoiceUrl (+ callback p/ /obrigado)
cliente paga (Asaas) → redireciona p/ mimba.com.br/obrigado
Asaas → asaas-webhook (valida token) → provisionar-cabanha → cabanha isolada    ✅ testado ponta a ponta
```
- Decisões: **cobrança imediata** (provisiona só após pagamento confirmado); assinatura **mensal**; cliente escolhe forma de pagamento (`billingType: UNDEFINED`); pagamento na **página hospedada do Asaas** (sem tocar em cartão).
- **Recorrência:** cartão cobra automático/mês; PIX/boleto gera invoice/mês (cliente paga manual). Cada pagamento dispara `PAYMENT_CONFIRMED` → webhook **idempotente** (2º mês+ não re-provisiona). Inadimplência (`PAYMENT_OVERDUE` → suspender) = futuro.

## Edge Functions
- `criar-checkout` (verify_jwt=false) — form → Asaas → signup → invoiceUrl. Usa `ASAAS_API_KEY` + `ASAAS_BASE_URL` (default sandbox). Tem `callback` p/ /obrigado.
- `asaas-webhook` (verify_jwt=false) — valida `asaas-access-token` = `ASAAS_WEBHOOK_TOKEN`; nos eventos PAYMENT_CONFIRMED/RECEIVED chama a `provisionar-cabanha`.
- `provisionar-cabanha` (verify_jwt=false, exige `Bearer service_role`) — cria tenant, RPC, admin no auth.users, membership, expõe schema (Management API).
- `buscar-abccc`, `analise-sangues` (features do app).
- RPCs: `provisionar_schema_cabanha(p_schema)`, `minhas_cabanhas()`, `tem_acesso_tenant(uuid)`, `buscar_auth_user_por_email`, `vincular_admin_cabanha`.

## Secrets (nomes; valores só no Supabase)
- `SB_MGMT_TOKEN` (Management API — expor schema).
- `ASAAS_WEBHOOK_TOKEN` (= token dos webhooks Asaas; mesmo valor serve sandbox e prod).
- `ASAAS_API_KEY` (chave da API Asaas — **sandbox** agora; trocar p/ prod no go-live). ⚠️ inclui o `$` do começo.
- `ASAAS_BASE_URL` (opcional; default sandbox `https://api-sandbox.asaas.com/v3`; no go-live setar prod `https://api.asaas.com/v3`).

## Estado por frente
| Frente | Estado |
|---|---|
| Login identity-first + isolamento | ✅ no ar |
| Provisionamento automático | ✅ testado |
| Marca Mimba no app | ✅ no ar |
| **Domínio + migração de conta** (mimba-app) | ✅ concluído (mimba.com.br + app.mimba.com.br, HTTPS) |
| Estrutura de dev (.claude agents/skills, docs/adr) | ✅ versionada |
| **Checkout self-serve (sandbox)** | ✅ testado ponta a ponta + UX (/obrigado, voltar) |
| **E-mail de acesso** | ❌ **próximo passo** |
| Linkar "Contratar" da home → /assinar | ⏳ (outra sessão trata a landing) |
| Cutover Asaas p/ produção | ❌ (trocar ASAAS_API_KEY p/ prod + ASAAS_BASE_URL + ativar webhook prod) |
| reCAPTCHA/rate-limit no criar-checkout | ❌ (antes de expor de verdade — é público) |
| Migrar os 3 usuários da Mãe de Deus | ⏳ (precisam de email) |
| Convidar usuário (vet/cabanheiro) no app | ❌ (edge fn cria auth identity + membership) |
| Inadimplência (PAYMENT_OVERDUE → suspender) | ❌ futuro |
| Refactor do index.html | ❌ futuro (ADR 0004 com o `arquiteto`) |

## Limpeza pendente (rodar no SQL Editor)
Sobraram 2 tenants de teste da Fase 5:
```sql
do $$ declare v uuid[]; begin
  select array_agg(id) into v from public.tenants where slug in ('qa_isolamento','qa_segunda');
  if v is not null then
    delete from public.provision_log where tenant_id = any(v);
    delete from public.tenant_memberships where tenant_id = any(v);
    delete from public.tenants where id = any(v);
  end if;
end $$;
drop schema if exists cab_qa_isolamento cascade;
drop schema if exists cab_qa_segunda cascade;
```
Manual: remover `cab_qa_isolamento`/`cab_qa_segunda` de **Exposed schemas**; apagar a identidade de teste `pportella23@gmail.com` em **Authentication → Users** (se ainda existir).

## Gotchas (já mordido)
- **Trocou secret de Edge Function → redeploy a função** (a instância usa o valor antigo em cache). Bateu com API key E webhook token.
- **Secret com `$`/especial via CLI → aspas simples** (`'...'`), ou use o painel (o shell come o `$`). A API key do Asaas começa com `$aact_`.
- `verify_jwt` deve ser **false** em webhook/funções públicas (Asaas/landing não mandam JWT do Supabase).
- `perfil` é enum `adm/vet/cab` (não 'admin'). `sangues_linhagem.total_anc` é coluna gerada (insert com lista explícita). `tenants`: email_admin/asaas_customer_id **não** únicos.
- MCP do Supabase é **read-only** → writes vão pelo SQL Editor.

## Repos e domínios
- App: `mimba-app/cabanha` → **app.mimba.com.br**. `origin` local já atualizado. `pportella23` é colaborador (push, sem admin).
- Landing: `mimba-app/mimba-landing` → **mimba.com.br**. Páginas `/assinar` e `/obrigado`.
- Deploy do app: push na `main` → Pages + `versionar.yml`.

Memória do projeto (auto-carrega): `MEMORY.md` + arquivos (migração tenant, segurança/auth, provisionamento, hospedagem/domínio).
