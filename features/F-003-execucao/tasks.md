# F-003 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Fonte das tarefas e do status desta feature.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-030 | Entidade `Execucao` | RF-01, RF-02, RF-10 | F-001, F-002 | PENDING |
| T-031 | Entidade `ItemExecucao` e situações | RF-04 | T-030 | PENDING |
| T-032 | Entidade `ReservaCobranca` e serviço de reserva | RF-05, RN-03–RN-05 | T-031 | PENDING |
| T-033 | Migrations multi-banco | RF-01, RF-04, RF-05 | T-030–T-032 | PENDING |
| T-034 | Derivação de prioridade | RF-03 | T-030 | PENDING |
| T-035 | `POST /execucoes` + privilégios | RF-06, RF-09 | T-030–T-034 | PENDING (contrato final com `agendador`: BLOCKED E-05) |
| T-036 | Situação, parar e encerrar | RF-07, RF-08 | T-035 | PENDING |
| T-037 | Telas no `frontng` | — | T-035 | BLOCKED (E-09) |
| T-038 | Testes BDD da feature | todos | T-035, T-036 | PENDING |

---

### T-030 — `Execucao`
- **Atende:** RF-01, RF-02, RF-10 · CA-12
- **Arquivos:** `entity/execucao/Execucao.java` (+ enums `FaseExecucao`, `OrigemExecucao`, `SituacaoExecucao`, `EstrategiaEmpacotamento`), `repository/execucao/ExecucaoRepository.java`, `service/execucao/SnapshotCriteriosService.java`
- **Testes:** snapshot com hash; hash confere após leitura

### T-031 — `ItemExecucao`
- **Atende:** RF-04 · CA-11
- **Arquivos:** `entity/execucao/ItemExecucao.java`, `SituacaoItemExecucao`, repository
- **Testes:** situações finais × não finais

### T-032 — Reserva
- **Atende:** RF-05, RN-03, RN-04, RN-05 · CA-05, CA-06
- **Arquivos:** `entity/execucao/ReservaCobranca.java`, `service/execucao/ReservaCobrancaService.java`
- **Testes:** concorrência real (duas threads, banco H2) ⇒ um item ativo; transferência notificação → ajuizamento com bypass; liberação ao finalizar

### T-033 — Migrations
- **Atende:** RF-01, RF-04, RF-05 (verificação multi-banco nos moldes da F-001)
- **Arquivos:** repo `scripts`, `microservicos/cobranca-dividaativa/` (3 tabelas, índices do design)
- **Verificação:** H2, PostgreSQL e Oracle

### T-034 — Prioridade
- **Atende:** RF-03 · CA-03, CA-04
- **Arquivos:** `service/execucao/PrioridadeExecucaoService.java`
- **Testes:** lista de documentos; lista de números; só filtros; origem `EXCEPCIONAL`

### T-035 — Disparo
- **Atende:** RF-06, RF-09, RN-02 · CA-01, CA-07
- **Arquivos:** `controller/ExecucaoController.java`, `component/execucao/IniciarExecucaoComponent.java`, `dto/execucao/CriteriosExecucaoDto.java`, `mapper/ExecucaoMapper.java`
- **Testes:** controller com e sem privilégio de bypass; payload do `agendador` de exemplo
- **Bloqueio parcial:** o DTO final depende de E-05; implementar contra o contrato proposto

### T-036 — Situação, parar, encerrar
- **Atende:** RF-07, RF-08 · CA-08, CA-09
- **Arquivos:** `component/execucao/PararExecucaoComponent.java`, endpoints no controller
- **Testes:** formato de situação compatível com `CobrancaJob.java`; parada impede novos blocos (verificado com F-004)

### T-037 — Telas
- **Status:** BLOCKED — E-09

### T-038 — Testes BDD
- **Atende:** CA-01…CA-12
- **Verificação:** cada CA tem um `@Nested Dado_*` correspondente; teste de isolamento de tenant (CA-10)
