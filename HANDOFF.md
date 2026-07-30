# Handoff — Mimba · checkpoint 2026-07-27 (pós-maratona pré-apresentação)

> Documento de retomada. Condensa o que já foi construído e o que falta. **Não contém segredos.**

## Como retomar (2 trilhas)
- **App/produto Mimba:** sessão em `projetos/cabanha` → *"lê o HANDOFF.md e vamos continuar"*. Carregam sozinhos: CLAUDE.md, memória (`MEMORY.md`), subagentes (`revisor-isolamento`, `arquiteto`, `engenheiro-frontend`) e skills (`nova-migration-tenant`, `deploy`, `testar-provisionamento`).
- **Landing:** sessão em `projetos/mimba-landing` (repo `mimba-app/mimba-landing`, clonado). O `index.html` é um bundle gerado; as páginas `/assinar` e `/obrigado` são hand-authored (editáveis à vontade).

## 🔴 O MAIS IMPORTANTE PRA SABER AGORA
Toda a maratona de correções desta sessão (ver `ROADMAP.md`) está **só na branch `staging`**, publicada em **`https://mimba-hml.pages.dev/`** via Cloudflare Pages — **nada disso está em produção** (`app.mimba.com.br`, branch `main`) ainda. A `main` só tem o que foi feito antes da `staging` existir (dashboard/toast fix + e-mail de acesso). **Antes da apresentação, decidir: promover `staging` → `main`?** (`git checkout main && git merge staging` + skill `deploy`, depois de testar tudo na URL de staging).

## ✅ ROADMAP — Prioridades 1 a 6 + vários itens extra: CONCLUÍDOS (nesta sessão, 2026-07-27)
Ver `ROADMAP.md` para o detalhe de cada um. Resumo rápido do que mudou no app (`index.html`, tudo na `staging`):
1. **Dashboard lento** → causa real era 12 requisições REST cada uma pagando preflight CORS; virou 1 RPC (`carregar_dados_cabanha`). Skeleton animado enquanto carrega.
2. **Tela de Conta** → card no rodapé da sidebar (estilo shadcn/ui, ícones SVG), editar nome/logo da cabanha, gestão de usuários unificada, **convite de usuário agora cria identidade real no Auth** (antes só gravava linha local, login não funcionava).
3. **Importar animais por lista de SBB** → cola lista ou `.txt`/`.csv`, busca em lote na ABCCC (3 em paralelo), insere tudo num POST só.
4. **Eventos não carregavam** → resolvido de bônus pela RPC do item 1 (PostgREST não via o relacionamento `eventos`↔`eventos_animais` nesse schema).
5. **Modal de detalhe do animal quebrado** → virou página cheia; corrigido bug de layout (texto colado) e um bug separado de nomes de campo (`participantes`/`animais`) que deixava a seção Eventos do detalhe sempre vazia.
6. **Responsividade mobile** → trava de `overflow-x` na raiz + `.tab-row` rolável + menu vira painel suspenso (antes era barra fixa de ícones).
7. **Login**: lembrar e-mail, mostrar/ocultar senha, **sessão sobrevive a recarregar a página** (access_token/refresh_token em localStorage, renova sozinho).
8. **Link SBB → ABCCC**: já existia mas nunca funcionava de verdade (tentava contornar cross-origin, impossível); corrigido pra fazer o POST direto contra o formulário real da ABCCC.
9. **Bug de Gestação/Medidas vazios na 1ª visita**: corrida com o sync do login — corrigido de forma abrangente (qualquer página ativa no momento em que o sync termina é re-renderizada).

### ⚠️ Pendente de confirmação — aplicado no banco, mas não reconfirmado nesta sessão
Dois artefatos da Tela de Conta (item 2 acima) foram **entregues** (SQL + edge function) mas eu não tenho confirmação explícita de que foram aplicados/deployados — **verificar antes de considerar esse item 100% funcional em qualquer ambiente**:
- Migration `migration-tela-conta.sql` (RPCs `atualizar_tenant`, `vincular_usuario_cabanha`, `revogar_acesso_usuario` + fix de grant em `vincular_admin_cabanha`).
- Edge function `convidar-usuario` (verify_jwt=**ON**, diferente das outras — é chamada direto pelo navegador do admin logado).
- A migration `migration-carregar-dados-cabanha.sql` (RPC do item 1) **essa sim foi confirmada aplicada e testada ao vivo**.

### Achado de segurança registrado, não corrigido (decisão deliberada)
A RLS de **todas** as tabelas de tenant (não só `usuarios`) libera INSERT/UPDATE/DELETE pra qualquer perfil ativo do tenant, não só admin — um cabanheiro logado podia, via API direta, se autopromover a admin. Pré-existente a esta sessão, sistêmico (toda tabela, todo schema). **Não mexido de propósito** — 2 dias antes de uma apresentação não é hora de reescrever RLS de todo o sistema. Ver memória `rls-permissiva-por-perfil` — retomar com o `arquiteto` depois de quarta.

## O que falta do ROADMAP (itens sem prioridade numerada, não feitos)
- Tela de Animais (V1) modernizar no padrão da tela de Gestações (V2).
- Tela de Animais: edição sem confirmação, editável direto na "planilha" (sem modo leitura/edição separado).
- Aba de Medidas: layout ainda precisa de melhoria visual (o *bug* de carregar foi corrigido; a estética não).
- Campo "situação"/"data de confirmação" no cadastro de novo animal.
- Buscar medidas de confirmação (altura/tórax/canela) da ABCCC pro cadastro.

## E-mail de acesso — construído (sessão anterior, ✅ já em produção)
Cliente paga → cabanha provisionada → admin recebe e-mail e define a própria senha (link recovery via Auth Admin `generateLink` + Resend, sem senha determinística). Edge functions `enviar-acesso`/`provisionar-cabanha`/`asaas-webhook` já deployadas e testadas ponta a ponta. Revisão de isolamento: ✅ aprovado.

## Arquitetura (o que existe e funciona)
- **App:** `index.html` único, sem framework/bundler. **Produção:** GitHub Pages, branch `main`, `app.mimba.com.br`. **Staging:** Cloudflare Pages, branch `staging`, `mimba-hml.pages.dev` — **mesmo banco Supabase da produção** (staging isola só código, não dados; só testar na "Cabanha Pedro Teste"; ver memória `staging-e-isolamento-de-dados`).
- **Backend:** Supabase `fmjfvfufkqswweyasjyp` (Postgres + Edge Functions Deno + Auth). anon key pública (no index.html).
- **Multi-tenant por schema:** cada cabanha = `cab_<slug>`. `public` é o **template** (tabelas vazias). Control-plane em `public`.
- **Login identity-first:** Supabase Auth (email+senha, JWT). `tenant_memberships` (identidade → N cabanhas + perfil). `minhas_cabanhas()` → 1 entra direto, N abre seletor. Sessão agora persiste entre reloads (ver item 7 acima).
- **Isolamento (RLS):** policies só `authenticated` via `tem_acesso_tenant(<tenant_id>)` em `(select ...)`. `anon` sem acesso aos schemas de cabanha (→ 401). Cabanha nova nasce isolada. *(Mas ver achado de segurança acima — escrita não é restrita por perfil ainda.)*

## Fluxo de contratação (self-serve) — construído e testado no sandbox
```
mimba.com.br/assinar (form) → Edge Function criar-checkout → Asaas (cliente + assinatura mensal)
   → insere signup → devolve invoiceUrl (+ callback p/ /obrigado)
cliente paga (Asaas) → redireciona p/ mimba.com.br/obrigado
Asaas → asaas-webhook (valida token) → provisionar-cabanha → cabanha isolada → enviar-acesso   ✅ testado ponta a ponta
```
- Decisões: **cobrança imediata**; assinatura **mensal**; cliente escolhe forma de pagamento; pagamento na **página hospedada do Asaas**.
- **Recorrência:** cartão cobra automático/mês; PIX/boleto gera invoice/mês. Webhook **idempotente**. Inadimplência = futuro.
- **Gotcha conhecido:** conta Asaas precisa ter um **site/domínio cadastrado** em "Minha Conta" pro `callback`/redirect do checkout funcionar, senão dá erro genérico "Falha ao criar assinatura". Vale pra sandbox **e** pra produção no cutover.

## Edge Functions (produção)
- `criar-checkout` (verify_jwt=false) — form → Asaas → signup → invoiceUrl.
- `asaas-webhook` (verify_jwt=false) — valida token; PAYMENT_CONFIRMED/RECEIVED → `provisionar-cabanha` → `enviar-acesso`.
- `provisionar-cabanha` (verify_jwt=false, exige `Bearer service_role`) — cria tenant, RPC, admin no auth.users (senha aleatória nunca divulgada), membership, expõe schema.
- `enviar-acesso` (verify_jwt=false, exige `Bearer service_role`) — link recovery + Resend.
- `convidar-usuario` (verify_jwt=**true**) — **enviada, aplicação não reconfirmada** (ver seção de pendências acima).
- `buscar-abccc`, `analise-sangues` (features do app).
- RPCs: `provisionar_schema_cabanha`, `minhas_cabanhas`, `tem_acesso_tenant`, `buscar_auth_user_por_email`, `vincular_admin_cabanha`, `carregar_dados_cabanha` (✅ aplicada), `atualizar_tenant`/`vincular_usuario_cabanha`/`revogar_acesso_usuario` (enviadas, **não reconfirmadas**).

## Secrets (nomes; valores só no Supabase)
- `SB_MGMT_TOKEN`, `ASAAS_WEBHOOK_TOKEN`, `ASAAS_API_KEY` (⚠️ `$` no início, sandbox agora), `ASAAS_BASE_URL` (opcional), `RESEND_API_KEY` (+ opcionais `RESEND_FROM`/`RESEND_REPLY_TO`/`APP_URL`/`ACCESS_REDIRECT_TO`).

## Estado por frente
| Frente | Estado |
|---|---|
| Login identity-first + isolamento + sessão persistente | ✅ no ar (staging) / parcial em prod |
| Provisionamento automático | ✅ testado |
| E-mail de acesso | ✅ em produção |
| Checkout self-serve (sandbox) | ✅ testado ponta a ponta |
| **ROADMAP Prioridades 1-6** | ✅ concluídas — **só na staging**, aguardando promover pra `main` |
| Tela de Conta (SQL/edge function) | ⚠️ enviado, aplicação não reconfirmada nesta sessão |
| RLS permissiva por perfil (achado de segurança) | ❌ registrado, não corrigido (pós-apresentação, c/ `arquiteto`) |
| Rename repo `cabanha`→`mimba` | ⏳ adiado pra depois de 29/07 (ver memória) |
| Linkar "Contratar" da home → /assinar | ⏳ (sessão da landing) |
| Cutover Asaas p/ produção | ❌ trocar `ASAAS_API_KEY`/`ASAAS_BASE_URL` + cadastrar domínio na conta Asaas de prod + redeploy `criar-checkout`/`asaas-webhook` |
| reCAPTCHA/rate-limit no `criar-checkout` | ❌ antes de expor de verdade (é público) |
| Migrar os 3 usuários da Mãe de Deus | ⏳ (agora que convite de usuário existe, destravar isso) |
| Refactor do index.html | ❌ futuro (ADR 0004 com o `arquiteto`) |

## Gotchas (já mordido)
- **Trocou secret de Edge Function → redeploy a função** (cache do valor antigo).
- **Secret com `$`/especial via CLI → aspas simples**, ou use o painel.
- `verify_jwt` **false** em webhook/funções públicas; **true** só na `convidar-usuario` (chamada direto pelo navegador do usuário logado).
- `perfil` é enum `adm/vet/cab`. `sangues_linhagem.total_anc` é gerada. `tenants`: email_admin/asaas_customer_id não únicos.
- MCP do Supabase agora tem **acesso completo** (não é mais read-only) — writes via `apply_migration`/`execute_sql` direto.
- **`convidar-usuario` (edge function, v3)**: dois bugs de reconvite corrigidos — (1) `revogar_acesso_usuario` só
  marca `tenant_memberships.ativo=false`, nunca apaga a linha, então reconvidar batia em 409 até a função aprender a
  reativar em vez de bloquear; (2) `identidadeNova` (decide qual e-mail mandar) usava só "acabei de criar a
  identidade agora?" — se o convite anterior expirou sem a pessoa nunca ter feito login, a identidade já existia
  mas sem senha nenhuma, e ela recebia "use a senha que você já tem" (que nunca existiu). Agora checa
  `last_sign_in_at IS NULL` via Admin API antes de decidir. `vincular_usuario_cabanha` (RPC) também virou idempotente
  (upsert por `login`) pro caso de reconvidar alguém só suspenso (não excluído — a linha local `usuarios` continua lá).
- **`tenants.ambiente_teste`** (bool, novo): segrega cabanhas de teste do staging/produção — `minhas_cabanhas()` devolve
  o campo, `index.html` filtra client-side (`AMBIENTE_STAGING = location.hostname==='mimba-hml.pages.dev'`). Não é
  barreira de segurança de verdade (mesmo banco/anon key nos dois ambientes) — é trava de UX. `true` hoje:
  `qa_isolamento`, `qa_segunda`, `cabanha_pedro_teste`. Marcar tenants novos de teste manualmente.
- Schema `public` tem `ALTER DEFAULT PRIVILEGES` do Supabase que concede `anon`/`PUBLIC` automaticamente em toda
  **tabela e função** nova criada por `postgres` — sempre `revoke ... from anon, public` explícito depois de criar
  (mordido 2x no módulo de Reprodução — ver `docs/roadmap-reproducao-equina.md`).
- Named-property shadowing do HTML: um `<input name="submit">` sobrescreve `form.submit()` — use `HTMLFormElement.prototype.submit.call(form)` (mordido no fix do link ABCCC).
- CORS preflight custa caro em request→request diferente de endpoint: prefira 1 RPC que devolve tudo a N chamadas REST separadas quando fizer sync de dados.

## Repos e domínios
- App: `mimba-app/cabanha` → **app.mimba.com.br** (main) / **mimba-hml.pages.dev** (staging, Cloudflare Pages).
- Landing: `mimba-app/mimba-landing` → **mimba.com.br**.
- Deploy do app (main): push → GitHub Pages + `versionar.yml`. Deploy staging: push na branch `staging` → Cloudflare Pages automático.

Memória do projeto (auto-carrega): `MEMORY.md` + arquivos — inclui agora `staging-e-isolamento-de-dados`, `pendencias-pos-apresentacao`, `rls-permissiva-por-perfil` (novos desta sessão).
