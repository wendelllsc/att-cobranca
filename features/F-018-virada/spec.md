# F-018 — Preparação da virada

> Roadmap: [F-018](../../ROADMAP.md) · Fase 7 · Status: BLOCKED
> Fontes: `PROJECT.md` §2, §8 · `ARCHITECTURE.md` §13 · inventário §2 · `docs/rules/agente/guias/branch-policy.md`

## Objetivo
Trocar, tenant a tenant, o caminho de cobrança dos legados (`divida` para ajuizamento, `cobranca`
legado para protesto) por este serviço, com drenagem e sem duplicidade. Feature operacional:
entrega verificações, parâmetro de roteamento e runbook.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Matriz de paridade 100% `DONE` ou `NOT_REQUIRED` antes da virada de qualquer tenant | `PROJECT.md` §8; roadmap T-160 |
| RF-02 | Parâmetro por tenant faz o `agendador` enviar os jobs a este serviço | `PROJECT.md` §8; `ARCHITECTURE.md` §13 |
| RF-03 | Com o parâmetro ligado, legados não iniciam cobrança nova no tenant e só concluem o que está em trânsito | `PROJECT.md` §8 |
| RF-04 | Durante a drenagem, dívida em lote legado ativo não é selecionada | `ARCHITECTURE.md` §13 |
| RF-05 | Consumidores das APIs legadas migrados (`agendador`, `frontng`, `demanda`, `processo`) | `ARCHITECTURE.md` §13; inventário §2 |
| RF-06 | Gate CNJ 547 e régua ativos desde o primeiro disparo | `PROJECT.md` §8 |
| RF-07 | Runbook por tenant seguindo a ordem de liberação (PGESP por último, janela de observação) | `ARCHITECTURE.md` §13; `branch-policy.md` |

## Regras de negócio
- **RN-01** — Não há migração de estado em trânsito.
- **RN-02** — A régua conta como operante **pelo WhatsApp**, pré-requisito da virada (E-13; `PROJECT.md` §8); confirmação de entrega é desejável, não bloqueante.
- **RN-03** — Virada por tenant; revert só com diagnóstico escrito (`branch-policy.md`).

## Critérios de aceite
### CA-01 — Paridade verificada (RF-01)
- **Dado** a matriz de paridade do roadmap
- **Quando** a virada de um tenant é proposta
- **Então** toda linha está `DONE` ou `NOT_REQUIRED` com decisão registrada

### CA-02 — Roteamento e drenagem (RF-02, RF-03)
- **Dado** o parâmetro ligado para um tenant
- **Quando** o `agendador` dispara jobs de protesto e ajuizamento
- **Então** as execuções nascem neste serviço e nenhum lote novo nasce no `divida` ou no `cobranca` legado daquele tenant

### CA-03 — Sem duplicidade (RF-04)
- **Dado** uma dívida em lote legado ainda ativo
- **Quando** uma execução deste serviço a encontraria
- **Então** ela não é selecionada

### CA-04 — Consumidores migrados (RF-05)
- **Dado** a lista de consumidores do inventário §2
- **Quando** o tenant vira
- **Então** nenhum deles chama API legada de cobrança para aquele tenant

### CA-05 — Gate e régua desde o dia 1 (RF-06)
- **Dado** o primeiro disparo de ajuizamento após a virada
- **Quando** ele roda sem bypass
- **Então** só dívidas com notificação válida e `PROTESTADO`/dispensa são encaminhadas

### CA-06 — Runbook (RF-07)
- **Dado** o runbook por tenant
- **Quando** revisado
- **Então** contém ordem, janela de observação, verificações pós-virada e critério de revert

## Fora de escopo
- Desligamento e remoção de código dos legados.
- `LOTE_AJUIZAMENTO_EF` (descontinuado).

## Dependências e pendências
- **Features:** F-001 a F-017
- **Externas:** E-01, E-02 (`divida`); E-05 (`agendador`: critérios, privilégio, rota); E-07 (`sms`); E-09 (`frontng`); E-10 (`admin`); E-12 (`divida`: opt-out de WhatsApp) e E-13 (serviço de WhatsApp — pré-requisito da virada, 28/09/2026); E-14 (`processo`: checagem de dívida em processo judicial)
- **Decidido (28/09/2026):** Q-04 — tópicos do legado sem consumidor Java não ganham substituto; sem prejuízo na virada (T-164 `NOT_REQUIRED`)
