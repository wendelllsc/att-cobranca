# F-001 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Este arquivo é a **fonte das tarefas e do status** desta feature. O `ROADMAP.md` só aponta para cá.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-001 | Esqueleto Spring Boot 4.1.1 / Java 25 / Gradle | — | — | DONE |
| T-002 | Renomear para `cobranca-dividaativa` | RF-01 | T-001 | PENDING |
| T-003 | Abrir E-01…E-17 como cards | RF-11 | — | PENDING |
| T-004 | Importar BOM e Reposilite | RF-02 | T-002 | PENDING |
| T-005 | Verificar BOM × Spring Boot 4.1.1 | RF-03 | T-004 | PENDING |
| T-006 | Adicionar `lib-*` e starters | RF-02 | T-005 | PENDING |
| T-007 | Estrutura de pacotes | RF-01 | T-002 | PENDING |
| T-008 | `application.yml` por profile | RF-04 | T-006 | PENDING |
| T-009 | Base de testes (H2, classes-base, WireMock) | RF-09 | T-006 | PENDING |
| T-010 | Verificar sincronia send × commit | RF-06 | T-009 | PENDING |
| T-011 | Teste de integração da sincronia | RF-06 | T-010 | PENDING |
| T-012 | DLQ central com consumer de teste | RF-07 | T-009 | PENDING |
| T-013 | Pasta no repo `scripts` + migration base | RF-05 | T-002 | PENDING |
| T-014 | Build GraalVM native | RF-08 | T-006 | PENDING |
| T-015 | Dockerfile, axion-release, jacoco/sonar | RF-10 | T-006 | PENDING |

---

### T-001 — Esqueleto
- **Status:** DONE — `CobrancaApplication` + `contextLoads` existem.

### T-002 — Renomear
- **Atende:** RF-01 · CA-01
- **Arquivos:** `settings.gradle`, `src/main/resources/application.properties` → `application.yml`, `src/main/java/ai/attus/cobranca/**` → `ai/attus/cobrancadividaativa/**`, classe principal `CobrancaDividaAtivaApplication`, teste correspondente, `AGENTS.md` (nome e comando `gate-pipeline.sh --projeto cobranca-dividaativa`)
- **Testes:** `contextLoads` verde no pacote novo
- **Verificação:** `grep -r "ai.attus.cobranca\b" src` vazio

### T-003 — Abrir dependências e pendências
- **Atende:** RF-11 · CA-08
- **Como:** subagent `gerenciador-youtrack-att`; um card por E-xx no projeto do serviço dono, um por Q-xx
- **Verificação:** link de cada card registrado na tabela de dependências do `ROADMAP.md`

### T-004 — BOM e Reposilite
- **Atende:** RF-02 · CA-01
- **Arquivos:** `build.gradle`, `gradle.properties` (`attusPlatformVersion`)
- **Verificação:** `./gradlew dependencies` resolve o BOM

### T-005 — Compatibilidade BOM × Boot 4.1.1
- **Atende:** RF-03 · CA-01
- **Como:** build + `contextLoads` com a última versão do BOM; se falhar, testar a versão do `kitcobranca`
- **Verificação:** versão escolhida e motivo registrados no `ARCHITECTURE.md` §2

### T-006 — `lib-*` e starters
- **Atende:** RF-02 · CA-01, CA-02
- **Arquivos:** `build.gradle`
- **Testes:** `contextLoads`; teste de controller sem token ⇒ 401

### T-007 — Pacotes
- **Atende:** RF-01
- **Arquivos:** só a raiz e `config/`; subpacotes nascem com a primeira classe

### T-008 — Configuração
- **Atende:** RF-04 · CA-03
- **Arquivos:** `application.yml`, `application-test.yml`
- **Verificação:** nenhum valor sensível literal; varredura de segredos do pipeline verde

### T-009 — Base de testes
- **Atende:** RF-09 · CA-09
- **Arquivos:** `src/test/java/.../AbstractServiceTest`, `AbstractControllerTest` (conforme ADR java 0003), configuração WireMock
- **Verificação:** teste de exemplo de cada base verde

### T-010 — Verificar sincronia send × commit
- **Atende:** RF-06 · CA-05
- **Como:** inspecionar a configuração do producer factory na `lib-messageria` da versão fixada (`transaction-id-prefix`) e rodar T-011
- **Verificação:** trecho registrado no `ARCHITECTURE.md` §10.3 com versão

### T-011 — Teste da sincronia
- **Atende:** RF-06 · CA-05
- **Arquivos:** `src/test/java/.../SincroniaEnvioTransacaoIT`
- **Testes:** rollback ⇒ nada publicado; commit ⇒ publicado

### T-012 — DLQ
- **Atende:** RF-07 · CA-06
- **Testes:** consumer de teste que lança exceção ⇒ mensagem em `attus.cmd.dead-letter.0`

### T-013 — Migration base
- **Atende:** RF-05 · CA-04
- **Arquivos:** repo `scripts`, `microservicos/cobranca-dividaativa/`
- **Verificação:** skill `desenvolvimento:testar-flyway-dev` ou execução em H2/PostgreSQL/Oracle

### T-014 — Native
- **Atende:** RF-08 · CA-07
- **Verificação:** resultado registrado no `ARCHITECTURE.md` §2

### T-015 — Empacotamento
- **Atende:** RF-10 · CA-10
- **Arquivos:** `Dockerfile`, `build.gradle`
- **Verificação:** imagem gerada; versão derivada da tag
