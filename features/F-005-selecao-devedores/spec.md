# F-005 — Seleção de devedores

> Roadmap: [F-005](../../ROADMAP.md) · Fase 2 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 1 · `ARCHITECTURE.md` §5.1, §5.2, §5.3 (R4), §9.1 · inventário P4, P5 (`divida` C5, `cobranca` legado C7)

## Objetivo

Primeira etapa de toda execução: obter, pela API do `divida`, os devedores que atendem aos
critérios da execução, atualizá-los (quando os critérios pedem) e validá-los, descartando com motivo
registrado quem não pode seguir. Os devedores aprovados alimentam a seleção de dívidas (F-006).

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Buscar devedores pela API de seleção do `divida`, paginada por cursor, com os critérios da execução | `ARCHITECTURE.md` §9.1 |
| RF-02 | Gravar os itens selecionados em lote, por bloco, sem ler o mesmo devedor mais de uma vez no bloco | `ARCHITECTURE.md` §12 |
| RF-03 | Atualizar o devedor via `pessoa` somente quando os critérios da execução trouxerem `higienizar` ou `enriquecer` ligado; `enriquecer` inclui a consulta à Receita Federal | `ARCHITECTURE.md` §5.2 · `divida` `CriteriosSelecaoCobranca`; decisão de 27/09/2026 |
| RF-04 | Descartar devedor com CPF/CNPJ inválido (bloqueio fixo) | `PROJECT.md` §3 item 1 |
| RF-05 | Descartar devedor com bloqueio de cobrança cadastrado no `divida` | `ARCHITECTURE.md` §5.3 (R4) |
| RF-06 | Registrar em cada descarte o motivo tipado, visível no sumário da execução | `ARCHITECTURE.md` §5.3 |
| RF-07 | Liberar a reserva das dívidas do devedor descartado | F-003 RN-05 |
| RF-08 | Nas fases `PROTESTO` e `AJUIZAMENTO`, quando `VALIDAR_ENDERECO_CONSISTENTE_GERACAO_KIT_AJUIZAMENTO` estiver habilitado, descartar devedor sem endereço principal, sem município no endereço principal ou com endereço inválido para citação quando não marcado como passível de ajuizamento | inventário `divida` C5, `cobranca` legado C7; decisão de 26/09/2026 |
| RF-09 | Aplicar os critérios de devedor do job — tipo de pessoa (PF/PJ), valor mínimo ajuizado e situação do devedor PJ (ATIVO/INATIVO/AMBOS, inclusive por raiz de CNPJ) — como filtros da consulta ao `divida` | inventário `divida` C5; `ARCHITECTURE.md` §9.1 |
| RF-10 | Execução com **lista explícita** de devedores (documentos nos critérios): conciliar a lista com o que a consulta devolveu; cada devedor informado **ausente** vira item `DESCARTADO` na etapa `QUALIFICACAO_DEVEDOR`, com o motivo — "devedor não encontrado", critério de devedor não atendido (tipo de pessoa, valor mínimo ajuizado, situação PJ) ou "sem dívidas aptas para a fase" — visível no sumário | paridade `cobranca` legado C7; decisão de 28/09/2026 |
| RNF-01 | Nenhuma chamada remota por devedor além da atualização (quando pedida nos critérios) e da consulta de bloqueio | `ARCHITECTURE.md` §12 |

## Regras de negócio

- **RN-01** — CPF/CNPJ inválido é bloqueio fixo para protesto e ajuizamento; não há parâmetro que o desligue. A origem `EXCEPCIONAL` não passa por esta etapa.
- **RN-02** — Devedor com bloqueio de cobrança não segue; o CRUD do bloqueio pertence ao `divida`.
- **RN-03** — Validação completa após atualização; sem atualização, só as validações que a consulta ao `divida` não garante (ver F-006 RN-02 para o catálogo).
- **RN-04** — A validação de endereço é **parametrizada** (mantém o comportamento atual do `divida`) e **não se aplica** à fase `NOTIFICACAO` (a régua usa telefone/e-mail) nem ao `EXCEPCIONAL`.
- **RN-05** — Os critérios de devedor (tipo de pessoa, valor mínimo ajuizado, situação PJ) são filtros da consulta; os validadores correspondentes são "cobertos pela consulta" e só executam quando o devedor foi atualizado.
- **RN-06** — Com `DEVE_GERAR_DEMANDA_CONFERENCIA_ENDERECO_DEVEDOR_INVALIDO` ligado, o endereço inválido **não descarta**: o devedor fica `ERRO` aguardando conferência e não segue para as dívidas (F-016 RF-06, Q-09). A abertura da demanda é da F-016.
- **RN-07** — A atualização do devedor **não é parâmetro do `admin`**: é escolha de cada execução, pelas flags
  `higienizar` e `enriquecer` dos critérios (snapshot da execução, F-003). Qualquer uma ligada dispara a chamada;
  `enriquecer` segue como argumento da chamada ao `pessoa`.
- **RN-09** — A confirmação visual é para quem tem **expectativa**: só execução com lista explícita concilia. Sem lista (seleção só por critérios), devedor fora do resultado não vira item — o procurador não tinha uma lista do que esperava ver selecionado. Devedor informado que é selecionado e depois descartado já aparece pelo RF-06. Mesma regra das dívidas (F-006 RF-12).
- **RN-08** — Falha na atualização do devedor (ex.: `pessoa` fora do ar) **para a esteira do devedor**: o devedor fica
  `ERRO` na etapa `QUALIFICACAO_DEVEDOR`, com motivo "falha na atualização do devedor"; suas dívidas não são consultadas
  e nenhum kit é gerado para ele. O erro é do devedor, não do kit: o reprocessamento do devedor (F-016) recomeça do mesmo
  ponto, sem desfazer kits. Decidido em 28/09/2026 (substitui a regra de 27/09).

## Critérios de aceite

### CA-01 — Paginação por cursor (RF-01, RF-02)
- **Dado** critérios que retornam 1.200 devedores com bloco de 500
- **Quando** a etapa roda
- **Então** 3 páginas são lidas por cursor e 3 blocos são publicados, com os itens gravados em lote

### CA-02 — Seleção vazia (RF-01)
- **Dado** critérios que não retornam devedores
- **Quando** a etapa roda
- **Então** a execução termina como concluída, com zero itens

### CA-03 — CPF/CNPJ inválido (RF-04, RF-06, RN-01)
- **Dado** um devedor com CNPJ de dígito verificador inválido
- **Quando** é validado
- **Então** é descartado com motivo "CPF/CNPJ inválido" e suas dívidas não seguem

### CA-04 — Devedor bloqueado (RF-05, RF-06, RN-02)
- **Dado** um devedor com bloqueio de cobrança ativo no `divida`
- **Quando** é validado
- **Então** é descartado com motivo "bloqueio de cobrança" visível no sumário

### CA-05 — Atualização desligada (RF-03, RN-07)
- **Dado** critérios com `higienizar` e `enriquecer` desligados
- **Quando** a etapa roda
- **Então** nenhuma chamada de atualização é feita ao `pessoa`

### CA-06 — Atualização ligada (RF-03, RN-03)
- **Dado** critérios com `higienizar` ligado e um devedor cujo endereço muda na atualização
- **Quando** a etapa roda
- **Então** a validação usa o dado atualizado

### CA-13 — Enriquecimento pela Receita (RF-03, RN-07)
- **Dado** critérios com `enriquecer` ligado
- **Quando** a etapa atualiza o devedor
- **Então** a chamada ao `pessoa` segue com `enriquecer=true`; com só `higienizar` ligado, segue com `enriquecer=false`

### CA-07 — Reserva liberada (RF-07)
- **Dado** um devedor descartado
- **Quando** o descarte é gravado
- **Então** as reservas das suas dívidas nesta execução são liberadas

### CA-08 — Endereço inconsistente em ajuizamento (RF-08, RN-04)
- **Dado** o parâmetro de endereço habilitado e um devedor sem município no endereço principal
- **Quando** uma execução de ajuizamento valida o devedor
- **Então** ele é descartado com motivo "endereço inconsistente"

### CA-09 — Endereço com parâmetro desligado (RF-08)
- **Dado** o parâmetro de endereço desabilitado e o mesmo devedor
- **Quando** uma execução de ajuizamento valida o devedor
- **Então** ele segue sem descarte por endereço

### CA-10 — Endereço não vale na notificação (RN-04)
- **Dado** o parâmetro de endereço habilitado e um devedor sem endereço principal
- **Quando** uma execução de notificação valida o devedor
- **Então** ele segue sem descarte por endereço

### CA-11 — Critérios de devedor na consulta (RF-09, RN-05)
- **Dado** um job de ajuizamento com tipo de pessoa PJ, valor mínimo ajuizado de R$ 10.000 e situação PJ ATIVO
- **Quando** a consulta ao `divida` é montada
- **Então** os três critérios seguem como filtros; devedor PF, abaixo do valor ou PJ inativo não retorna

### CA-12 — Chamadas remotas limitadas (RNF-01)
- **Dado** um bloco de 500 devedores com `higienizar` e `enriquecer` desligados e stubs WireMock
- **Quando** o bloco é processado
- **Então** ocorre 1 chamada de página à API de seleção e 1 consulta de bloqueio por devedor; nenhuma chamada ao `pessoa` e nenhuma outra chamada remota por devedor

### CA-14 — Falha na atualização do devedor (RF-03, RN-08)
- **Dado** critérios com `higienizar` ligado e o `pessoa` indisponível
- **Quando** a etapa tenta atualizar o devedor
- **Então** o devedor fica `ERRO` na qualificação de devedor com motivo "falha na atualização do devedor", suas dívidas não são consultadas e nenhum kit é gerado para ele

### CA-15 — Devedor informado não selecionado (RF-10, RN-09)
- **Dado** uma execução com a lista dos documentos A, B e C, sendo que B não tem dívida apta para a fase e C não existe no tenant
- **Quando** a consulta devolve só A
- **Então** B vira item `DESCARTADO` com motivo "sem dívidas aptas para a fase" e C com "devedor não encontrado", ambos no sumário

### CA-16 — Devedor informado fora do critério (RF-10)
- **Dado** uma execução com lista de documentos e critério tipo de pessoa = PJ, e um documento de PF na lista
- **Quando** a conciliação roda
- **Então** o devedor PF vira item `DESCARTADO` com motivo "critério de devedor não atendido: tipo de pessoa"

### CA-17 — Sem lista não há conciliação (RN-09)
- **Dado** uma execução só por critérios
- **Quando** a consulta devolve os devedores
- **Então** nenhum devedor fora do resultado vira item

## Fora de escopo

- Seleção e validação das dívidas (F-006).
- CRUD de bloqueio de cobrança (fica no `divida`).
- Demanda de conferência de endereço inválido (F-016, RF-06).

## Dependências e pendências

- **Features:** F-004.
- **Externas:** E-01 — API de seleção paginada no `divida` (integração real), incluindo os filtros de critérios de devedor (RF-09) e a busca de devedores por documento **sem** filtros, para a conciliação (RF-10).
- **Em aberto:** nenhuma (Q-05 decidida em 27/09/2026: flags nos critérios, não parâmetro). As flags `higienizar`/`enriquecer` precisam constar nos critérios do job no `agendador` (E-05) — já existem nos critérios do legado.
