# F-015 — Ajuizamento excepcional

> Roadmap: [F-015](../../ROADMAP.md) · Fase 6 · Status: PENDING
> Fontes: `PROJECT.md` §3 (item 8), §5 · `ARCHITECTURE.md` §3, §6.4, §8.3 · inventário P22

## Objetivo
Encaminhar manualmente **uma** dívida para ajuizamento, em qualquer situação, desde que ainda não
esteja ajuizada. É o segundo escape do gate CNJ 547 e é paridade com o caminho ativo do `divida`.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Criar execução de origem `EXCEPCIONAL`, fase `AJUIZAMENTO`, com exatamente 1 dívida | `ARCHITECTURE.md` §4, §8.3 |
| RF-02 | Ignorar situação da dívida, validadores e gate | `PROJECT.md` §5; `ARCHITECTURE.md` §8.3 |
| RF-03 | Recusar dívida já ajuizada (processo com número) | `ARCHITECTURE.md` §8.3 |
| RF-04 | Passar pela reserva de exclusividade | `ARCHITECTURE.md` §8.3 |
| RF-05 | Registrar `AvaliacaoGate = LIBERADO_POR_EXCEPCIONAL` com autor e data | `ARCHITECTURE.md` §3, §8.3 |
| RF-06 | Respeitar `AJUIZAMENTO_EXCEPCIONAL_HABILITADO` | roadmap T-144 |
| RF-07 | A execução é sempre prioritária | `ARCHITECTURE.md` §3, §4 |

## Regras de negócio
- **RN-01** — Dívida liquidada, cancelada ou parcelada pode ser ajuizada pelo excepcional; é intencional (`PROJECT.md` §5).
- **RN-02** — A única barreira de negócio é já estar ajuizada.
- **RN-03** — O excepcional não é retirado por mudança de situação (`ARCHITECTURE.md` §5.4).
- **RN-04** — Se a dívida estiver em régua ativa **ainda sem aviso válido** (reservada), o item da notificação é retirado com motivo "encaminhada pela execução X" e sai do kit, que segue com as demais; os passos pendentes só são cancelados se era a última dívida do kit. Dívida **já notificada** está livre: o excepcional a encaminha e a régua segue em paralelo. Avisos já enviados permanecem como evidência (`ARCHITECTURE.md` §6.4).
- **RN-05** — Instituição sem o parâmetro habilitado não pode usar o excepcional.

## Critérios de aceite
### CA-01 — Dívida em qualquer situação é encaminhada (RF-01, RF-02, RF-05)
- **Dado** uma dívida liquidada, não ajuizada, sem notificação nem protesto
- **Quando** o procurador dispara o ajuizamento excepcional
- **Então** o kit é gerado, o BPMN é iniciado e existe `AvaliacaoGate = LIBERADO_POR_EXCEPCIONAL` com autor e data

### CA-02 — Dívida já ajuizada é recusada (RF-03)
- **Dado** uma dívida com processo numerado
- **Quando** o excepcional é disparado
- **Então** a execução recusa a dívida com motivo "já ajuizada" e nada é cadastrado

### CA-03 — Reserva respeitada (RF-04)
- **Dado** uma dívida em execução de protesto ou ajuizamento ativa
- **Quando** o excepcional é disparado para ela
- **Então** a colisão de reserva é registrada e não há encaminhamento duplicado

### CA-04 — Instituição não habilitada (RF-06)
- **Dado** um tenant com `AJUIZAMENTO_EXCEPCIONAL_HABILITADO` desligado
- **Quando** o excepcional é disparado
- **Então** a requisição é negada

### CA-05 — Dívida em régua ainda não notificada (RN-04)
- **Dado** uma dívida sem aviso válido num kit de notificação com outras dívidas
- **Quando** o excepcional a encaminha
- **Então** o item da notificação fica `DESCARTADO` com o motivo, a dívida sai do kit, o kit segue com as demais e as evidências permanecem

### CA-07 — Dívida em régua já notificada (RN-04)
- **Dado** uma dívida já notificada num kit `EM_REGUA`
- **Quando** o excepcional a encaminha
- **Então** o encaminhamento ocorre e a régua segue em paralelo para ela

### CA-06 — Prioridade (RF-07)
- **Dado** uma execução padrão grande em andamento
- **Quando** o excepcional é disparado
- **Então** ele é processado pela fila prioritária

## Fora de escopo
- Mais de uma dívida por execução excepcional.
- Excepcional para protesto.

## Dependências e pendências
- **Features:** F-012 (gerar cobrança), F-010 (`AvaliacaoGate`), F-003 (reserva, prioridade), F-008 (retirada da régua)
- **Externas:** E-09 — tela no `frontng` (a compensação do `Ajuizamento` usa a cadeia `processo.excluiu.processos.0` → `divida`, já existente)
- **Em aberto:** nenhuma
