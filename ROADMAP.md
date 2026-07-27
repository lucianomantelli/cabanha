# Roadmap — pré-apresentação (quarta-feira, 2026-07-29)

> Levantado numa conversa com o sócio (2026-07-27) após feedback de uso real.
> Objetivo: vencer o máximo desta lista até quarta, na ordem de prioridade abaixo.
> Itens sem prioridade explícita ficam no fim, na ordem em que foram citados.

## Como usar este documento
- Cada item tem checkbox — marcar `[x]` ao concluir.
- "Onde mexe" aponta o ponto de partida no código (todo o frontend é `index.html`, sem framework).
- Ordem de prioridade é a ordem de trabalho recomendada — mas nada impede paralelizar se fizer sentido.

---

## 🔴 Prioridade 1 — Dashboard lento ✅
- [x] **Demora excessiva na tela de dashboard para carregar os dados sobre os animais.**
  Causa raiz medida com dados reais de rede: 12 requisições REST separadas no sync, cada uma
  disparando seu próprio preflight CORS — não era o banco (provado com `EXPLAIN ANALYZE`, poucos
  ms). Resolvido com a RPC `carregar_dados_cabanha` (1 requisição em vez de 12) + skeleton animado
  enquanto carrega. Ver memória `staging-e-isolamento-de-dados` / commits `perf:` na `staging`.

## 🔴 Prioridade 2 — Tela de conta centralizada + onboarding de usuário via Auth ✅
- [x] Tela de conta (modal "Conta", abas Cabanha/Usuários) acessível por um card no rodapé do
  menu lateral — avatar, nome, badge de perfil, botão "Sair" (vermelho, sempre visível) e ícone
  de engrenagem (SVG, estilo shadcn/ui) que abre as configurações.
- [x] Aba "Usuários" solta removida do menu lateral, unificada dentro da Tela de Conta.
- [x] **Convite de usuário corrigido** — nova edge function `convidar-usuario` cria identidade
  real no Supabase Auth + `tenant_memberships` + linha no schema, em vez de só uma linha local.
- [x] Achado extra do `revisor-isolamento` também corrigido: suspender/excluir usuário agora
  revoga o acesso de verdade (`tenant_memberships`), não só a linha local.
- [x] *(Achado maior, registrado para depois — ver memória `rls-permissiva-por-perfil`):* a RLS
  de todas as tabelas de tenant libera escrita a qualquer perfil, não só admin — não corrigido
  nesta rodada, precisa do `arquiteto`.

## 🔴 Prioridade 3 — Importação de animais por lista de códigos SBB ✅
- [x] Botão "📋 Importar por SBB" na página Animais — cola lista ou carrega `.txt`/`.csv`,
  deduplica, cruza contra SBBs já cadastrados, busca cada um novo na ABCCC (`buscar-abccc`, 3 em
  paralelo) com barra de progresso real, e insere tudo num único POST em lote.
- [x] `.xlsx` fica para depois (decisão consciente — evita introduzir a 1ª dependência externa
  do projeto sem necessidade imediata).
- [ ] *(Futuro — não entra nesta rodada):* puxar automaticamente todos os animais da cabanha via
  código/afixo na ABCCC e sugerir quais importar.

## 🔴 Prioridade 4 — Eventos não estão carregando ✅
- [x] Causa da **não-carga** encontrada e corrigida como efeito colateral da Prioridade 1: o
  PostgREST não reconhecia o relacionamento `eventos`→`eventos_animais` nesse schema
  (`PGRST200`). A RPC `carregar_dados_cabanha` monta esse vínculo manualmente em SQL.
- [x] **Confirmado ao vivo na Cabanha Mãe de Deus (2026-07-27)** pelo Pedro: os eventos que
  estavam sumidos voltaram a aparecer. Não era perda de dado na migração — era só o bug de
  leitura acima.

## 🔴 Prioridade 5 — Modal de detalhes do animal quebrado ✅
- [x] Causa do "texto colado" achada: `.ficha-row` (flex, label de largura fixa) estava dentro
  de um grid de 2 colunas, espremendo cada linha pela metade. Trocado por um grid de campos
  independentes.
- [x] Virou **página cheia** (`page-detalhe-animal`), com botão "← Voltar" — mesmo padrão de
  navegação das outras telas, em vez de modal de 900px.
- [x] Bônus: achado e corrigido um bug separado que deixava a seção "Eventos" do detalhe sempre
  vazia (`e.participantes`/`p.nome` vs. `e.animais`/`p.animal` — nomes de campo divergentes).
  Corrigir na origem também resolveu ~9 outros pontos de leitura no app (ranking Reprodutivo,
  busca da aba Eventos, timeline).

## 🔴 Prioridade 6 — Responsividade mobile quebrada ✅
- [x] Causa raiz medida em 375px (não suposição): nada travava `overflow-x` na raiz
  (`html`/`body`), então `.tab-row` (sub-abas tipo Vacinas/Exames/Vermifugação, sem quebra de
  linha nem scroll próprio) forçava a página inteira a crescer (scrollWidth chegava a 572px numa
  tela de 375px). Corrigido: trava `overflow-x:hidden` na raiz + `.tab-row` agora rola dentro de
  si mesma.
- [x] Menu lateral mobile **oculto por padrão**, com botão hamburger fixo que abre um painel
  suspenso (drawer) com overlay — fecha sozinho ao navegar ou tocar fora. Não disputa mais espaço
  fixo com o conteúdo (antes era uma barra de 60px só ícones sempre visível).
- [x] Testado em 375px: as 12 páginas + login sem overflow (`scrollWidth === innerWidth` em
  todas). Desktop confirmado sem regressão.

---

## Sem prioridade explícita (ordem em que foram citados — ficam para depois dos 6 acima)

- [ ] **Tela de Animais (V1) — modernizar usando a tela de Gestações (V2) como referência de padrão.**
- [x] **Remover o recurso "salvar e carregar dados"** do menu lateral inferior esquerdo — resquício
  da versão antiga de importação por planilha, não é mais usado. *(Feito junto da Prioridade 2.)*
- [ ] **Tela de Animais — edição sem confirmação e editável direto na "planilha"** (mistura
  modo leitura e edição ao mesmo tempo). Ideal: visualização em modo leitura + botão "editar" que
  abre um pop-up/tela de edição com confirmação — no mesmo padrão do fluxo "criar novo animal"
  que já existe e funciona bem.
- [ ] **Bug de troca de aba em Gestação:** dados de gestações ativas não carregam na primeira
  visita à aba — só aparecem depois de sair e voltar. Mesmo bug acontece na aba **Medidas**.
- [ ] **Aba de Medidas — layout precisa de melhoria visual** (além do bug acima).
- [ ] **Link do SBB para a ABCCC:** em Reprodutivo → Matrizes, clicar no SBB do animal deveria
  abrir a ABCCC com a busca já preenchida com o código. Aplicar o mesmo comportamento na busca de
  animais da aba Animais.
- [ ] **Campo "situação" e "data de confirmação"** no cadastro de novos animais — hoje ausente.
  Motivação: no futuro será preciso reportar quantidade de animais não confirmados (regra ABCCC:
  confirmação exige idade mínima de 2 anos).
- [ ] **Buscar as medidas da confirmação** (altura, tórax, canela) direto da ABCCC para
  pré-preencher o cadastro de novo animal.
- [x] **Login concluído:** e-mail lembrado (localStorage, nunca a senha), botão de mostrar/ocultar
  senha, e sessão persistida entre recarregamentos — guarda `access_token`/`refresh_token`, renova
  silenciosamente no boot se expirado, cai de volta no login (com e-mail lembrado) se falhar.
  "Sair" limpa a sessão persistida de propósito.

---

## Notas de contexto (não fazer sem entender antes)
- Todo o frontend é um `index.html` único sem framework/bundler — qualquer mudança de UI é direto
  nesse arquivo. Deploy: só commitar o `index.html`, push na `main` (skill `deploy`).
- Mudanças que tocam **auth, provisionamento, RLS ou queries cross-schema** (caso do item de
  Prioridade 2 sobre convite de usuário) precisam passar pelo subagente `revisor-isolamento`
  antes de mergear — é regra do projeto (`CLAUDE.md`).
- Para investigar o bug de Eventos (Prioridade 4), começar pelo banco (schema
  `cab_mae_de_deus`) antes de mexer em código — precisa confirmar se é perda de dado ou bug de
  leitura antes de decidir o que corrigir.
