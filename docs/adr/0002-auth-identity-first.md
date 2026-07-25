# 0002 — Auth identity-first (Supabase Auth + membership + RLS)

**Status:** Aceito (2026-07). Substituiu o login legado (senha em texto puro comparada via query string, RLS `allow_all` para `anon`).

## Contexto
O login original comparava senha em texto puro na query string e o RLS era `allow_all` para `anon` — sem isolamento real. Além disso, um profissional/dono pode atuar em **mais de uma cabanha** (mesma pessoa, N cabanhas).

## Decisão
- **Identidade global** no **Supabase Auth** (`auth.users`, email+senha, bcrypt, JWT).
- **`tenant_memberships`** liga uma identidade a N cabanhas, com um `perfil` por cabanha (adm/vet/cab).
- Login: email+senha → JWT → RPC `minhas_cabanhas()` → 1 cabanha entra direto, N abre **seletor**.
- O app usa o **JWT do usuário** nas queries; o RLS de cada `cab_*` exige `authenticated` + `tem_acesso_tenant(<tenant_id>)` envolto em `(select ...)`. `anon` não acessa schemas de cabanha.

## Consequências
- (+) Isolamento real e login único que suporta multi-cabanha (seletor).
- (+) Fim da senha em texto puro; JWT/bcrypt de graça pelo GoTrue.
- (−) Login é por email — usuários sem email (ex.: cabanheiro) precisam de um caminho (fluxo "convidar usuário", pendente).
- (−) Policies precisam do wrap `(select ...)` pra não ficar lentas (avaliação por linha).

## Alternativas consideradas
- **Login caseiro com hash próprio, sem JWT:** mantinha auth artesanal e não dava isolamento real (sem role autenticada pro RLS). Rejeitado.
- **Login por slug na URL (`?cab=`):** exigiria saber a URL da cabanha, permitiria enumerar cabanhas e não resolvia multi-cabanha. Rejeitado.
