# Roadmap — pré-apresentação (quarta-feira, 2026-07-29)

> Levantado numa conversa com o sócio (2026-07-27) após feedback de uso real.
> Objetivo: vencer o máximo desta lista até quarta, na ordem de prioridade abaixo.
> Itens sem prioridade explícita ficam no fim, na ordem em que foram citados.

## Como usar este documento
- Cada item tem checkbox — marcar `[x]` ao concluir.
- "Onde mexe" aponta o ponto de partida no código (todo o frontend é `index.html`, sem framework).
- Ordem de prioridade é a ordem de trabalho recomendada — mas nada impede paralelizar se fizer sentido.

---

## 🔴 Prioridade 1 — Dashboard lento
- [ ] **Demora excessiva na tela de dashboard para carregar os dados sobre os animais.**
  Investigar: é 1 query grande, N+1 (várias por animal), ou processamento no cliente após carregar tudo?
  Onde mexe: função de carregamento inicial / `renderDashboard` e o que ela busca via `_supa`.

## 🔴 Prioridade 2 — Tela de conta centralizada + onboarding de usuário via Auth
- [ ] Criar uma **tela de conta** (dados da cabanha: nome, afixo, logo) acessível por um
  card/botão fixo no rodapé do menu lateral esquerdo (foto de perfil, nome, badge do perfil
  de acesso — adm/vet/cab —, opção de sair da conta, botão para abrir as configurações).
- [ ] Remover a aba "usuários" do menu lateral solto e **unificar** dentro dessa tela de conta.
- [ ] **Aproveitar para corrigir o convite de usuário** conforme já registrado no `HANDOFF.md`:
  hoje criar um usuário (vet/cabanheiro) provavelmente não usa o fluxo correto de identidade via
  Supabase Auth (login único, JWT, `tenant_memberships`). Corrigir para que todo usuário novo
  nasça como identidade real no Auth (mesma lógica do admin no provisionamento), não só uma linha
  na tabela `usuarios` do schema.
  Onde mexe: telas/tabela de usuários do app + possivelmente nova edge function
  "convidar-usuario" (cria identidade no Auth + membership), análoga à `provisionar-cabanha`.

## 🔴 Prioridade 3 — Importação de animais por lista de códigos SBB
- [ ] Recurso de **importar lista de códigos SBB** para criar animais em lote, buscando os dados
  direto na busca da ABCCC (reaproveita a função `buscar-abccc` já existente).
- [ ] Aceitar: upload de `.txt`/`.csv`/`.xlsx` simples (1 código por linha), **ou** colar/digitar
  uma lista (um valor por linha) direto num textarea.
- [ ] É pré-requisito para um onboarding bom de cabanha nova.
- [ ] *(Futuro — não entra nesta rodada):* usar o código/afixo da cabanha na ABCCC para acessar a
  área logada de lá, puxar todos os animais cadastrados e sugerir quais importar (com seleção
  pelo administrador).

## 🔴 Prioridade 4 — Eventos não estão carregando
- [ ] Investigar por que a aba de **eventos** não mostra os registros esperados — a cabanha
  Mãe de Deus tinha eventos cadastrados que deveriam aparecer.
- [ ] Checar se os dados **se perderam na migração** para o schema próprio (`cab_mae_de_deus`)
  ou se foi introduzido por alguma alteração de código posterior.
  Onde mexe: conferir a tabela de eventos no schema `cab_mae_de_deus` direto no banco
  (existem os registros?) vs. a query/render da aba Eventos no app.

## 🔴 Prioridade 5 — Modal de detalhes do animal quebrado
- [ ] Ao abrir os detalhes de um animal na lista, o visual está quebrado: textos colados,
  cards com espaçamento excessivo, informações mal organizadas.
- [ ] Trocar de **janela/modal** para **tela cheia** (mesmo padrão de navegação das outras telas).

## 🔴 Prioridade 6 — Responsividade mobile quebrada
- [ ] Logo quebrada em mobile; a página inteira fica **mais larga que a tela**, gerando
  scroll lateral indevido.
- [ ] Repensar o menu lateral em mobile: começar **oculto**, com um botão de atalho que abre em
  painel suspenso (overlay), em vez de disputar espaço fixo com o conteúdo da página.

---

## Sem prioridade explícita (ordem em que foram citados — ficam para depois dos 6 acima)

- [ ] **Tela de Animais (V1) — modernizar usando a tela de Gestações (V2) como referência de padrão.**
- [ ] **Remover o recurso "salvar e carregar dados"** do menu lateral inferior esquerdo — resquício
  da versão antiga de importação por planilha, não é mais usado.
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
- [ ] **Login:** lembrar o e-mail digitado (evitar redigitar sempre), opção de mostrar/ocultar a
  senha digitada, e investigar manter a sessão entre recarregamentos de página (hoje qualquer
  reload derruba o acesso — avaliar persistir o JWT/refresh de forma seragura, ex. localStorage
  com expiração, já que hoje `AUTH_TOKEN` só vive em memória).

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
