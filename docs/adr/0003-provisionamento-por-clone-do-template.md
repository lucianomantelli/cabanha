# 0003 — Provisionamento por clone do template

**Status:** Aceito (2026-07). Substituiu a versão anterior (DDL escrita à mão na função).

## Contexto
A função de provisionamento original recriava cada tabela com **DDL escrita à mão**, que divergiu do schema real do app — toda cabanha nova nascia "torta" (colunas/tipos errados, sem triggers, RLS aberto). Precisávamos de uma **fonte única da verdade**.

## Decisão
O provisionamento **clona a estrutura do `public`** (`CREATE TABLE cab_x.t (LIKE public.t INCLUDING ALL)`) — **nunca DDL manual**. A RPC `provisionar_schema_cabanha(p_schema)` também emite RLS por membership (só `authenticated`), grants só authenticated, replica os triggers do `public` e dá sequence própria ao `audit_log`. A Edge Function `provisionar-cabanha` cria o admin no `auth.users` (reaproveita se o email já existe), grava membership + linha no schema, e expõe o schema via Management API. A única porta pública é o `asaas-webhook` (valida o token do Asaas); `provisionar-cabanha` exige `Bearer service_role`.

## Consequências
- (+) Cabanha nova nasce idêntica ao app e **isolada por padrão** (anon → 401). Verificado ponta a ponta.
- (+) Evoluir o schema = mexer no template; cabanhas novas herdam de graça.
- (−) Cabanhas existentes ainda precisam da replicação manual (skill `nova-migration-tenant`).
- (−) Depende da Management API pra expor schema (secret `SB_MGMT_TOKEN`).
- (−) Colunas geradas exigem insert com lista explícita (ex.: `sangues_linhagem.total_anc`).

## Alternativas consideradas
- **Manter a DDL à mão:** foi a causa do bug; descartado.
- **Schema `template` dedicado** (em vez do `public`): mais limpo conceitualmente, mas adiciona um schema pra manter em sincronia; o `public` já é a fonte da verdade do app. Pode ser revisitado.
