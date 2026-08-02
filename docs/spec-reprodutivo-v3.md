# Spec — Reprodutivo v3 (unificação Gestação + Reprodutivo, planejamento por ciclo)

> **Status: rascunho em construção.** Este documento consolida o que o Pedro passou em 2026-08-02 (primeira
> leva de informações soltas). Ainda vai receber mais input antes de virar plano de desenvolvimento — as
> seções marcadas com ❓ são pontos que precisam de decisão/confirmação antes de começar a implementar.

## 1. Problema

Hoje existem **duas telas paralelas e defasadas** — "Reprodutivo" e "Gestação" — cobrindo o mesmo domínio de
forma inconsistente, confundindo o usuário sobre onde fazer o quê. A aba "Gestação" (v2, da sessão de
Reprodução Equina) já é onde cabanheiro/gerente/dono efetivamente planejam o ciclo reprodutivo; a aba
"Reprodutivo" ficou para trás e hoje pede dados que não fazem sentido pro fluxo real (registro de cobertura
com campos desnecessários).

Esta é considerada **a funcionalidade mais estratégica do produto** — a que mais justifica a assinatura,
o "brincar de deus" do dono da cabanha planejando o futuro do plantel. Merece prioridade e um desenho único,
não dois fragmentos.

## 2. Objetivo desta revisão

Unificar tudo em **uma única tela** ("Reprodutivo", substituindo as duas atuais), organizada em torno do
conceito de **Ciclo Reprodutivo** — com uma aba pro ciclo atual (gestações em andamento) e a possibilidade de
já planejar o próximo ciclo, mesmo com éguas ainda prenhas do ciclo atual.

## 3. Conceito central: Ciclo Reprodutivo

- Um ciclo é nomeado pelo período de **cobertura**, não de nascimento — ex.: ciclo **25/26** é quando as
  coberturas acontecem; os nascimentos resultantes caem no ciclo seguinte, **26/27**.
- A tela deve permitir acompanhar o ciclo atual **e** planejar o próximo em paralelo. Uma égua prenha no
  ciclo atual não pode virar "inseminação" (ela está de cria), mas o sistema deve permitir **planejá-la** já
  para o próximo ciclo — quanto antes o planejamento for liberado pro veterinário, melhor pro cliente.
- ❓ **A definir:** como o sistema decide quando um ciclo "vira" o atual (data de corte? ação manual do
  admin encerrando o ciclo anterior — já existe uma funcionalidade de "encerramento de ciclo" da Fase 8 do
  módulo anterior, avaliar se reaproveita)? Quantos ciclos futuros podem coexistir planejados ao mesmo tempo
  (só o próximo, ou N)?

## 4. Ponto de partida do planejamento: o cadastro de Animais

Tudo no planejamento do ciclo nasce do que já está cadastrado na aba Animais — o Reprodutivo não deve
duplicar cadastro, só **completar com informações que são específicas do ciclo** (não do animal em si).

### 4.1 Garanhões (reprodutores)

- Animal cadastrado como reprodutor (macho, >20 meses) aparece disponível no **planejador de ciclo**, não na
  aba Animais.
- No planejador, o admin informa **quantas coberturas daquele garanhão usar naquele ciclo específico** —
  limite regulatório: até **120 coberturas/ano**, ou **240 se o garanhão for registrado como Demérito**.
  O valor pode ser menor que o limite máximo (ex.: cotas vendidas a terceiros reduzem a disponibilidade real
  pra cabanha) — quem decide o número disponível é o admin, por ciclo.
- Conforme coberturas vão sendo lançadas no ciclo, o sistema **desconta** do saldo informado (contador
  regressivo, por garanhão, por ciclo).
- ❓ **A definir:** o sistema deve *bloquear* lançar mais coberturas que o saldo informado, ou só avisar/permitir
  passar do limite com confirmação? O status "Demérito" já existe em algum campo do cadastro do animal, ou
  precisa ser adicionado?

### 4.2 Confirmação de animal (pré-requisito, mexe na aba Animais)

- Hoje não existe conceito de "confirmação" no cadastro de animal. Confirmação = um técnico valida que o
  animal é Crioulo com pedigree confirmado e medidas dentro do padrão.
- Regra: **só pode confirmar a partir de 2 anos de idade**.
- Adicionar ao cadastro/edição de animal (aba Animais) um campo **Confirmado: sim/não**, selecionável pelo
  usuário — respeitando a regra de idade mínima.
- ❓ **A definir:** o app deve *impedir* marcar "confirmado" antes dos 2 anos (validação bloqueante), ou só
  avisar? Confirmação afeta elegibilidade pra reprodução (ex.: só confirmados entram no planejador) ou é só
  um dado informativo por enquanto? Isso conecta com o item já registrado no `ROADMAP.md` ("Campo 'situação'
  e 'data de confirmação' no cadastro de novos animais") — tratar como o mesmo item, não duplicar.

### 4.3 Éguas — três origens possíveis no planejador

O planejador de ciclo precisa aceitar égua reprodutora de três origens, todas configuradas **no menu de
Reprodutivo, por ciclo** — nunca fixo no cadastro do animal, porque isso muda ciclo a ciclo:

1. **Égua de cria já cadastrada na cabanha** (aba Animais) — no planejador, o admin escolhe se ela será
   reprodutora *naquele ciclo*.
2. **Égua receptora** — égua que recebe embrião de outra (fêmea doadora + sêmen de outro garanhão), comum em
   TE (transferência de embrião). Cadastro pelo **SBB** da receptora, associando-a ao ciclo.
3. **Cobertura comprada** — registro de cobertura adquirida (hoje sem sentido no cadastro atual, que pede
   dados errados). Deve puxar os dados **do SBB**, não formulário manual solto.

Resumo do fluxo: ao iniciar/abrir o planejador de um ciclo, o sistema carrega o que já existe cadastrado na
cabanha (garanhões, éguas de cria) e oferece **completar** com o que falta (coberturas compradas, receptoras)
— todas essas opções ficam disponíveis pra seleção dentro do mesmo fluxo de planejamento.

## 5. Remover / substituir

- **Excluir** a funcionalidade atual de registro de cobertura da aba Reprodutivo (pede campos que não fazem
  sentido pro fluxo real).
- Substituir por lançamento de cobertura **puxando do SBB** (garanhão e, quando aplicável, égua/receptora).

## 6. Fora de escopo desta rodada (registrado pra não perder, não construir agora)

- **Marketplace de coberturas entre cabanhas**: um usuário lançar a venda de uma cobertura e ela cair, para a
  cabanha compradora, como um aceite + cadastro da quantidade comprada — via integração entre tenants dentro
  do próprio Mimba. Ideia registrada, **não entra nesta rodada**.

## 7. Decisões (respondidas em 2026-08-02)

1. **Transição de ciclo**: **data de corte fixa**, não ação manual. Ciclo reprodutivo vai sempre de
   **julho a junho**, nomeado pelo ano da cobertura → ex. ciclo **25/26** (coberturas jul/25–jun/26) gera as
   crias do ciclo **26/27**. A troca de "atual" é automática por data: em qualquer momento a partir de 1º de
   julho, o ciclo que começa nessa data vira o atual. Exemplo prático (hoje, 2026-08-02): estamos dentro do
   ciclo **26/27** (atual), e o **27/28** já pode estar sendo planejado em paralelo.
2. **Ciclos paralelos**: só **dois** de cada vez — o atual e o próximo. Nunca mais que isso.
3. **Estouro de saldo de coberturas**: **não bloqueia** — só **avisa de forma bem incisiva** (visualmente
   evidente, não um toast discreto) ao ultrapassar o saldo informado pro garanhão naquele ciclo.
4. **Cotas/Demérito**: sem integração externa por enquanto — **o próprio usuário informa manualmente**, no
   cadastro do animal (aba Animais), a quantidade de cotas/coberturas disponíveis daquele garanhão; o
   planejador do ciclo vai consumindo esse saldo conforme lança coberturas.
5. **Confirmação de animal**: **não bloqueia** nada — é só orientação/aviso. Um animal não confirmado (ou
   com menos de 2 anos) pode ser usado no planejamento normalmente; a responsabilidade é do dono da cabanha.
6. **Receptora/cobertura via SBB**: reaproveitar **exatamente** a mesma integração/mecanismo já usado no
   link SBB→ABCCC e na importação de animais por lista de SBB — sem construir uma integração nova.
7. **Dados/telas atuais do "Reprodutivo"**: **arquivar**, não descartar de vez — motivo explícito: garantir
   que, se os sócios identificarem depois que algum recurso importante foi cortado, dá pra recuperar. (Ver
   nota de implementação abaixo — provavelmente renomear/mover tabelas em vez de `DROP`, nunca apagar dado.)

## 8. Levantamento do schema atual (2026-08-02)

Tabelas em `public` (template, clonado em cada `cab_<slug>`) relevantes ao domínio:

| Tabela | O que guarda hoje |
|---|---|
| `fontes_cobertura` | Já é quase exatamente o conceito de "saldo de coberturas por ciclo" pedido na spec: `tipo` (`proprio`/`cota`/`direito_uso`), `garanhao_nome`/`garanhao_sbb`, `tem_rm` (bool), `quantidade_adquirida`, `ciclo` (texto), `vigencia_inicio/fim`, `proprietario_cota`, `percentual_cota`, `status` (`ativa`/`vencida`). |
| `acasalamentos` | Lançamento de cobertura em si: `egua_id`, `fonte_cobertura_id`, `tipo_cobertura`, `ciclo`, `status` (`rascunho`/`em_curso`/`confirmado`/`cancelado`), aprovação (`aprovado_em/por`). |
| `tentativas` | Tentativas dentro de um acasalamento (inseminação/diagnóstico), com veterinário e resultado. |
| `gestacoes` | Uma gestação confirmada, ligada a um `acasalamento_id` — datas de cobertura/confirmação/parto, protocolo aplicado. |
| `protocolos_reproducao` | Protocolos reutilizáveis (etapas em jsonb). |
| `coberturas_negociadas` | **Marketplace entre cabanhas já existe**: `fonte_cobertura_id`, `quantidade`, `comprador_tenant_id`/`nome`/`contato`, `valor_total`, `parcelas` (jsonb), `status` (`pendente`/`aceito`/`quitado`/...). |
| `coberturas` | Tabela **antiga** (pré-v2), campos soltos (`egua_nome`, `garanhao` texto livre, `sbb_padrillo`, `cria_nome`/`sexo` direto na linha) — é a que a spec pede pra **arquivar** (seção 7, item 7). |
| `animais` | Sem campo de reprodutor/cotas, sem campo de confirmação — só `situacao`, `estagio`, `status_cadastro`, `sexo`, `nasc`, `sbb`. Confirma que os dois campos novos da spec (seção 4.1 e 4.2) realmente não existem ainda. |

RPCs/funções já implementadas:
- `_calc_ciclo_texto(data)` — calcula o texto do ciclo (`"25/26"`) a partir de uma data.
- `encerrar_ciclo_reproducao()` — rotina (chamada por cron, `SECURITY DEFINER`, roda pra **todos os tenants**) que
  a cada virada de ciclo: vence fontes de cobertura própria com saldo (indevidamente, hoje — ver conflito 1
  abaixo), cancela acasalamentos `em_curso` não confirmados, **recria automaticamente** uma fonte de cobertura
  nova pro ciclo seguinte por garanhão (`120` coberturas, ou `150` se `tem_rm=true`), vence cotas/direitos de
  uso com `vigencia_fim` expirada.
- `negociar_cobertura_mimba` / `aceitar_negociacao_cobertura` / `recusar_negociacao_cobertura` /
  `buscar_tenant_para_negociacao` — fluxo completo de venda/aceite de cobertura **entre cabanhas diferentes do
  Mimba**, já funcionando.

### Conflitos com as decisões da seção 7 — resolvidos em 2026-08-02

1. **✅ Corte do ciclo: muda pra julho.** `_calc_ciclo_texto` passa de `month >= 8` pra **`month >= 7`**.
   Como essa função já roda em produção via cron (`encerrar_ciclo_reproducao`) pra todos os tenants, é
   migration simples na função — mas **atenção no dia do deploy**: rodar fora da janela de virada de ciclo
   (não fazer isso em julho de um ano real sem revisar o que já foi criado com o corte antigo).
2. **✅ Investigado — `tem_rm` NÃO tem relação com Demérito.** Confirmado no código-fonte
   (`index.html`, checkbox `#fc-tem-rm`): o rótulo é **"Reserva de material (RM)"**, um conceito de reserva de
   sêmen/material genético — sem nenhuma ligação com a classificação Demérito da ABCCC. É só uma coincidência os
   dois hoje darem bônus parecido (RM libera 150 em vez de 120). **Decisão**: `tem_rm` continua existindo como
   está (não mexe); Demérito vira um **campo novo e independente** em `fontes_cobertura` (ex.: `demerito
   boolean`), que sozinho eleva o teto de 120 pra 240. ❓ Combinação `tem_rm=true` + `demerito=true` ao mesmo
   tempo: soma os dois bônus, ou o maior teto (240) já cobre tudo? Decidir na fase de implementação, não é
   bloqueante pra desenhar o resto.
3. **✅ Marketplace fica exposto.** A tela nova de Reprodutivo vai dar acesso ao fluxo de
   `negociar_cobertura_mimba`/`aceitar_negociacao_cobertura` (venda/compra de cotas entre cabanhas) e à
   marcação automática que ele já habilita — não é preciso construir nada novo de backend pra isso, só desenhar
   a UI que hoje não existe pra esse fluxo.
4. **✅ Herança automática mantida, com edição manual liberada.** `encerrar_ciclo_reproducao` continua recriando
   a fonte de cobertura do ciclo novo herdando do ciclo anterior (comportamento atual, sem mudar a rotina) — o
   admin só precisa poder **editar manualmente** depois (garanhão trocou de cota disponível, ou a cabanha
   adquiriu mais coberturas entre um ciclo e outro). Isso já é possível hoje via `editFonteCobertura` — não
   precisa de mudança de banco, só garantir que a tela nova deixa essa edição visível e óbvia no fluxo do
   planejador (não escondida numa tela separada de "fontes de cobertura" como está hoje).

## 9. Próximos passos deste documento

Spec de requisitos, decisões de produto e levantamento de schema **fechados**. Antes de começar a
implementação, ainda falta:
- Desenhar a tela única (wireframe/fluxo) substituindo as duas abas atuais — incluindo onde entra o fluxo de
  marketplace (item 3 acima), hoje sem UI própria.
- Quebrar em fases de implementação (banco → planejador → integração SBB → UI final → arquivamento de
  `coberturas`).
