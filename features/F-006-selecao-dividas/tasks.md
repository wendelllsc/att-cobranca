# F-006 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Fonte das tarefas e do status desta feature.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-060 | Montagem da consulta por fase | RF-01–RF-04 | F-005 | PENDING (integração real: BLOCKED E-01) |
| T-061 | Filtros dependentes do índice ampliado | RF-02, RF-03 | T-060 | BLOCKED (E-02) |
| T-062 | Bypass omitindo filtros | RF-04 | T-060 | PENDING |
| T-063 | Atualização em lote com resultado por dívida | RF-08, RNF-01, RN-04 | T-060 | PENDING (integração real: BLOCKED E-03) |
| T-064 | Catálogo de validadores com cobertura | RF-09, RN-02 | T-063 | PENDING |
| T-065 | Exclusões R1 e valor mínimo R3 | RF-06, RF-07 | T-060 | PENDING |
| T-066 | Validadores de bloqueio | RF-10, RF-11 | T-064 | PENDING |
| T-067 | ~~Parâmetro de validade da notificação~~ | — | — | NOT_REQUIRED (removido em 28/09/2026) |
| T-069 | Conciliação da lista explícita: ausentes viram `DESCARTADO` com motivo | RF-12, RN-06 | T-064 | PENDING (integração real: BLOCKED E-01) |
| T-059 | Validador "já em régua" na fase de notificação | RF-13 | T-064 | PENDING |
| T-068 | Testes BDD da feature | todos | T-059–T-066, T-069 | PENDING |

---

### T-060 — Consulta por fase
- **Atende:** RF-01–RF-04, RN-03 · CA-01, CA-04, CA-05, CA-10
- **Arquivos:** `service/selecao/MontadorConsultaDividas.java`, `client/DividaSelecaoClient.java` (método de dívidas), `dto/selecao/ConsultaDividasDto.java`
- **Testes:** tabela fase × bypass gera as flags esperadas (teste unitário do montador); WireMock para o cliente

### T-061 — Filtros do índice ampliado
- **Status:** BLOCKED — E-02 (o `divida` ainda não indexa notificação, dispensa e categoria do protesto)
- **Verificação quando desbloquear:** teste de contrato com o `divida` cobrindo CA-01…CA-05

### T-062 — Bypass
- **Atende:** RF-04 · CA-02, CA-03
- **Testes:** combinações de `bypassNotificacao`/`bypassProtesto`

### T-063 — Atualização em lote
- **Atende:** RF-08, RNF-01, RN-04, RN-05 · CA-07
- **Arquivos:** `service/selecao/AtualizacaoDividaService.java`, `client/AtualizacaoDividaLoteClient.java`
- **Testes:** parâmetro desligado ⇒ nenhuma chamada; lote com falhas parciais (dívidas com falha ficam `ERRO` e **não** são descartadas); lote maior que o bloco é rejeitado localmente

### T-064 — Catálogo de validadores
- **Atende:** RF-09, RN-02 · CA-08
- **Arquivos:** `service/selecao/ValidadorDivida.java` + implementações
- **Testes:** dívida não atualizada executa só os não cobertos; atualizada executa todos

### T-065 — R1 e R3
- **Atende:** RF-06, RF-07 · CA-06, CA-11
- **Testes:** `PAGO` e `ENVIADO_PARA_CARTORIO` excluídos mesmo com bypass; valor mínimo vazio × preenchido

### T-066 — Bloqueios
- **Atende:** RF-10, RF-11 · CA-09
- **Testes:** um `@Nested` por bloqueio, com caso que bloqueia e caso que não bloqueia; extinção sem mérito não bloqueia

### T-067 — Parâmetro de validade
- **Status:** NOT_REQUIRED — a notificação não expira (28/09/2026); CA-05 passa a ser coberto pela T-060

### T-069 — Conciliação da lista explícita
- **Atende:** RF-12, RN-06 · CA-12, CA-13, CA-14
- **Arquivos:** `service/selecao/ConciliadorListaInformada.java`, `client/DividaSelecaoClient.java` (busca por número)
- **Testes:** ausente por condição de fase, por validador, inexistente e de devedor descartado; execução sem lista não concilia; ordem do catálogo decide o motivo

### T-059 — Já em régua
- **Atende:** RF-13 · CA-15
- **Arquivos:** `service/selecao/ValidadorJaEmRegua.java`
- **Testes:** dívida em kit `EM_REGUA` descartada; dívida de kit `REGUA_CONCLUIDA` elegível; validador só roda na fase `NOTIFICACAO`

### T-068 — Testes BDD
- **Atende:** CA-01…CA-15
