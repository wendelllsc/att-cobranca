# F-013 — Interrupção

> Roadmap: [F-013](../../ROADMAP.md) · Fase 5 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 6 · `ARCHITECTURE.md` §5.4, §10.1

## Objetivo

Retirar a dívida de qualquer execução em andamento quando o `divida` informar pagamento,
parcelamento, suspensão ou encerramento, evitando novo aviso ou encaminhamento de dívida que não
deve mais ser cobrada.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Consumir `divida.salvou.dividas.0` e avaliar a `situacaoAtual` da dívida | `ARCHITECTURE.md` §5.4, §10.1 |
| RF-02 | `LIQUIDADA`, `CANCELADA`, `ANISTIADO`, `PREESCRITO`, `PARCELADA` e `SUSPENSA` retiram o item da execução ativa (`DESCARTADO` com o motivo da situação) e liberam a reserva | `PROJECT.md` §3 item 6 · `ARCHITECTURE.md` §5.4 |
| RF-03 | Situações parciais mantêm a dívida na execução | `PROJECT.md` §3 item 6 · `ARCHITECTURE.md` §5.4 |
| RF-04 | Kit de protesto/ajuizamento `MONTADO`, em saga (`PROCESSO_CADASTRADO`/`VINCULADO`) ou `INVALIDO` é **desmontado** e as demais dívidas são descartadas com motivo "kit desmontado: dívida X retirada por <situação>"; no kit em saga, desmontar é **compensar** (exclusão do processo, F-012). Kit com BPMN iniciado não é afetado. **Kit de notificação** (`MONTADO`/`EM_REGUA`) não é desmontado: a dívida retirada sai `DESCARTADO` e o kit segue com as demais; kit que fica vazio termina `DESCARTADO` | `ARCHITECTURE.md` §5.4, §6.2 (A23); Q-19 |
| RF-05 | Item de execução `EXCEPCIONAL` não é retirado por situação | `ARCHITECTURE.md` §5.4, §8.3 |
| RF-06 | Mensagem de dívida sem reserva ativa **e** fora de kit de notificação `EM_REGUA` **e** de kit `INVALIDO` é descartada sem efeito (dívida notificada segue na régua sem reserva; kit `INVALIDO` de execução encerrada já não tem reserva) | `ARCHITECTURE.md` §5.4 |
| RF-07 | Toda retirada gera auditoria | `PROJECT.md` §5 |
| RNF-01 | Descarte rápido com consulta indexada e concorrência > 1 (tópico de alto volume) | `ARCHITECTURE.md` §5.4, §10.1 |
| RNF-02 | Reentrega da mesma mensagem não gera efeito duplicado | `ARCHITECTURE.md` §10.3 |

## Regras de negócio

- **RN-01** — A decisão de reagir é do consumer (ADR comum 0005: filtrar no consumer).
- **RN-02** — `parcelamento` e `debito` não publicam fatos; a interrupção depende do reflexo da
  situação no `divida`.
- **RN-03** — A reconsulta da situação antes de cada passo da régua é responsabilidade de F-008.
- **RN-04** — A dívida retirada pode ser selecionada de novo por execução futura, se voltar a ser elegível.

## Critérios de aceite

### CA-01 — Pagamento retira da régua, dívida ainda não notificada (RF-01, RF-02, RF-04, RF-07)
- **Dado** um kit de notificação com as CDAs 101 e 102, a 101 ainda sem aviso válido (reservada)
- **Quando** chega `divida.salvou.dividas.0` com situação `LIQUIDADA` para a 101
- **Então** a 101 fica `DESCARTADO` com o motivo da situação, sua reserva é liberada, a auditoria é gravada e o kit segue só com a 102

### CA-09 — Pagamento retira da régua, dívida já notificada (RF-01, RF-02, RF-06)
- **Dado** a CDA 103 já notificada (sem reserva) num kit `EM_REGUA` com outras dívidas
- **Quando** chega a situação `LIQUIDADA` para a 103
- **Então** a 103 fica `DESCARTADO`, não recebe os próximos avisos e o kit segue com as demais

### CA-10 — Kit de notificação que fica vazio (RF-04)
- **Dado** um kit de notificação `EM_REGUA` com uma única dívida
- **Quando** chega a situação `PARCELADA` para ela
- **Então** a dívida fica `DESCARTADO`, os passos pendentes são cancelados e o kit termina `DESCARTADO`

### CA-02 — Parcelamento desmonta kit não encaminhado (RF-02, RF-04)
- **Dado** uma dívida em kit `MONTADO`
- **Quando** chega a situação `PARCELADA`
- **Então** o item fica `DESCARTADO` com o motivo da situação, o kit é desmontado e as demais dívidas saem `DESCARTADO` com motivo "kit desmontado: dívida X retirada por PARCELADA"

### CA-11 — Kit em saga é compensado (RF-04)
- **Dado** o kit judicial com as CDAs 101, 102 e 103, com o processo EF cadastrado e o BPMN ainda não iniciado
- **Quando** chega a situação `PARCELADA` para a 102
- **Então** o processo é excluído pela compensação, o kit termina `DESCARTADO` e as três dívidas saem `DESCARTADO` (a 102 com o motivo da situação; 101 e 103 com "kit desmontado: dívida 102 retirada por PARCELADA")

### CA-12 — Kit inválido é desmontado (RF-04, RF-06)
- **Dado** um kit `INVALIDO` com as CDAs 101 (`ERRO`), 102 (`ERRO`) e 103, de execução já encerrada
- **Quando** chega a situação `LIQUIDADA` para a 103
- **Então** a mensagem não é descartada pelo filtro, o kit é desmontado e as três dívidas saem `DESCARTADO` com motivo

### CA-03 — Kit encaminhado não é afetado (RF-04)
- **Dado** uma dívida em kit `BPMN_INICIADO`
- **Quando** chega a situação `LIQUIDADA`
- **Então** o kit permanece inalterado

### CA-04 — Situação parcial (RF-03)
- **Dado** um item em execução ativa
- **Quando** chega a situação `LIQUIDADA_PARCIAL`
- **Então** o item continua na execução

### CA-05 — Excepcional (RF-05)
- **Dado** um item de execução `EXCEPCIONAL`
- **Quando** chega a situação `CANCELADA`
- **Então** o item não é retirado

### CA-06 — Dívida fora de execução (RF-06)
- **Dado** uma dívida sem reserva ativa e fora de régua em andamento
- **Quando** chega qualquer mensagem sobre ela
- **Então** a mensagem é descartada sem escrita

### CA-07 — Reentrega (RNF-02)
- **Dado** uma mensagem de `SUSPENSA` já processada
- **Quando** a mesma mensagem é reentregue
- **Então** não há nova retirada nem nova auditoria

### CA-08 — Descarte rápido em alto volume (RNF-01)
- **Dado** 10.000 mensagens de `divida.salvou.dividas.0`, das quais só 10 de dívidas com reserva ativa ou em kit de notificação `EM_REGUA`
- **Quando** as mensagens são consumidas com concorrência maior que 1
- **Então** as 9.990 restantes são descartadas com uma única consulta indexada por mensagem e nenhuma escrita

## Fora de escopo

- Reconsulta antes de cada passo da régua (F-008).
- Reação direta a fatos de `parcelamento`/`debito` (não publicam).

## Dependências e pendências

- **Features:** F-003 (execução/reserva), F-011 (kits)
- **Externas:** nenhuma
- **Decidido (28/09/2026):** Q-19 — kit `MONTADO`, em saga ou `INVALIDO` é desmontado (em saga, compensado) e as demais dívidas são descartadas com motivo (RF-04)
