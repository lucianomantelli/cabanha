# 0001 — Multi-tenant por schema Postgres

**Status:** Aceito (2026-07)

## Contexto
Mimba é SaaS multi-tenant: várias cabanhas, dados que **nunca** podem vazar entre elas. Precisávamos de um modelo de isolamento sustentável por 1 pessoa mantendo o sistema, sobre Supabase (Postgres + PostgREST).

## Decisão
Cada cabanha = um **schema Postgres próprio** (`cab_<slug>`). O schema `public` é o **template** (tabelas operacionais, mantidas vazias). O app seleciona o schema por request via headers `Accept-Profile`/`Content-Profile` do PostgREST.

## Consequências
- (+) Isolamento forte e fácil de raciocinar — a fronteira é o schema.
- (+) Provisionar = clonar a estrutura do template.
- (+) Cabe no PostgREST/Supabase sem serviço extra.
- (−) Cada schema novo precisa ser **exposto** no PostgREST (feito via Management API no provisionamento).
- (−) Mudança no template precisa refletir em todos os `cab_*` (skill `nova-migration-tenant`).
- (−) Muitos schemas alongam a lista de *exposed schemas* — aceitável no volume atual; revisitar em escala grande.

## Alternativas consideradas
- **Uma tabela com `tenant_id` + RLS por linha:** menos schemas, mas o isolamento vira responsabilidade de cada policy/query — mais fácil vazar por engano. Rejeitado pelo risco.
- **Um banco/projeto por cabanha:** isolamento máximo, custo/operacional inviável pra 1 pessoa. Rejeitado.
