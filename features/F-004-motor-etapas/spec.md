# F-004 — Motor de etapas via Kafka

> Roadmap: [F-004](../../ROADMAP.md) · Fase 1 · Status: PENDING
> Fontes: `PROJECT.md` §6, §7 · `ARCHITECTURE.md` §5.5, §10, §12 · inventário §5 (lições do `kitcobranca`)

## Objetivo

Encadear as etapas de uma execução por mensagens Kafka com granularidade de **bloco** (e de **kit**
na geração de cobrança), nunca por dívida, com fila separada para execuções prioritárias. Resolve o
principal problema de desempenho do `kitcobranca` (1 mensagem por etapa por item) e permite que um
procurador ajuíze poucas dívidas sem esperar uma execução grande terminar.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Transição entre etapas da execução por mensagem Kafka, com **um tópico por etapa** (e consumer próprio), para que uma etapa não limite a vazão das outras | `PROJECT.md` §6 · §7 (lição do `kitcobranca`); decisão de 28/09/2026 |
| RF-02 | Uma mensagem por **bloco** nas etapas de seleção, validação, agrupamento e passo da régua | `ARCHITECTURE.md` §5.5 |
| RF-03 | Uma mensagem por **kit** na etapa gerar cobrança | `ARCHITECTURE.md` §5.5 |
| RF-04 | Tópicos `.prioritario` com consumidores próprios; execução prioritária publica neles | `ARCHITECTURE.md` §5.5 |
| RF-05 | Consumo idempotente por (execução, bloco) e por kit | `ARCHITECTURE.md` §10.3 |
| RF-06 | Tamanho do bloco parametrizável por tenant; concorrência dos consumidores configurável por deploy (containers Kafka não variam concorrência por mensagem) | `ARCHITECTURE.md` §5.5 |
| RF-07 | Progresso e métricas da execução atualizados por bloco, não por item | `ARCHITECTURE.md` §5.5 |
| RF-08 | Execução parada não processa blocos novos | F-003 RF-08 |
| RNF-01 | Nenhuma mensagem por dívida; N devedores ⇒ ⌈N/bloco⌉ mensagens de bloco por etapa encadeada + 1 por kit | `ARCHITECTURE.md` §12 |
| RNF-02 | Publicação dentro da transação do estado que a origina (sincronia send × commit) | `ARCHITECTURE.md` A15 |

## Regras de negócio

- **RN-01** — A escolha da fila (padrão × prioritária) é da camada de mensageria, a partir de `Execucao.prioritaria`; o payload não carrega o tópico (ADR comum 0005).
- **RN-02** — Mensagem de bloco de execução parada é consumida e descartada sem efeito.
- **RN-03** — Reentrega da mesma mensagem não repete efeito já aplicado.

## Critérios de aceite

### CA-01 — Granularidade por bloco (RF-02, RNF-01)
- **Dado** uma execução que seleciona 5.000 devedores com bloco de 500
- **Quando** a etapa de seleção termina de paginar
- **Então** são publicadas 10 mensagens de bloco, nenhuma por dívida

### CA-02 — Granularidade por kit (RF-03)
- **Dado** um bloco agrupado em 12 kits
- **Quando** a etapa de agrupamento conclui
- **Então** são publicadas 12 mensagens de gerar cobrança

### CA-03 — Fila prioritária (RF-04, RN-01)
- **Dado** uma execução padrão com 50 blocos pendentes e uma execução prioritária com 1 bloco
- **Quando** a prioritária é disparada
- **Então** seu bloco é consumido pelos consumidores prioritários sem esperar os 50 blocos

### CA-04 — Idempotência (RF-05, RN-03)
- **Dado** um bloco já processado
- **Quando** a mesma mensagem é reentregue
- **Então** nenhum item, reserva ou contador é duplicado

### CA-05 — Execução parada (RF-08, RN-02)
- **Dado** uma execução parada com blocos ainda no tópico
- **Quando** esses blocos são consumidos
- **Então** são descartados sem efeito

### CA-06 — Rollback não publica (RNF-02)
- **Dado** o processamento de um bloco que falha antes do commit
- **Quando** a transação sofre rollback
- **Então** a mensagem da etapa seguinte não é publicada e o bloco vai para retry/DLQ

### CA-07 — Parâmetros (RF-06)
- **Dado** o tenant com bloco de 200 configurado
- **Quando** uma execução pagina 1.000 devedores
- **Então** gera 5 mensagens de bloco

### CA-08 — Transição de etapa por mensagem (RF-01)
- **Dado** um bloco que concluiu a seleção de devedores
- **Quando** a transação do bloco faz commit
- **Então** a etapa seguinte (qualificação de dívida) é disparada por mensagem de bloco **no tópico dessa etapa**, e não por chamada direta entre componentes nem pelo tópico da etapa anterior

### CA-09 — Progresso por bloco (RF-07)
- **Dado** uma execução com 5 blocos
- **Quando** cada bloco é concluído
- **Então** os contadores da execução são atualizados uma vez por bloco (5 atualizações), nunca por item

## Fora de escopo

- O conteúdo de cada etapa (F-005, F-006, F-008, F-011, F-012).
- Troca do polling do `agendador` por evento de conclusão (opcional, `ARCHITECTURE.md` §10.2).

## Dependências e pendências

- **Features:** F-003.
- **Externas:** E-10 — catálogo dos parâmetros de bloco e concorrência.
- **Em aberto:** nenhuma.
