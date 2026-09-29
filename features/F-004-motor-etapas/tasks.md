# F-004 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Fonte das tarefas e do status desta feature.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-040 | Tópicos de bloco e kit (+ `.prioritario`) | RF-01–RF-04 | F-003 | PENDING |
| T-041 | Roteamento padrão × prioritário no `MessageService` | RF-04, RN-01 | T-040 | PENDING |
| T-042 | Consumers com idempotência e checagem de parada | RF-01, RF-05, RF-08, RN-02, RN-03 | T-040 | PENDING |
| T-043 | Parâmetros de bloco e concorrência | RF-06 | T-042 | PENDING (catálogo: BLOCKED E-10) |
| T-044 | Progresso e métricas por bloco | RF-07 | T-042 | PENDING |
| T-045 | Testes BDD da feature | todos | T-041–T-044 | PENDING |

---

### T-040 — Tópicos
- **Atende:** RF-01–RF-04
- **Arquivos:** `config/KafkaTopicosConfig.java`, `messageria/BlocoEtapaMensagem.java`, `messageria/KitMensagem.java`
- **Verificação:** um tópico por etapa (+ `.prioritario`), nomes seguindo ADR comum 0005; tópicos criados no ambiente de teste

### T-041 — Roteamento
- **Atende:** RF-04, RN-01 · CA-03
- **Arquivos:** `messageria/MessageService.java`, `messageria/MessageServiceImpl.java`
- **Testes:** execução prioritária ⇒ tópico `.prioritario`; padrão ⇒ tópico base

### T-042 — Consumers
- **Atende:** RF-01, RF-05, RF-08, RN-02, RN-03 · CA-04, CA-05, CA-06, CA-08
- **Arquivos:** `messageria/QualificarDevedoresConsumer.java`, `messageria/QualificarDividasConsumer.java`, `messageria/AgruparDividasConsumer.java`, `messageria/ExecutarPassoReguaConsumer.java`, `messageria/GerarCobrancaKitConsumer.java`, `entity/execucao/BlocoProcessado.java`, `service/execucao/ControleBlocoService.java`, migration `bloco_processado`
- **Testes:** reentrega; execução parada; rollback não publica a etapa seguinte; etapa lenta não atrasa o consumo das outras (tópicos independentes)

### T-043 — Parâmetros
- **Atende:** RF-06 · CA-07
- **Arquivos:** `service/execucao/ParametrosProcessamentoService.java`, `config/KafkaConsumerConfig.java`
- **Bloqueio parcial:** catálogo no `admin` (E-10); usar valor padrão enquanto isso

### T-044 — Progresso
- **Atende:** RF-07 · CA-09
- **Testes:** contadores somam por bloco; nenhuma escrita de contador por item

### T-045 — Testes BDD
- **Atende:** CA-01…CA-09
- **Verificação:** teste que conta mensagens publicadas para N devedores (CA-01) e prova que nenhuma é por dívida
