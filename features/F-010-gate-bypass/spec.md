# F-010 — Gate CNJ 547 e bypass

> Roadmap: [F-010](../../ROADMAP.md) · Fase 4 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 3, §5 · `ARCHITECTURE.md` §3, §8.1, §8.2, §11.2

## Objetivo

Garantir, no domínio, que nenhuma dívida entre em kit judicial sem notificação prévia válida e sem
protesto `PROTESTADO` ou dispensa — exceto pelos escapes explícitos (bypass da execução e
ajuizamento excepcional) —, registrando cada liberação com as provas ou o escape usado.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Antes de agrupar em kit judicial, cada item validado é avaliado pelo gate | `ARCHITECTURE.md` §8.1 |
| RF-02 | Libera quando há notificação válida **e** (protesto `PROTESTADO` **ou** dispensa **ou** negativação — neste caso o gate grava a dispensa `NEGATIVACAO`/`REGRA`, F-009 RF-01) | `PROJECT.md` §3 item 3 · `ARCHITECTURE.md` §8.1, §8.4 |
| RF-04 | Bypass da execução omite a exigência pulada (notificação, protesto ou ambas) | `ARCHITECTURE.md` §8.2 |
| RF-05 | Origem `EXCEPCIONAL` libera sem avaliar condições | `ARCHITECTURE.md` §8.1, §8.3 |
| RF-06 | O gate **reconfirma** no domínio o que a consulta ao índice filtrou, lendo a categoria do protesto no `protesto` (consulta do protesto ativo por dívida, existente) e notificação/dispensa localmente; divergência ⇒ item `DESCARTADO` com motivo "divergência na reconfirmação", nunca liberação | `ARCHITECTURE.md` §8.1; decisão de 28/09/2026 |
| RF-07 | Toda liberação grava `AvaliacaoGate` append-only com resultado (`LIBERADO`, `LIBERADO_POR_BYPASS`, `LIBERADO_POR_EXCEPCIONAL`) e referências às provas | `ARCHITECTURE.md` §3, §8.1 |
| RF-08 | Protesto efetivado = categoria `PROTESTADO`; categoria desconhecida não tem efeito e é registrada em log | `ARCHITECTURE.md` §8.1 |
| RNF-01 | Nenhum parâmetro desliga o gate | `PROJECT.md` §5 · `ARCHITECTURE.md` §8.1 |
| RNF-02 | Nenhum kit judicial nasce sem passar pelo gate (verificável por teste de arquitetura) | `ARCHITECTURE.md` §8.1 |

## Regras de negócio

- **RN-01** — Bypass: flags `bypassNotificacao` e `bypassProtesto` nos critérios da execução; sem
  justificativa; privilégio checado ao salvar o job (fora deste serviço) ou no request manual (F-003).
- **RN-02** — O snapshot dos critérios (com as flags) e o autor da execução são a evidência do bypass.
- **RN-03** — O evento `protesto.protestou.0` (emitido em `CONFIRMADO`) não é usado como efetivação.
- **RN-04** — Avaliação vale para a execução de ajuizamento; a execução de protesto não passa pelo gate nem
  reconfirma a notificação — só o filtro da seleção (F-006). Q-07 revertida em 28/09/2026: sem validade, a notificação
  não muda depois de feita.
- **RN-05** — A notificação não expira: um aviso válido obtido em qualquer momento vale para sempre como prova (28/09/2026).

## Critérios de aceite

### CA-01 — Liberação regular (RF-02, RF-07)
- **Dado** um item com notificação válida e protesto `PROTESTADO`
- **Quando** o gate avalia
- **Então** grava `AvaliacaoGate = LIBERADO` com referências à notificação e ao protesto

### CA-02 — Dispensa no lugar do protesto (RF-02)
- **Dado** um item com notificação válida e dispensa, sem protesto
- **Quando** o gate avalia
- **Então** libera com referência à dispensa

### CA-04 — Notificação não expira (RN-05)
- **Dado** uma notificação com aviso válido de 5 anos atrás
- **Quando** o gate avalia
- **Então** a notificação é considerada válida

### CA-05 — Bypass parcial (RF-04)
- **Dado** uma execução com `bypassProtesto` e sem `bypassNotificacao`
- **Quando** avalia item com notificação válida e sem protesto
- **Então** libera com `LIBERADO`; o bypass usado fica registrado no snapshot dos critérios da execução (`LIBERADO_POR_BYPASS` só com bypass de notificação **e** protesto)

### CA-06 — Bypass parcial não cobre a outra exigência (RF-04, RF-06)
- **Dado** uma execução com `bypassProtesto` e sem `bypassNotificacao`
- **Quando** avalia item sem notificação válida
- **Então** o item é `DESCARTADO` com motivo "divergência na reconfirmação"

### CA-07 — Excepcional (RF-05)
- **Dado** um item de execução `EXCEPCIONAL`
- **Quando** o gate avalia
- **Então** libera com `LIBERADO_POR_EXCEPCIONAL`, autor e data

### CA-08 — Índice defasado (RF-06)
- **Dado** um item retornado pela consulta como protestado, mas cujo protesto está `CANCELADO` no momento da avaliação
- **Quando** o gate reconfirma
- **Então** o item é `DESCARTADO` com motivo "divergência na reconfirmação" e não entra em kit

### CA-09 — Categoria de protesto (RF-08, RN-03)
- **Dado** um protesto em `CONFIRMADO`
- **Quando** o gate avalia
- **Então** não conta como efetivado

### CA-10 — Gate obrigatório (RF-01, RNF-01, RNF-02)
- **Dado** o código do serviço
- **Quando** o teste de arquitetura roda
- **Então** falha se algum caminho criar kit judicial sem `AvaliacaoGate`

## Fora de escopo

- Filtro das condições na consulta ao `divida` (F-006).
- Checagem do privilégio de bypass ao salvar o job (E-05, `agendador`/`frontng`).
- Ajuizamento excepcional como origem de execução (F-015).

## Dependências e pendências

- **Features:** F-006 (seleção), F-008 (notificação), F-009 (dispensa)
- **Externas:** E-02 — índice do `divida` com categoria do protesto (filtro da consulta); a reconfirmação usa a consulta existente do `protesto`, sem dependência nova
- **Em aberto:** nenhuma
