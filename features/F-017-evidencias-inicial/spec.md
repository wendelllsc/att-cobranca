# F-017 — Evidências para a petição inicial

> Roadmap: [F-017](../../ROADMAP.md) · Fase 7 · Status: PENDING
> Fontes: `PROJECT.md` §5 · `ARCHITECTURE.md` §8, §9.3, §11.2 · `docs/base_legal_divida_ativa.md`

## Objetivo
Permitir que o BPMN, ao montar o dossiê da inicial, obtenha as provas que liberaram o ajuizamento
(notificação, protesto ou dispensa) ou o escape usado (bypass ou excepcional), com integridade verificável.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Consulta de evidências por kit e por dívida | `ARCHITECTURE.md` §9.3 |
| RF-02 | Resposta inclui `AvaliacaoGate`, notificações com nível de prova, referência de protesto/dispensa e o escape, quando houver | `ARCHITECTURE.md` §3, §8, §9.3 |
| RF-03 | Conteúdos armazenados entregues por link assinado da `lib-storage` | `ARCHITECTURE.md` §9.3 |
| RF-04 | Cada registro devolve o hash para conferência de integridade | `ARCHITECTURE.md` §11.2 |
| RF-05 | Destino (telefone/e-mail) mascarado | `ARCHITECTURE.md` §11.2 |

## Regras de negócio
- **RN-01** — Todo kit judicial encaminhado tem `AvaliacaoGate`; a consulta nunca devolve kit judicial sem ela.
- **RN-02** — Liberação por bypass ou excepcional é devolvida como tal, com autor, data e snapshot; nunca apresentada como notificação/protesto.
- **RN-03** — O nível de prova (entregue × envio aceito) aparece explicitamente em cada notificação.
- **RN-04** — Evidência é append-only; correções aparecem como novos registros que referenciam o anterior.

## Critérios de aceite
### CA-01 — Kit liberado pelo caminho normal (RF-01, RF-02, RF-04)
- **Dado** um kit judicial liberado por notificação válida e protesto `PROTESTADO`
- **Quando** a evidência do kit é consultada
- **Então** a resposta traz a `AvaliacaoGate`, a notificação com nível de prova e a referência do protesto, cada uma com hash

### CA-02 — Kit liberado por escape (RF-02, RN-02)
- **Dado** um kit liberado por bypass
- **Quando** a evidência é consultada
- **Então** a resposta identifica `LIBERADO_POR_BYPASS` com autor, data e hash do snapshot

### CA-03 — Integridade do conteúdo (RF-03, RF-04)
- **Dado** uma notificação com texto no storage
- **Quando** o conteúdo é baixado pelo link assinado
- **Então** o hash calculado do conteúdo confere com o hash devolvido

### CA-04 — Mascaramento (RF-05)
- **Dado** uma notificação por SMS
- **Quando** a evidência é consultada
- **Então** o telefone aparece mascarado

## Fora de escopo
- Montagem do dossiê e da petição (etapa do BPMN, `PROJECT.md` §3).
- Expurgo (F-020 / E-15 (antiga Q-06)).

## Dependências e pendências
- **Features:** F-010 (gate), F-008 (notificações), F-009 (dispensa), F-002 (hash/storage)
- **Externas:** E-06 — passo do BPMN na `demanda` que consome a consulta (define o contrato final)
- **Em aberto:** E-15 (antiga Q-06) — política de retenção (hoje sem expurgo)
