# F-009 — Dispensa do protesto

> Roadmap: [F-009](../../ROADMAP.md) · Fase 4 · Status: PENDING
> Fontes: `PROJECT.md` §5 · `ARCHITECTURE.md` §3, §8.4, §9.2, §11.2

## Objetivo

Registrar as hipóteses que substituem o protesto prévio para fins do gate CNJ 547 — negativação
(por regra) e ineficiência administrativa (por decisão de procurador) — como evidência auditável, e
publicar o fato para que o `divida` indexe a dispensa e a seleção de ajuizamento a considere.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Registrar dispensa por regra com hipótese `NEGATIVACAO` **no gate** da execução de ajuizamento, a partir de `negativada`/`dataNegativacao` que o `divida` já mantém no índice (sem consumer novo) | `PROJECT.md` §5 · `ARCHITECTURE.md` §8.4; decisão de 28/09/2026 |
| RF-02 | Registrar dispensa por procurador com hipótese `INEFICIENCIA_ADMINISTRATIVA`, justificativa obrigatória, autor e data | `PROJECT.md` §5 · `ARCHITECTURE.md` §8.4 |
| RF-03 | Toda dispensa é append-only com hash | `ARCHITECTURE.md` §11.2 |
| RF-04 | Toda dispensa **por procurador** publica `…dispensou.protestos.0` na mesma transação do registro (a negativação já está no índice e não precisa de fato) | `ARCHITECTURE.md` §8.4, §9.2, §10.3 |
| RF-05 | Consulta das dispensas de uma dívida | `ARCHITECTURE.md` §9.3 |
| RNF-01 | Dispensa por procurador exige privilégio próprio | `ARCHITECTURE.md` §8.4 |

## Regras de negócio

- **RN-01** — Hipóteses aceitas na 1ª entrega: `NEGATIVACAO` (origem `REGRA`) e
  `INEFICIENCIA_ADMINISTRATIVA` (origem `PROCURADOR`).
- **RN-02** — Averbação em órgão de registro e indicação de bens à penhora ficam para a 2ª entrega
  (sem fonte de dado hoje).
- **RN-03** — Correção de uma dispensa é um novo registro que referencia o anterior; nunca alteração.
- **RN-04** — A dispensa substitui o protesto no gate; não substitui a notificação prévia.

## Critérios de aceite

### CA-01 — Dispensa por negativação (RF-01, RF-03)
- **Dado** uma dívida notificada, sem protesto, com `negativada = true` e data de negativação no índice do `divida`
- **Quando** uma execução de ajuizamento sem bypass a seleciona e o gate a avalia
- **Então** é gravada dispensa `NEGATIVACAO`/`REGRA` com a data da negativação e hash, o gate libera com referência a ela e nenhum fato `…dispensou.protestos.0` é publicado

### CA-02 — Dispensa por procurador (RF-02, RF-04)
- **Dado** um procurador com o privilégio de dispensa
- **Quando** registra ineficiência administrativa com justificativa
- **Então** a dispensa é gravada com autor, data, justificativa e hash, e o fato é publicado

### CA-03 — Justificativa ausente (RF-02)
- **Dado** um procurador com o privilégio
- **Quando** tenta registrar ineficiência sem justificativa
- **Então** o registro é rejeitado

### CA-04 — Sem privilégio (RNF-01)
- **Dado** um usuário sem o privilégio de dispensa
- **Quando** tenta registrar ineficiência administrativa
- **Então** recebe 403

### CA-05 — Rollback não publica (RF-04)
- **Dado** um registro de dispensa cuja transação falha
- **Quando** a transação é revertida
- **Então** o fato não é publicado

### CA-06 — Correção sem alteração (RF-03, RN-03)
- **Dado** uma dispensa registrada
- **Quando** é corrigida
- **Então** existe um novo registro que referencia o anterior e o original permanece inalterado

### CA-07 — Consulta (RF-05)
- **Dado** uma dívida com duas dispensas
- **Quando** as dispensas da dívida são consultadas
- **Então** ambas são retornadas com hipótese, origem, autor e data

## Fora de escopo

- Dispensa por averbação e por bens à penhora (2ª entrega, F-020).
- Indexação no `divida` (E-02).
- Tela no `frontng` (E-09).

## Dependências e pendências

- **Features:** F-002 (evidência), F-003 (execução/itens)
- **Externas:** E-02 — o `divida` precisa consumir `…dispensou.protestos.0` (dispensa por procurador) para a seleção por fase considerar a dispensa; a negativação já está no índice; E-09 — tela
- **Em aberto:** nenhuma
