# F-008 — Execução da régua e evidência de notificação

> Roadmap: [F-008](../../ROADMAP.md) · Fase 3 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 2, §5, §8 · `ARCHITECTURE.md` §4, §5.4, §6.2–§6.4, §11.2

## Objetivo

Executar a régua escolhida numa execução de notificação sobre as dívidas selecionadas, registrar
cada aviso e sua prova como evidência append-only com nível de prova explícito, e marcar a dívida
como notificada no primeiro aviso válido — que é a condição prévia exigida pela Res. CNJ 547. A régua segue
todos os passos do ciclo, em paralelo às demais fases (28/09/2026).

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Execução de fase `NOTIFICACAO` roda a régua escolhida **por kit de notificação** (F-011): o devedor recebe o aviso pelo grupo de débitos do kit | `ARCHITECTURE.md` §4, §6.2 (A23) |
| RF-02 | Cada aviso gera `Notificacao` com o texto exato enviado (em storage, com hash) e destino mascarado | `ARCHITECTURE.md` §6.2, §11.2 |
| RF-03 | Cada resposta do provedor gera evidência com **nível de prova**: entregue (canal com confirmação) ou envio aceito (canal sem confirmação) | `PROJECT.md` §3 item 2 · `ARCHITECTURE.md` §6.3 |
| RF-04 | No primeiro aviso válido, os itens do kit ficam `SUCESSO` na etapa `REGUA` e a reserva das dívidas é liberada; a régua **continua** os passos seguintes | `ARCHITECTURE.md` §6.2 |
| RF-04b | **Cada** aviso válido publica o fato `…notificou.dividas.0` com as dívidas do kit (o `divida` indexa a referência; a notificação não expira) | `ARCHITECTURE.md` §6.2, §9.2 |
| RF-05 | Sem aviso válido no prazo do passo, segue para o próximo passo ou o fallback | `ARCHITECTURE.md` §6.2 |
| RF-06 | Fim do ciclo (último passo executado) ⇒ kit `REGUA_CONCLUIDA`; itens sem aviso válido no ciclo ⇒ `DESCARTADO` com motivo "régua esgotada"; dívidas abertas disponíveis para novo ciclo de notificação | `ARCHITECTURE.md` §6.2 |
| RF-07 | Antes de cada passo, a situação das dívidas do kit é reconsultada; dívida retirada sai `DESCARTADO` e o kit segue com as demais (próximos avisos só com as restantes); kit sem dívidas cancela os passos pendentes | `ARCHITECTURE.md` §5.4, §6.2 |
| RF-08 | Bypass ou excepcional que toma dívida em régua ativa **ainda não notificada** ⇒ item `DESCARTADO` com motivo "encaminhada pela execução X" e a dívida sai do kit de notificação, que **segue com as demais**; os passos pendentes só são cancelados se era a última dívida do kit; evidências mantidas | `ARCHITECTURE.md` §6.4 |
| RF-09 | Validação de conteúdo e de links do modelo reaplicada no envio | `ARCHITECTURE.md` §6.1 |
| RF-10 | Status de entrega vindo do `sms`, quando existir, eleva o nível da evidência para entregue | `ARCHITECTURE.md` §6.3 |
| RF-11 | Opt-out **só no WhatsApp**, lido do índice de devedor do `divida`: o passo WhatsApp é pulado e a cobrança não se encerra; SMS e e-mail/carta não têm opt-out | `PROJECT.md` §5 · `ARCHITECTURE.md` §6.2 |
| RNF-01 | Evidências append-only com hash verificável; destino mascarado em consultas (LGPD) | `ARCHITECTURE.md` §11.2 |
| RNF-02 | Envios agrupados em 1 mensagem Kafka por bloco de envios do passo | `ARCHITECTURE.md` §5.5 |
| RNF-03 | Reentrega de status do provedor não duplica evidência nem transição | `ARCHITECTURE.md` §10.3 |
| RNF-04 | Cada canal é uma implementação de interface comum, sem ramificação por canal | `ARCHITECTURE.md` §6.2 |

## Regras de negócio

- **RN-01** — Basta um aviso válido em qualquer canal para a dívida ser considerada notificada; ainda assim a
  régua segue os demais passos/canais do ciclo (o kit pode estar notificado por um canal e ainda não por outro).
- **RN-02** — Canal com confirmação de entrega só conta com status entregue; canal sem confirmação
  conta com envio aceito. O SMS atual é "envio aceito" (`SUCESSO` = aceite do provedor,
  `sms/.../SmsStatus.java:11-18`).
- **RN-03** — O nível obtido fica gravado na evidência; não há regra diferente por nível no gate.
- **RN-04** — Base legal das mensagens: obrigação legal/política pública, nunca consentimento.
- **RN-05** — Nenhum dado é expurgado na 1ª entrega.

## Critérios de aceite

### CA-01 — Primeiro aviso válido não encerra a régua (RF-01, RF-04, RF-04b)
- **Dado** uma execução de notificação com régua SMS (dia 0) + e-mail (dia 8)
- **Quando** o SMS do dia 0 é aceito pelo provedor
- **Então** o item fica `SUCESSO` na etapa `REGUA`, a reserva é liberada, o fato `…notificou.dividas.0` é publicado e o e-mail do dia 8 **é enviado**; se aceito, publica novo fato

### CA-14 — Dívida notificada vai a protesto com a régua em paralelo (RF-04)
- **Dado** uma dívida notificada pelo SMS do dia 0, com o e-mail do dia 8 pendente
- **Quando** uma execução de protesto sem bypass a seleciona
- **Então** a reserva é concedida ao protesto e o e-mail do dia 8 continua agendado

### CA-02 — Evidência com texto exato e hash (RF-02, RNF-01)
- **Dado** um aviso enviado
- **Quando** a `Notificacao` é consultada
- **Então** o conteúdo armazenado confere com o hash gravado e o destino aparece mascarado

### CA-03 — Nível de prova por canal (RF-03)
- **Dado** um canal sem confirmação de entrega
- **Quando** o provedor aceita o envio
- **Então** a evidência registra nível `ENVIO_ACEITO`

### CA-04 — Canal com confirmação exige entrega (RF-03)
- **Dado** um canal com confirmação de entrega
- **Quando** o provedor aceita o envio mas ainda não confirma a entrega
- **Então** o item não é marcado `SUCESSO`

### CA-05 — Fallback (RF-05)
- **Dado** um passo SMS com fallback e-mail
- **Quando** o SMS é rejeitado pelo provedor
- **Então** o fallback é acionado

### CA-06 — Fim do ciclo (RF-06)
- **Dado** um kit com duas dívidas abertas, uma notificada no ciclo e outra cujos avisos falharam todos
- **Quando** o último passo é executado
- **Então** o kit fica `REGUA_CONCLUIDA`, a dívida sem aviso válido fica `DESCARTADO` com motivo "régua esgotada" e ambas podem entrar num novo ciclo de notificação

### CA-07 — Dívida retirada no meio da régua (RF-07)
- **Dado** um kit de notificação com 3 dívidas aguardando o segundo passo
- **Quando** a reconsulta indica situação `PARCELADA` para uma delas
- **Então** essa dívida fica `DESCARTADO` com o motivo e o segundo passo é enviado só com as outras 2

### CA-15 — Última dívida retirada (RF-07)
- **Dado** um kit de notificação com 1 dívida aguardando o segundo passo
- **Quando** a reconsulta indica situação `LIQUIDADA`
- **Então** a dívida fica `DESCARTADO` e os passos pendentes do kit são cancelados

### CA-08 — Bypass toma dívida em régua (RF-08)
- **Dado** um item em régua ativa, ainda sem aviso válido, que já recebeu um aviso rejeitado pelo provedor
- **Quando** uma execução de ajuizamento com bypass seleciona a dívida
- **Então** o item da notificação fica `DESCARTADO` com motivo "encaminhada pela execução X" e sai do kit (os passos pendentes seguem só para as demais dívidas do kit, ou são cancelados se era a última) e a evidência do aviso enviado permanece

### CA-09 — Revalidação do modelo no envio (RF-09)
- **Dado** um modelo cujo domínio do link foi removido da lista oficial após o cadastro
- **Quando** o passo vai ser enviado
- **Então** o envio não ocorre e o motivo é registrado

### CA-10 — Status de entrega eleva o nível (RF-10, RNF-03)
- **Dado** uma evidência `ENVIO_ACEITO` de SMS
- **Quando** chega (uma ou mais vezes) o status de entrega correspondente
- **Então** é registrada uma única evidência de nível `ENTREGUE`

### CA-11 — Opt-out no WhatsApp (RF-11)
- **Dado** um devedor com opt-out de WhatsApp no índice de devedor do `divida`
- **Quando** o passo WhatsApp vai ser executado
- **Então** o passo é pulado, o fallback é usado (se houver) e o item continua na execução

### CA-16 — SMS não tem opt-out (RF-11)
- **Dado** o mesmo devedor, com um passo SMS na régua
- **Quando** o passo SMS vai ser executado
- **Então** o SMS é enviado normalmente

### CA-12 — Envios agrupados por bloco (RNF-02)
- **Dado** um passo da régua com 1.200 dívidas e bloco de 500
- **Quando** o passo é disparado
- **Então** são publicadas 3 mensagens de bloco de envios, nenhuma por dívida

### CA-13 — Canal sem ramificação (RNF-04)
- **Dado** o código do serviço
- **Quando** o teste de arquitetura roda
- **Então** falha se a orquestração da régua ramificar por tipo de canal (`if`/`switch` sobre canal) fora das implementações de canal

## Fora de escopo

- Cadastro da régua (F-007).
- Canal e-mail/carta enquanto não houver adaptador (E-08).
- Gate e seleção das fases de protesto/ajuizamento (F-006, F-010).

## Dependências e pendências

- **Features:** F-006 (seleção de dívidas), F-011 (kit de notificação), F-007 (configuração da régua), F-002 (evidência)
- **Externas** — nenhuma bloqueia a implementação (28/09/2026): o lado deste serviço segue contra a interface de canal e o contrato proposto; canal ausente = envio falho do passo. E-07 — id de correlação na inclusão do `sms`;
  E-08 — status de entrega e adaptador de e-mail/carta (desejável; integração real do RF-10 e do canal e-mail);
  E-13 — serviço de WhatsApp, **principal canal de cobrança** (não existe hoje; integração real do canal WhatsApp)
- **Decidido (28/09/2026):** Q-01 — este serviço só lê o opt-out do índice de devedor (E-12); captação e gravação são externas.
- **Em aberto:** E-16 (antiga Q-17) — templates por canal, pendência externa que não bloqueia
- **Decidido (28/09/2026):** Q-18 — cada passo usa a versão **atual** da régua; régua inativada encerra o kit (`DESCARTADO`, "régua inativada") (F-007 RF-03).
- **Decidido (28/09/2026):** ciclo completo da régua — não para no primeiro aviso válido; fato a cada aviso válido; reserva liberada no primeiro; fim do ciclo libera para novo ciclo. Q-16 — estados do kit `MONTADO` → `EM_REGUA` → `REGUA_CONCLUIDA` | `DESCARTADO` (+ `INVALIDO`/`REPROCESSADO`)
- **Decidido (28/09/2026):** Q-13 — a notificação tem kit (via `NOTIFICACAO`, F-011); reprocessamento e desmontar de kit `INVALIDO` valem como nas outras fases (F-016)
- **Decidido (28/09/2026):** Q-12 — régua **automática**: as dívidas selecionadas entram na esteira e seguem os passos sem escolha de canal por execução; interrompem a esteira as mesmas situações da F-013 RF-02 (`LIQUIDADA`, `CANCELADA`, `ANISTIADO`, `PREESCRITO`, `PARCELADA`, `SUSPENSA`; parciais continuam pelo saldo), via RF-07 e F-013
