# F-003 — Execução, itens e reserva

> Roadmap: [F-003](../../ROADMAP.md) · Fase 1 · Status: PENDING
> Fontes: `PROJECT.md` §3 (itens 7, 8), §5 · `ARCHITECTURE.md` §3, §4, §8.2 · inventário P1, P8, P18, P27

## Objetivo

Representar cada disparo de cobrança como uma **execução de uma fase** (notificação, protesto ou
ajuizamento), com snapshot imutável dos critérios e das flags de bypass, os itens (dívidas) que ela
processa e a garantia de que uma dívida está em no máximo uma execução ativa. É a base sobre a qual
seleção, régua, gate e geração de cobrança operam.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Criar execução com fase (`NOTIFICACAO`, `PROTESTO`, `AJUIZAMENTO`) e origem (`AGENDADA`, `MANUAL_CRITERIOS`, `MANUAL_ARQUIVO`, `EXCEPCIONAL`) | `ARCHITECTURE.md` §3, §4 |
| RF-02 | Guardar snapshot imutável dos critérios recebidos (incl. flags `bypassNotificacao`/`bypassProtesto`, régua, estratégia de empacotamento) com hash, autor e data | `PROJECT.md` §5; `ARCHITECTURE.md` §8.2 |
| RF-03 | Marcar a execução como **prioritária** quando os critérios trazem lista de documentos de devedor ou de números de dívida, e sempre na origem `EXCEPCIONAL` | `PROJECT.md` §3 item 7 |
| RF-04 | Registrar os itens da execução (devedor, dívida) com **etapa** (`QUALIFICACAO_DEVEDOR`, `QUALIFICACAO_DIVIDA`, `REGUA`, `AGRUPAMENTO`, `GERACAO_COBRANCA`), **status** (`SUCESSO`, `ERRO`, `AVISO`, `DESCARTADO`) e motivo — `ERRO`/`AVISO` não tiram o item da execução (decisão no agrupamento, F-011 RN-06) | `ARCHITECTURE.md` §3, §4 |
| RF-05 | Garantir que uma dívida tenha no máximo um item ativo em todas as execuções (reserva) | inventário P8; `ARCHITECTURE.md` §3 |
| RF-06 | Expor `POST /execucoes` para disparo agendado e manual por critérios | `ARCHITECTURE.md` §4 |
| RF-07 | Expor `GET /execucoes/{id}/situacao` compatível com o polling do `agendador` | `ARCHITECTURE.md` §4 |
| RF-08 | Parar execuções por `jobId` e encerrar as execuções ativas do tenant | inventário P27 |
| RF-09 | Exigir privilégio de disparo; exigir privilégio de bypass quando a execução **manual** traz flag de bypass | `PROJECT.md` §5; `ARCHITECTURE.md` §8.2 |
| RF-10 | Manter contadores de progresso da execução (selecionados, descartados por motivo, notificados, encaminhados, com erro, com aviso) | inventário P18 |
| RF-11 | Reprocessamento (F-016) reabre a execução original: `CONCLUIDA_COM_FALHAS` → `PROCESSANDO`, com o mesmo snapshot de critérios | `ARCHITECTURE.md` §7.1 (A22) |
| RNF-01 | Isolamento por tenant em todas as consultas e escritas | ADR segurança 0001 |

## Regras de negócio

- **RN-01** — O snapshot dos critérios é imutável: alterar o job no `agendador` depois não altera execuções existentes.
- **RN-02** — Bypass não exige justificativa; exige privilégio. Na execução agendada, o privilégio é checado ao salvar o job (fora deste serviço, E-05); na manual, no request.
- **RN-03** — Colisão de reserva (dívida já em outra execução ativa) deixa o item `DESCARTADO` com motivo "dívida em outra execução", não é erro da execução. Não é `AVISO`: a dívida não pertence a esta execução e não deve chegar ao agrupamento.
- **RN-04** — Exceção à reserva: execução de protesto/ajuizamento com bypass, ou excepcional, toma a dívida de uma execução de **notificação** ativa (a retirada do item da régua é tratada na F-008).
- **RN-05** — A reserva vale **enquanto a execução estiver ativa**: é liberada quando o item sai da execução (`DESCARTADO`) ou quando a execução termina — inclusive para as dívidas de kit `INVALIDO`, que a partir daí podem ser pegas por outra execução (por isso o reprocessamento revalida). Na fase de **notificação**, a reserva é liberada também no **primeiro aviso válido** do item: a dívida segue na régua em paralelo, livre para protesto/ajuizamento (28/09/2026, `ARCHITECTURE.md` §6.2).
- **RN-06** — Significado dos status (A21): **`SUCESSO`** = a etapa concluiu; **`ERRO`** = algo deu errado, mas pode ser
  corrigido; **`AVISO`** = não há o que fazer, é só um aviso; **`DESCARTADO`** = saiu da execução, com motivo.

## Critérios de aceite

### CA-01 — Disparo agendado (RF-01, RF-02, RF-06)
- **Dado** critérios de um job de ajuizamento enviados pelo `agendador`
- **Quando** `POST /execucoes` é chamado
- **Então** a execução é criada com fase `AJUIZAMENTO`, origem `AGENDADA`, snapshot e hash dos critérios, e o id é devolvido

### CA-02 — Snapshot imutável (RN-01)
- **Dado** uma execução criada a partir do job J
- **Quando** os critérios de J mudam no `agendador`
- **Então** o snapshot da execução permanece igual e o hash confere

### CA-03 — Prioridade por lista explícita (RF-03)
- **Dado** critérios com uma lista de números de dívida
- **Quando** a execução é criada
- **Então** ela é marcada prioritária

### CA-04 — Sem lista, não prioritária (RF-03)
- **Dado** critérios só com filtros (valores, datas, tributo)
- **Quando** a execução é criada
- **Então** ela não é prioritária

### CA-05 — Reserva única concorrente (RF-05)
- **Dado** duas execuções ativas processando a mesma dívida ao mesmo tempo
- **Quando** ambas tentam criar o item
- **Então** só uma cria o item ativo; na outra, o item fica `DESCARTADO` com motivo "dívida em outra execução"

### CA-06 — Bypass/excepcional toma dívida em régua (RN-04)
- **Dado** uma dívida com item ativo numa execução de notificação
- **Quando** uma execução de ajuizamento com bypass seleciona essa dívida
- **Então** a reserva passa para a execução de ajuizamento

### CA-07 — Bypass manual sem privilégio (RF-09)
- **Dado** um usuário com privilégio de disparo e sem privilégio de bypass
- **Quando** dispara execução manual com flag de bypass
- **Então** recebe 403 e nada é criado

### CA-08 — Situação para o agendador (RF-07)
- **Dado** uma execução concluída
- **Quando** o `agendador` consulta `GET /execucoes/{id}/situacao`
- **Então** a resposta informa conclusão no formato que o polling espera

### CA-09 — Parar por job (RF-08)
- **Dado** uma execução em andamento do job J
- **Quando** é solicitada a parada por `jobId` J
- **Então** nenhum bloco novo é processado e a execução termina como parada

### CA-10 — Isolamento de tenant (RNF-01)
- **Dado** execuções dos tenants A e B
- **Quando** um usuário do tenant A consulta execuções
- **Então** só vê as do tenant A

### CA-11 — Item com etapa, situação e motivo (RF-04)
- **Dado** uma execução com um item descartado na validação
- **Quando** o item é consultado
- **Então** ele traz devedor, dívida, etapa em que parou, situação `DESCARTADO` e o motivo tipado

### CA-13 — Reprocessamento reabre a execução (RF-11)
- **Dado** uma execução `CONCLUIDA_COM_FALHAS`
- **Quando** o procurador reprocessa um devedor ou um kit dela (F-016)
- **Então** a mesma execução volta a `PROCESSANDO`, com o mesmo snapshot de critérios, sem criar execução nova

### CA-12 — Contadores de progresso (RF-10)
- **Dado** uma execução com 10 itens selecionados, dos quais 3 descartados por motivos diferentes e 7 encaminhados
- **Quando** a execução é consultada
- **Então** os contadores informam 10 selecionados, 3 descartados agrupados por motivo e 7 encaminhados, coerentes com a situação dos itens

## Fora de escopo

- Processamento das etapas (F-004 em diante).
- Origem `MANUAL_ARQUIVO` (F-014) e `EXCEPCIONAL` (F-015) — o modelo as suporta; os fluxos ficam nessas features.
- Telas (E-09).

## Dependências e pendências

- **Features:** F-001, F-002.
- **Externas:** E-05 — contrato dos critérios com flags e checagem de privilégio ao salvar o job; E-09 — telas.
- **Em aberto:** nenhuma.
