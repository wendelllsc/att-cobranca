# F-019 — Medição de desempenho

> Roadmap: [F-019](../../ROADMAP.md) · Fase 7 · Status: PENDING
> Fontes: `ARCHITECTURE.md` §5.5, §12 · `PROJECT.md` §7 · inventário §5

## Objetivo
Validar com medição a premissa de pior caso e os riscos de desempenho do desenho (chamadas REST por
kit, consulta de bloqueio por devedor), comparando com o `kitcobranca`. Feature operacional: entrega
números registrados e ajustes de parâmetro, não código de produto.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Cenário de carga com o pior caso do inventário (≥ 5 mil devedores) | `ARCHITECTURE.md` §12; roadmap T-170 |
| RF-02 | Medir mensagens Kafka, linhas gravadas, chamadas remotas e tempo total por execução e por etapa | roadmap T-171 |
| RF-03 | Ajustar tamanho de bloco e concorrência por parâmetro a partir da medição | `ARCHITECTURE.md` §5.5, §12 |
| RF-04 | Registrar resultado no `ARCHITECTURE.md` §12 | roadmap T-171 |

## Regras de negócio
- **RN-01** — Medir antes de otimizar; otimização sem número medido não entra.
- **RN-02** — Nenhuma mensagem Kafka por dívida (`PROJECT.md` §6).

## Critérios de aceite
### CA-01 — Volume de mensagens (RF-01, RF-02, RN-02)
- **Dado** uma execução de ajuizamento com ≥ 5 mil devedores
- **Quando** ela conclui
- **Então** o número de mensagens é ⌈devedores / bloco⌉ por etapa de bloco + 1 por kit, registrado e comparado às ~68 mil do `kitcobranca`

### CA-02 — Custo por etapa (RF-02)
- **Dado** a mesma execução
- **Quando** as métricas são coletadas
- **Então** há tempo e chamadas remotas por etapa (seleção de devedores, seleção de dívidas, agrupamento, gerar cobrança)

### CA-03 — Ajuste registrado (RF-03, RF-04)
- **Dado** os números medidos
- **Quando** bloco/concorrência forem ajustados
- **Então** os valores escolhidos e os números antes/depois constam no `ARCHITECTURE.md` §12

## Fora de escopo
- Otimizações nos serviços vizinhos (`processo`, `divida`, `demanda`); apenas registrar gargalo observado.

## Dependências e pendências
- **Features:** F-012 (pipeline completa até gerar cobrança)
- **Externas:** E-01, E-03 — medição real depende da seleção e da atualização em lote do `divida` (ou de stubs com latência simulada, a registrar como tal)
- **Em aberto:** nenhuma
