---
name: testar-provisionamento
description: Testa o provisionamento de cabanha ponta a ponta (signup → webhook → schema isolado) sem pagamento real, e verifica o isolamento. Use após mudar a RPC de provisionamento, a edge function, o webhook ou as policies.
---

# Testar provisionamento (Asaas → cabanha isolada)

Valida o ciclo completo simulando o evento do Asaas. O MCP é read-only → o usuário roda os writes (SQL Editor / curl).

## Passos
1. **Inserir signup de teste** (SQL Editor) — faz o papel do checkout:
   ```sql
   insert into public.signups (nome_cabanha, email_admin, plano_id, asaas_subscription_id, asaas_customer_id, status)
   values ('QA <algo>', 'qa.<algo>@example.com', '<plano_id>', 'sub_qa_<n>', 'cus_qa_<n>', 'aguardando_pagamento')
   returning id, asaas_subscription_id;
   ```
2. **Disparar o webhook** (terminal) — simula pagamento confirmado (usuário troca o token):
   ```bash
   curl -s -X POST "https://<ref>.supabase.co/functions/v1/asaas-webhook" \
     -H "asaas-access-token: <ASAAS_WEBHOOK_TOKEN>" -H "Content-Type: application/json" \
     -d '{"event":"PAYMENT_CONFIRMED","payment":{"id":"pay_qa_<n>","customer":"cus_qa_<n>","subscription":"sub_qa_<n>","value":97,"status":"CONFIRMED"}}'
   ```
   Esperado: `{"provisionado":true,"tenant_id":"...","schema_name":"cab_qa_<algo>"}`.
3. **Verificar isolamento** (MCP read-only + curl):
   - `signups.status`=`provisionado`; tenant criado; membership `adm`; admin com `senha_hash` NULL.
   - Policies: **0 para `anon`**, ~19 para `authenticated`; `has_schema_privilege('anon', schema, 'USAGE')`=false.
   - curl com `Accept-Profile: cab_qa_<algo>` + anon key → **HTTP 401**.
   - `provision_log`: `iniciado → schema_criado → usuario_criado → schema_exposto → concluido`.
4. **Limpar** (SQL + manual):
   ```sql
   delete from public.provision_log where tenant_id='<id>';
   delete from public.tenant_memberships where tenant_id='<id>';
   delete from public.signups where asaas_subscription_id='sub_qa_<n>';
   delete from public.tenants where id='<id>';
   drop schema if exists cab_qa_<algo> cascade;
   ```
   Manual: remover o schema de **Exposed schemas** (Settings → API) e apagar a identidade de teste em **Authentication → Users**.

## Regra
Se qualquer verificação de isolamento falhar (anon lê algo, policy anon existe), é **bug crítico** — pare e chame o `revisor-isolamento`.
