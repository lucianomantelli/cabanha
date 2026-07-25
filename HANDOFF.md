# Handoff — Mimba · checkpoint 2026-07-25

> Documento de retomada. Condensa tudo que já foi construído e o que falta.
> Para continuar em sessão nova: leia este arquivo + os arquivos de memória do projeto (MEMORY.md).
> **Não contém segredos** (só nomes de secrets e onde vivem). Mantenha local se o repo for público.

## O que é
**Mimba** — SaaS multi-tenant de gestão para cabanhas de cavalo crioulo (marca: Mimba; razão social: Mimba Tech; nicho v1: "Gestão Crioulos", integra ABCCC). Domínios: **mimba.com.br** (landing) e **app.mimba.com.br** (sistema).

## Arquitetura (o que existe e funciona)
- **App:** `index.html` único, sem framework/bundler, hospedado no **GitHub Pages** (push na `main` → deploy; workflow `versionar.yml` arquiva as últimas 10 versões em `versions/`). ~457 KB.
- **Backend:** Supabase — projeto **`fmjfvfufkqswweyasjyp`** (Postgres + Edge Functions Deno + Auth). anon key é pública (está no index.html).
- **Multi-tenancy por schema Postgres:** cada cabanha = schema `cab_<slug>`. `public` é o **template** (tabelas operacionais vazias — nunca dropar).
- **Identidade/login (identity-first):** Supabase Auth (email+senha, bcrypt, JWT). `public.tenant_memberships` liga uma identidade a N cabanhas com um `perfil` (adm/vet/cab) por cabanha. Fluxo: email+senha → JWT → RPC `minhas_cabanhas()` → 1 cabanha entra direto, N abre seletor. O app envia o **JWT do usuário** (não a anon key) nas queries.
- **Isolamento (RLS):** cada schema de tenant tem policies só para `authenticated` via `public.tem_acesso_tenant(<tenant_id>)`, sempre envolto em `(select ...)` (performance = InitPlan). `anon` não tem policy/grant/USAGE nos schemas de cabanha → a anon key sozinha não lê nada. Verificado: cabanha nova nasce com anon → 401.
- **Provisionamento automático:** `signups` (checkout grava o que provisionar) → Edge Function **`asaas-webhook`** (valida `asaas-access-token`) → Edge Function **`provisionar-cabanha`** (exige `Authorization: Bearer <service_role>` — foi assim que o "verify_jwt" foi fechado) → RPC `provisionar_schema_cabanha(p_schema)` (clona estrutura do `public` via `CREATE TABLE (LIKE ... INCLUDING ALL)`, RLS por membership, grants só authenticated, triggers, sequence própria do audit_log) → cria admin no `auth.users` (reaproveita se o email já existe → multi-cabanha) + membership + linha no schema → **expõe o schema no PostgREST via Management API** (secret `SB_MGMT_TOKEN`). Cabanha nova nasce isolada. Validado ponta a ponta com evento simulado.
- **Marca Mimba** aplicada no app: paleta (verde campo `#2D5A3D` ação, ouro `#B8860B` acento, terra/creme fundos) via variáveis CSS (light+dark); fontes DM Sans (corpo), Playfair Display (títulos), DM Mono. Logo/nome da cabanha vêm de `tenants.logo_url`/`nome_cabanha` (não mais hardcoded).

## Objetos-chave no banco
- `public.tenants` — control-plane das cabanhas (slug, schema_name, email_admin, cnpj_cpf, abccc_codigo, nome_exibicao, logo_url, plano_id, status, asaas_*, provisionado…). Únicos: slug, schema_name, asaas_subscription_id. **NÃO** únicos: email_admin, asaas_customer_id (multi-cabanha).
- `public.signups` — contrato do checkout (nome_cabanha, email_admin, plano_id, cnpj_cpf, abccc_codigo, nome_exibicao, logo_url, asaas_*, status, tenant_id). Só service_role.
- `public.tenant_memberships` — (user_id→auth.users, tenant_id, perfil, ativo).
- `public.planos` — Potro `62b00a6d-6481-4c76-bb7e-cb3ff1b51ad1`, Arreio, Tropilha.
- `public.usuarios_master` — superadmin da plataforma (`admin@arandu.app`).
- RPCs: `provisionar_schema_cabanha(p_schema)`, `minhas_cabanhas()`, `tem_acesso_tenant(uuid)`, `buscar_auth_user_por_email(text)`, `vincular_admin_cabanha(...)` (as 2 últimas restritas a service_role).
- Edge Functions: `provisionar-cabanha` (v6, verify_jwt=false + guard service_role), `asaas-webhook` (verify_jwt=false), `buscar-abccc`, `analise-sangues`.
- Enum `perfil_usuario` = **adm/vet/cab** (não aceita 'admin'). `provision_log.status` tem CHECK com lista fixa (inclui `email_enviado`, `schema_exposto`).

## Cabanha piloto: Mãe de Deus
- schema `cab_mae_de_deus`, slug `mae_de_deus`, tenant_id `d8dde7d7-092b-42dd-9305-f77946323d2e`.
- Admin migrado pro Auth: **admin@maededeus.com.br** (perfil adm).
- **PENDENTE:** os outros 3 usuários (`veterinario`, `cabanheiro`, `ana.julia`) ainda **não** têm identidade no `auth.users` → não conseguem logar até serem migrados (precisam de emails). O app está no ar; só o admin loga hoje.

## Config / credenciais (onde estão — não os valores)
- Secrets nas Edge Functions: **`SB_MGMT_TOKEN`** (PAT Management API do Supabase), **`ASAAS_WEBHOOK_TOKEN`** (= token do webhook no Asaas).
- **Asaas:** webhook configurado em **produção** mas **DESATIVADO** (liga só no go-live). Sandbox não montado. Eventos: PAYMENT_CONFIRMED, PAYMENT_RECEIVED.
- **Supabase MCP** conectado em modo **read-only** (todo write é aplicado pelo usuário no SQL Editor / dashboard).
- **GitHub:** app em `lucianomantelli/cabanha` (a transferir); landing em `mimba-app/mimba-landing`. Conta nova **`mimba-app`** (email `app.mimba@gmail.com`) será a dona dos dois repos.

## Estado por frente
| Frente | Estado |
|---|---|
| Login identity-first + isolamento (RLS por membership) | ✅ feito, no ar |
| Provisionamento automático (webhook→schema isolado) | ✅ feito e testado |
| `verify_jwt` fechado | ✅ |
| Marca Mimba no app | ✅ no ar |
| Modelo de dados (ABCCC, logo, doc da cabanha) | ✅ |
| Landing publicada e renderizando | ✅ (`mimba-app.github.io/mimba-landing`) |
| Custom domain da landing (mimba.com.br) | ⏳ DNS **em transição no registro.br** (config correta, aguardando publicar) |
| Transfer `cabanha`→`mimba-app` + `app.mimba.com.br` | ⏳ com o sócio |
| Supabase Auth URL → app.mimba.com.br | ⏳ |
| Migrar os outros 3 usuários da Mãe de Deus | ⏳ quando tiver emails |
| **E-mail de acesso** ao cliente | ❌ pendente (definir provedor: domínio mimba comprado; falta subir serviço de email) |
| **Checkout na landing** (form → Asaas + insere signup) | ❌ pendente |
| **Convidar usuário** no app (vet/cabanheiro) | ❌ pendente (tela "Usuários" ainda no modelo antigo login+senha; precisa edge function criando auth identity + membership) |
| Ativar webhook no Asaas | ❌ só no go-live |

## DNS (registro.br) — já configurado, aguardando publicar
Apex `mimba.com.br` → 4x A `185.199.108–111.153` (Nome VAZIO no painel, não `@`). `www` e `app` → CNAME `mimba-app.github.io.`. Checar publicação: `dig +short @a.auto.dns.br mimba.com.br A` (quando retornar os 4 IPs, publicou → no Pages clicar "Check again" → Enforce HTTPS).

## Regras que não podem ser quebradas
- Nunca vazar dados entre cabanhas (isolamento por membership; anon sem acesso aos schemas de cabanha).
- Provisionamento clona do `public` (LIKE) — **nunca DDL à mão** (foi o bug original que gerava schema "torto").
- Não dropar as tabelas do `public` (é o template).
- Nunca commitar secrets (anon key é ok/pública; service_role, PATs e tokens não).
- Migration que altera o template `public.usuarios` precisa ser replicada em todos os schemas `cab_*` existentes.

## Próximos passos sugeridos (ordem)
1. **Fechar domínio:** DNS publicar → landing em mimba.com.br (+HTTPS) → transfer `cabanha`→`mimba-app` → custom domain `app.mimba.com.br` → Supabase Auth URL → atualizar `origin` local. (Blast radius do transfer é pequeno: backend é desacoplado do GitHub.)
2. **E-mail de acesso** (provedor a definir) — completa o loop do provisionamento (cliente recebe como entrar). `provision_log` já prevê `email_enviado`.
3. **Checkout na landing** — formulário → cria cliente/assinatura no Asaas + insere `signup` → ativar o webhook.
4. **Convidar usuário** no app (edge function: cria auth identity + membership; resolver caso do cabanheiro sem email).
5. **Ajustes de landing/design** — placeholder de imagem no herói, trocar `<title>` "Bundled Page", refinos.

## Histórico detalhado
Memória do projeto (carregada automaticamente em sessões novas): `MEMORY.md` + `migracao-tenant-cab-mae-de-deus`, `seguranca-debito-adiado`, `provisionamento-cabanha`, `hospedagem-e-dominio`. Scripts SQL e código das edge functions ficaram no scratchpad da sessão (não versionado) — os deployados estão no Supabase.
