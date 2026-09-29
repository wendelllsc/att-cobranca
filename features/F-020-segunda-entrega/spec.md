# F-020 — Itens da 2ª entrega

> Roadmap: [F-020](../../ROADMAP.md) · Pós-virada · Status: PENDING
> Fontes: `PROJECT.md` §3 (Fora), §5, §9 · `ARCHITECTURE.md` §4, §8.4, §11.2 · inventário P16

## Objetivo
Agrupar o que foi explicitamente adiado para depois da virada. Cada item abaixo registra o que já
está definido e o que ainda precisa de decisão antes de virar feature própria com design.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Modo simulação: execução sem efeito externo | `PROJECT.md` §3; inventário P16; roadmap T-180 |
| RF-02 | Dispensa do protesto por averbação em órgão de registro e por indicação de bens à penhora | `PROJECT.md` §5; `ARCHITECTURE.md` §8.4 |
| RF-03 | Modo encadeado: uma execução atravessa várias fases conforme as condições são cumpridas | `PROJECT.md` §3; `ARCHITECTURE.md` §4 |
| RF-04 | Expurgo de evidência conforme política de retenção | `PROJECT.md` §5; `ARCHITECTURE.md` §11.2 |

## Regras de negócio
- **RN-01 (simulação)** — Definido: não gera processo, protesto nem BPMN (paridade com `SIMULACAO_EXECUCAO_PROTESTO` do `cobranca` legado). Em aberto: se envia notificação, se grava itens/evidências e como o resultado é apresentado.
- **RN-02 (dispensas)** — Definido: são hipóteses da Res. 547 art. 3º e seguem o registro append-only da dispensa. Em aberto: fonte do dado (nenhum serviço guarda hoje) e se o registro é automático ou pelo procurador.
- **RN-03 (encadeado)** — Definido: reusa o motor por fase; nunca pula o gate (salvo bypass) e produz as mesmas evidências. Em aberto: como a execução reavalia itens ao longo do tempo e quando termina.
- **RN-04 (expurgo)** — Definido: nada é expurgado na 1ª entrega. Em aberto: prazo e critério (E-15 (antiga Q-06)).

## Critérios de aceite
### CA-01 — Simulação sem efeito externo (RF-01, RN-01)
- **Dado** uma execução em modo simulação
- **Quando** ela conclui
- **Então** nenhum processo, protesto, `Ajuizamento` ou BPMN foi criado

### CA-02 — Dispensa por averbação/bens (RF-02, RN-02)
- **Dado** a fonte de dado definida para a hipótese
- **Quando** a dispensa é registrada
- **Então** ela fica append-only com hipótese, origem e hash, e é publicada para indexação no `divida`

### CA-03 — Encadeado respeita o gate (RF-03, RN-03)
- **Dado** uma execução encadeada sem bypass
- **Quando** uma dívida ainda sem `PROTESTADO`/dispensa chegaria ao ajuizamento
- **Então** ela não gera kit judicial

### CA-04 — Expurgo conforme política (RF-04, RN-04)
- **Dado** a política de retenção aprovada
- **Quando** o expurgo roda
- **Então** só registros fora do prazo são removidos e a remoção é auditada

## Fora de escopo
- Qualquer item acima na 1ª entrega.

## Dependências e pendências
- **Features:** F-018 (virada); RF-02 também F-009; RF-03 também F-004/F-010
- **Externas:** fonte de dado de averbação e bens à penhora (serviço a definir)
- **Em aberto:** E-15 (antiga Q-06) — retenção; comportamento detalhado da simulação e do encadeado (RN-01, RN-03) adiado para o planejamento da 2ª entrega (Q-10; `ARCHITECTURE.md` §14 item 21)
