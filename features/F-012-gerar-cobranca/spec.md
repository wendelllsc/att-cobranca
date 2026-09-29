# F-012 — Gerar cobrança (SAGA)

> Roadmap: [F-012](../../ROADMAP.md) · Fase 5 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 5, §6 · `ARCHITECTURE.md` §7, §10.2 · inventário P12–P15, P17

## Objetivo

Para cada kit montado, cadastrar o processo (EF ou administrativo), registrar o vínculo
(`Ajuizamento` na dívida ou protesto `AGUARDANDO_ENVIO`) e iniciar o BPMN na `demanda`, com passos
persistidos, retry automático retomável e compensação quando as tentativas se esgotam — nada fica em estado indefinido.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Via judicial: `POST /processos/execucao-fiscal` → `POST /dividas/divida/ajuizamento` → `POST /bpm/cobranca/ajuizamento-ef` | `ARCHITECTURE.md` §7 |
| RF-02 | Via protesto: `POST /processos` (ADMINISTRATIVO) → `POST /protestos` (`AGUARDANDO_ENVIO`) → `POST /bpm/cobranca/protesto` | `ARCHITECTURE.md` §7 |
| RF-03 | Estados do kit `MONTADO → PROCESSO_CADASTRADO → VINCULADO → BPMN_INICIADO`, cada passo persistido antes do seguinte | `ARCHITECTURE.md` §7 |
| RF-04 | Falha em qualquer chamada da etapa (processo, vínculo, BPMN) tem **retry automático** que retoma do último estado persistido, até o **parâmetro geral de tentativas** (catálogo, E-10) | `ARCHITECTURE.md` §7 (A22) |
| RF-05 | BPMN já iniciado (`FluxoJaIniciadoException` pela businessKey) é tratado como sucesso | `ARCHITECTURE.md` §7 |
| RF-06 | Tentativas esgotadas antes de `BPMN_INICIADO` ⇒ compensação `DELETE /processos/{id}` (processo inexistente = já compensado), com o mesmo retry automático. Sucesso ⇒ kit e dívidas `DESCARTADO` com a justificativa, **final, sem ação**. Esgotada ⇒ `COMPENSACAO_PENDENTE`, com notificação; única ação: reprocessar a compensação (F-016) | `ARCHITECTURE.md` §7 (A22) |
| RF-07 | Compensação do `Ajuizamento` gravado no `divida` pela cadeia existente: a exclusão do processo publica `processo.excluiu.processos.0` e o `divida` remove o `Ajuizamento` (`ProcessoConsumer.java:62-74`) | `ARCHITECTURE.md` §7, §14 item 4 |
| RF-08 | Montagem do processo EF (partes, corresponsáveis, valores da pasta, taxa judiciária) com os parâmetros usados hoje pelo `divida` | `ARCHITECTURE.md` §7 · inventário §3 |
| RF-09 | Montagem do processo administrativo + protesto com os parâmetros usados hoje pelo `cobranca` legado | `PROJECT.md` §2 · inventário §3 |
| RF-10 | Kit com BPMN iniciado publica `…encaminhou.kits.0` na transação do estado final | `ARCHITECTURE.md` §10.2, §10.3 |
| RF-11 | Antes de cadastrar o processo EF, checar as dívidas do kit já vinculadas a processo **judicial** (E-14). Encontrada ⇒ **descarte por regra**, independente de `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO`: `DESCARTADO` com motivo "dívida já ajuizada (processo X)" e o kit é cadastrado com as demais. Recusa da trigger de unicidade no cadastro (corrida residual) ⇒ nada é criado e o retry refaz a checagem (Q-11, 28/09/2026) | `ARCHITECTURE.md` §7 |
| RNF-01 | 1 mensagem Kafka por kit (isola falha e retry); fila prioritária para execução prioritária | `ARCHITECTURE.md` §5.5 |
| RNF-02 | Chamadas a `processo`, `divida`, `protesto` e `demanda` por REST síncrono | `PROJECT.md` §6 · `ARCHITECTURE.md` §7 |

## Regras de negócio

- **RN-01** — O cadastro de processo/protesto e a gravação do `Ajuizamento` acontecem neste serviço,
  antes do BPMN (D-B2/D-B3).
- **RN-02** — O protesto é removido em cascata quando o processo é excluído (o `protesto` consome
  `processo.excluiu.processos.0`); não há compensação explícita de protesto.
- **RN-03** — Kit de protesto = 1 CDA (F-011).
- **RN-04** — Paridade de montagem: `divida` para EF, `cobranca` legado para protesto.

## Critérios de aceite

### CA-01 — Via judicial completa (RF-01, RF-03, RF-10)
- **Dado** um kit judicial `MONTADO`
- **Quando** os três serviços respondem com sucesso
- **Então** o kit passa por cada estado até `BPMN_INICIADO` e o fato `…encaminhou.kits.0` é publicado

### CA-02 — Via protesto completa (RF-02, RF-03)
- **Dado** um kit de protesto `MONTADO`
- **Quando** os serviços respondem com sucesso
- **Então** o protesto é registrado `AGUARDANDO_ENVIO` e o BPMN de protesto é iniciado

### CA-03 — Retry automático retomável (RF-04)
- **Dado** um kit em `PROCESSO_CADASTRADO` após falha na gravação do `Ajuizamento`, com tentativas restantes
- **Quando** o retry automático roda
- **Então** o processo não é cadastrado de novo e o fluxo retoma na gravação do `Ajuizamento`

### CA-04 — BPMN duplicado (RF-05)
- **Dado** um kit em `VINCULADO` cujo BPMN já foi iniciado numa tentativa anterior
- **Quando** a `demanda` responde `FluxoJaIniciadoException`
- **Então** o kit vai para `BPMN_INICIADO` sem erro

### CA-05 — Compensação (RF-06)
- **Dado** as tentativas de iniciar o BPMN esgotadas
- **Quando** a compensação exclui o processo com sucesso
- **Então** o kit e as dívidas ficam `DESCARTADO` com a justificativa, e o kit não tem ação disponível

### CA-06 — Compensação pendente (RF-06)
- **Dado** falha na exclusão do processo após as tentativas
- **Quando** a compensação se esgota
- **Então** o kit fica `COMPENSACAO_PENDENTE`, consultável, a falha é notificada e a única ação disponível é reprocessar a compensação

### CA-07 — Montagem EF (RF-08)
- **Dado** um kit judicial com corresponsáveis e parâmetro de inclusão de corresponsáveis ligado
- **Quando** o processo EF é montado
- **Então** os corresponsáveis entram como partes conforme a regra do `divida`

### CA-08 — Rollback não publica (RF-10)
- **Dado** falha na transação que grava `BPMN_INICIADO`
- **Quando** a transação é revertida
- **Então** o fato `…encaminhou.kits.0` não é publicado

### CA-09 — Compensação do `Ajuizamento` (RF-07)
- **Dado** um kit judicial que falhou depois de gravar o `Ajuizamento` no `divida` (estado `VINCULADO`)
- **Quando** a compensação roda
- **Então** a exclusão do processo publica `processo.excluiu.processos.0` e o `divida` remove o `Ajuizamento` das dívidas do processo

### CA-10 — Montagem do processo administrativo e do protesto (RF-09)
- **Dado** um kit de protesto e os parâmetros `CLASSE_PROTESTO`, `ID_TIPO_PARTICIPACAO_CONTRARIA_PROCESSO_ADM` e `ID_TIPO_PARTICIPACAO_REPRESENTADA_PROCESSO_ADM` configurados
- **Quando** o processo administrativo e o protesto são montados
- **Então** o processo usa a classe e as participações parametrizadas, e o protesto nasce em `AGUARDANDO_ENVIO`, como no `cobranca` legado

### CA-11 — Uma mensagem por kit (RNF-01)
- **Dado** um bloco agrupado em 4 kits, numa execução prioritária
- **Quando** a geração de cobrança é disparada
- **Então** são publicadas 4 mensagens de kit no tópico prioritário; a falha de um kit não impede os outros três

### CA-12 — Chamadas REST síncronas (RNF-02)
- **Dado** stubs WireMock de `processo`, `divida`, `protesto` e `demanda`
- **Quando** um kit judicial é processado
- **Então** as chamadas ocorrem por HTTP na ordem cadastrar processo → gravar `Ajuizamento` → iniciar BPMN, e nenhum comando Kafka é enviado a esses serviços

### CA-13 — Dívida já ajuizada é descartada por regra (RF-11)
- **Dado** um kit judicial com as CDAs 101, 102 e 103, sendo a 101 já vinculada ao processo judicial P1
- **Quando** a geração de cobrança roda, com `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` ligado **ou** desligado
- **Então** a 101 sai `DESCARTADO` com motivo "dívida já ajuizada (processo P1)" e o processo EF é cadastrado só com 102 e 103

### CA-14 — Visível ao procurador quando a dívida foi informada (RF-11, F-016 RF-12)
- **Dado** uma execução com lista explícita das CDAs 101, 102 e 103, e a 101 já ajuizada no processo P1
- **Quando** o procurador consulta o kit gerado
- **Então** o kit mostra 2 dívidas e a indicação "1 de 3 dívidas informadas descartada: já ajuizada no processo P1"

### CA-15 — Corrida residual barrada pela trigger (RF-11)
- **Dado** uma dívida ajuizada por outra execução entre a checagem prévia e o cadastro
- **Quando** o `processo` recusa o cadastro pela trigger de unicidade
- **Então** nenhum processo é criado, o retry refaz a checagem e o caso segue o CA-13 (teste em PostgreSQL/Oracle; o H2 não tem a trigger)

## Fora de escopo

- Montagem dos kits (F-011).
- Ciclo do protesto com a CRA e etapas do BPMN (`demanda`).
- Decisão da unidade judicial especial do ajuizamento 4.0 (F-011 RF-10); aqui o processo EF só usa a unidade que veio no kit.

## Dependências e pendências

- **Features:** F-011 (kits), F-004 (motor Kafka)
- **Externas:** E-10 — catálogo do parâmetro geral de tentativas; E-14 — checagem de dívidas já em processo judicial (bloqueia RF-11); E-11 — recusa no `divida`, proteção em profundidade (não bloqueia)
- **Decidido (28/09/2026):** Q-02 — tipos de participação do processo ADMINISTRATIVO por `ID_TIPO_PARTICIPACAO_CONTRARIA_PROCESSO_ADM`/`ID_TIPO_PARTICIPACAO_REPRESENTADA_PROCESSO_ADM` (catálogo E-10); nenhum hardcode por tenant
- **Em aberto:** nenhuma
