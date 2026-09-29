# F-006 — Seleção de dívidas

> Roadmap: [F-006](../../ROADMAP.md) · Fase 2 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 1, §5 · `ARCHITECTURE.md` §5.1–§5.3, §9.1, §9.2 · inventário P6, P7

## Objetivo

Para os devedores aprovados na F-005, obter as dívidas aptas **para a fase da execução**, atualizá-las
em lote quando o parâmetro manda e validá-las. Sem bypass, a dívida que não cumpre as condições da
fase (notificação válida; protesto `PROTESTADO` ou dispensa) nem retorna da consulta. É aqui que o
gate CNJ 547 começa a valer na prática.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Consultar as dívidas pela API do `divida` com critérios do job + filtros de condição da fase | `ARCHITECTURE.md` §5.1, §9.1 |
| RF-02 | Fase `PROTESTO` (sem bypass de notificação): só dívidas com notificação válida e sem protesto ativo | `ARCHITECTURE.md` §5.1 |
| RF-03 | Fase `AJUIZAMENTO` (sem bypass): só dívidas com notificação válida **e** (protesto `PROTESTADO` **ou** dispensa **ou** negativada) | `ARCHITECTURE.md` §5.1; decisão de 28/09/2026 |
| RF-04 | Flags de bypass omitem da consulta o filtro correspondente (notificação e/ou protesto) | `PROJECT.md` §5 |
| RF-06 | Em todas as fases: excluir dívida já ajuizada e protesto em `PAGO` ou `ENVIADO_PARA_CARTORIO` | `ARCHITECTURE.md` §5.3 (R1) |
| RF-07 | Valor mínimo de protesto como critério do job (vazio = sem limite) | `ARCHITECTURE.md` §5.3 (R3) |
| RF-08 | Atualizar as dívidas **em lote** quando `INTEGRACAO_ATUALIZA_TODA_DIVIDA` estiver habilitado, tratando o resultado por dívida | `ARCHITECTURE.md` §5.2 |
| RF-09 | Dívida atualizada ⇒ todos os validadores; não atualizada ⇒ só os não cobertos pela consulta | `ARCHITECTURE.md` §5.2 (R5) |
| RF-10 | Bloqueios: CDA prescrita, já ajuizada (salvo extinção sem mérito), exigibilidade suspensa, bloqueio de cobrança da dívida | `PROJECT.md` §3 item 1 |
| RF-11 | Cada descarte registra motivo tipado no item e no sumário; reserva liberada | F-003 RN-05 |
| RF-12 | Execução com **lista explícita** de dívidas (critérios ou CSV): conciliar a lista informada com o que a consulta devolveu; cada dívida informada **ausente** vira item `DESCARTADO` na etapa `QUALIFICACAO_DIVIDA`, com o motivo do primeiro critério que ela não cumpre (ou "dívida não encontrada" / "devedor descartado: <motivo>"), visível no sumário | paridade `divida` C6, `cobranca` legado C17; decisão de 28/09/2026 |
| RF-13 | Na fase `NOTIFICACAO`, dívida que já está em kit de notificação `EM_REGUA` sai `DESCARTADO` com motivo "já em régua" (descarte por regra, verificado neste serviço) — impede dois ciclos simultâneos, já que a dívida notificada fica sem reserva | decisão de 28/09/2026; `ARCHITECTURE.md` §5.1 |
| RNF-01 | Falha na atualização de uma dívida não derruba o bloco nem descarta a dívida | `ARCHITECTURE.md` §5.2 |

## Regras de negócio

- **RN-01** — Sem bypass, dívida inapta para a fase **não é selecionada** (filtro na consulta), em vez de selecionada e descartada.
- **RN-02** — Cada validador declara se é "coberto pela consulta". Exemplos: situação da dívida (coberto), bloqueio de cobrança (não coberto).
- **RN-03** — Notificação com aviso válido vale para sempre; a consulta só exige que exista (28/09/2026).
- **RN-04** — Dívida cuja atualização falhou **não é descartada aqui**: o item fica com status `ERRO`
  e motivo "falha na atualização" e segue para o agrupamento, onde `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` decide (F-011 RN-06).
  Assim o procurador pode reprocessar o kit quando a causa for resolvida (ex.: serviço de atualização fora do ar).
  Sem atualização, a dívida passa pelos validadores não cobertos pela consulta (RF-09); descarte por regra continua valendo.
- **RN-06** — Nenhuma dívida informada some sem motivo: toda dívida da lista explícita termina em kit ou em item `DESCARTADO` com motivo (RF-12; F-016 RF-12). O motivo é o do primeiro critério não cumprido, na ordem do catálogo de validadores, incluindo as condições de fase (sem notificação válida; sem protesto `PROTESTADO` ou dispensa). A dívida ausente não chega a ser reservada. Sem lista explícita não há conciliação: sem expectativa, não há confirmação visual (mesma regra dos devedores, F-005 RN-09).
- **RN-05** — Os parâmetros `SINCRONIZAR_DIVIDA_COM_INTEGRACAO_AO_PREPARAR_COBRANCA` e `SINCRONIZAR_DIVIDA_COMPLETA_AO_ATUALIZAR_VALORES` não são lidos.

## Critérios de aceite

### CA-01 — Ajuizamento sem bypass filtra condições (RF-03, RN-01)
- **Dado** três dívidas de um devedor: A notificada e `PROTESTADO`; B notificada sem protesto; C sem notificação
- **Quando** uma execução de ajuizamento sem bypass consulta
- **Então** só A retorna

### CA-02 — Ajuizamento com bypass de protesto (RF-04)
- **Dado** as mesmas três dívidas
- **Quando** a execução tem `bypassProtesto`
- **Então** retornam A e B; C não retorna

### CA-03 — Bypass total (RF-04)
- **Dado** as mesmas três dívidas
- **Quando** a execução tem `bypassNotificacao` e `bypassProtesto`
- **Então** retornam A, B e C

### CA-04 — Protesto sem bypass (RF-02)
- **Dado** B notificada sem protesto e C sem notificação
- **Quando** uma execução de protesto consulta
- **Então** só B retorna

### CA-05 — Notificação não expira (RN-03)
- **Dado** uma dívida notificada há 5 anos
- **Quando** uma execução de protesto sem bypass consulta
- **Então** a dívida retorna

### CA-06 — Protesto pago ou em cartório (RF-06)
- **Dado** dívida com protesto em `PAGO` e outra em `ENVIADO_PARA_CARTORIO`
- **Quando** qualquer execução consulta
- **Então** nenhuma das duas retorna, mesmo com bypass

### CA-07 — Atualização em lote parcial (RF-08, RNF-01, RN-04)
- **Dado** `INTEGRACAO_ATUALIZA_TODA_DIVIDA` habilitado e um bloco de 500 dívidas, das quais 3 falham na atualização
- **Quando** a etapa roda
- **Então** as 500 seguem; as 3 ficam com status `ERRO` e motivo "falha na atualização", sem descarte nesta etapa

### CA-08 — Validação conforme atualização (RF-09, RN-02)
- **Dado** uma dívida não atualizada
- **Quando** é validada
- **Então** só validadores não cobertos pela consulta executam; se atualizada, todos executam

### CA-09 — Bloqueios (RF-10, RF-11)
- **Dado** dívidas prescrita, já ajuizada, com exigibilidade suspensa e com bloqueio de cobrança
- **Quando** são validadas (após atualização)
- **Então** cada uma é descartada com o motivo correspondente no sumário

### CA-10 — Consulta leva critérios e filtros de fase (RF-01)
- **Dado** uma execução de protesto sem bypass com critérios de valor e tributo
- **Quando** a consulta ao `divida` é montada
- **Então** a requisição contém os critérios do job, a fase e `exigirNotificacao = true`

### CA-11 — Valor mínimo de protesto (RF-07)
- **Dado** um job de protesto com valor mínimo de R$ 500 e dívidas de R$ 300 e R$ 800
- **Quando** a consulta é feita
- **Então** só a dívida de R$ 800 retorna; com o critério vazio, retornam as duas

### CA-15 — Dívida já em régua não entra em 2º ciclo (RF-13)
- **Dado** a CDA 101 já notificada, sem reserva, num kit de notificação `EM_REGUA`
- **Quando** outra execução de notificação a devolve na consulta
- **Então** a 101 sai `DESCARTADO` com motivo "já em régua" e não entra em kit novo; quando o kit anterior termina `REGUA_CONCLUIDA`, ela volta a ser elegível

### CA-12 — Dívida informada filtrada pela consulta (RF-12, RN-06)
- **Dado** uma execução de ajuizamento sem bypass com lista explícita das CDAs 101, 102 e 103, sendo a 103 sem notificação válida
- **Quando** a consulta devolve só 101 e 102
- **Então** a 103 vira item `DESCARTADO` com motivo "sem notificação válida" e aparece no sumário

### CA-13 — Dívida informada inexistente ou de devedor descartado (RF-12)
- **Dado** uma lista explícita com a CDA 104, que não existe no tenant, e a CDA 105, cujo devedor foi descartado na F-005 por CPF/CNPJ inválido
- **Quando** a conciliação roda
- **Então** a 104 fica `DESCARTADO` com motivo "dívida não encontrada" e a 105 com "devedor descartado: CPF/CNPJ inválido"

### CA-14 — Sem lista explícita não há conciliação (RF-12)
- **Dado** uma execução só por critérios
- **Quando** a consulta devolve as dívidas
- **Então** nenhuma dívida fora do resultado vira item

## Fora de escopo

- Reconfirmação do gate no domínio antes do kit (F-010).
- Agrupamento (F-011).

## Dependências e pendências

- **Features:** F-005.
- **Externas:** E-01 — API de seleção com filtros de fase (inclui a busca por números, sem filtros de fase, para a conciliação do RF-12); E-02 — índice do `divida` com notificação, dispensa e categoria do protesto; E-03 — atualização em lote.
- **Em aberto:** nenhuma.
