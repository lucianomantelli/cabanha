---
name: nova-migration-tenant
description: Playbook para migrations que alteram o template (public) e precisam refletir em TODOS os schemas de cabanha (cab_*) já existentes. Use quando adicionar/alterar coluna, tabela, policy, trigger ou índice que deva valer para todas as cabanhas.
---

# Migration que reflete em todos os tenants

No Mimba, `public` é o **template** do provisionamento: cabanhas novas clonam dele (`CREATE TABLE ... LIKE public.x INCLUDING ALL`). Cabanhas **já existentes** (`cab_*`) **não** herdam mudanças automaticamente — é preciso aplicar em cada schema. Toda alteração estrutural precisa ir no template **e** ser replicada.

## Regras
- O MCP do Supabase é **read-only**: você **gera o SQL revisado**, o usuário aplica no **SQL Editor**. Nunca tente aplicar direto.
- Sempre idempotente (`IF NOT EXISTS`, `DROP POLICY IF EXISTS`, etc.).
- Se alterar policy/RLS/grants, passe pelo subagente `revisor-isolamento` antes.

## Passos

### 1. Alterar o template `public`
Aplique a mudança em `public.<tabela>` (a versão vazia). Ex.: `alter table public.usuarios add column if not exists x text;`

### 2. Replicar em todos os schemas de cabanha existentes
Rode um bloco que itera sobre os tenants provisionados:
```sql
do $$
declare r record;
begin
  for r in select schema_name from public.tenants
           where provisionado = true
             and exists (select 1 from pg_namespace where nspname = schema_name)
  loop
    execute format('alter table %I.<tabela> add column if not exists x text', r.schema_name);
    -- ... demais alterações, sempre com format(%I) e idempotentes
  end loop;
end $$;
```

### 3. Atualizar o provisionamento se necessário
Se a mudança for em algo que a RPC `provisionar_schema_cabanha` recria explicitamente (policies, triggers, grants, sequence do audit_log) — e não apenas herdado pelo `LIKE` —, atualize a RPC também, senão cabanhas **novas** nascem sem a mudança. Colunas simples vêm de graça pelo `LIKE`; policies/triggers/grants são emitidos pela RPC.

### 4. Verificar
- Diff estrutural entre `public` e um `cab_*` (deve bater no que interessa).
- Contagens/estado conforme o caso.
- Se tocou isolamento: teste anon → 401 e login do admin.

## Armadilhas conhecidas (já mordido)
- **`perfil`** é enum `perfil_usuario` = `adm`/`vet`/`cab` — não aceita 'admin'.
- **Colunas geradas** (ex.: `sangues_linhagem.total_anc` é `GENERATED`): não dá pra `INSERT ... SELECT *`; liste colunas explicitamente pulando a gerada.
- **`provision_log.status`** tem CHECK com lista fixa de valores — status novo precisa entrar no CHECK.
- **Policies** devem ser `TO authenticated` + `tem_acesso_tenant` **envolto em `(select ...)`** (senão fica lento, avaliado por linha).
- **`tenants`**: `email_admin` e `asaas_customer_id` **não** são únicos (multi-cabanha); `slug`/`schema_name`/`asaas_subscription_id` são.
- Rode tudo em transação quando possível, pra rollback limpo em caso de erro.
