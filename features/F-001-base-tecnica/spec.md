# F-001 — Base técnica e renomeação

> Roadmap: [F-001](../../ROADMAP.md) · Fase 0 · Status: PENDING
> Fontes: `PROJECT.md` §1, §6 · `ARCHITECTURE.md` §2, §10.3, §11.1 · ADR java 0006, 0018, 0021 · ADR comum 0005 · ADR agente 0006

## Objetivo

Deixar o serviço compilando, subindo e testável com a plataforma Attus (multitenant, segurança,
banco multi-banco, Kafka, parâmetros), já com o nome definitivo `cobranca-dividaativa`, para que as
features de domínio sejam construídas sobre uma base verificada.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | O serviço se chama `cobranca-dividaativa` (artefato Gradle, `spring.application.name`) e usa o pacote `ai.attus.cobrancadividaativa` | `PROJECT.md` §1, `ARCHITECTURE.md` A1 |
| RF-02 | O build usa o `attus-platform-bom` e as `lib-*` da plataforma (core, database, messageria, parametro, security, auditoria, storage, cache, utils) | `ARCHITECTURE.md` §2 |
| RF-03 | A versão do BOM é compatível com Spring Boot 4.1.1, ou a divergência fica registrada com a versão escolhida | `ARCHITECTURE.md` §2; roadmap C-01 |
| RF-04 | Configuração por profile sem nenhum segredo no repositório | ADR segurança 0004 |
| RF-05 | Migrations Flyway multi-banco (Oracle e PostgreSQL, com placeholders) em pasta própria `cobranca-dividaativa` no repo `scripts`, validadas em H2 | `ARCHITECTURE.md` §11.1; ADR java 0006 |
| RF-06 | O comportamento "send dentro de `@Transactional` sincroniza com o commit" é **verificado** na `lib-messageria` da versão fixada e registrado no `ARCHITECTURE.md` §10.3 | `ARCHITECTURE.md` A15; ADR agente 0006 |
| RF-07 | Mensagem com falha de consumo vai para a DLQ central `attus.cmd.dead-letter.0` | `ARCHITECTURE.md` §10 |
| RF-08 | Tentativa de build GraalVM native; em falha, o motivo é registrado e o serviço segue em JVM | `PROJECT.md` §6 |
| RF-09 | Base de testes disponível: H2, classes-base (ADR java 0003) e WireMock (ADR java 0010) | ADRs java |
| RF-10 | Empacotamento e versionamento no padrão do `kitcobranca` (Dockerfile, axion-release, jacoco/sonar) | `kitcobranca/build.gradle` |
| RF-11 | Dependências externas E-01…E-17 abertas como cards com dono (as Q-xx foram decididas em 28/09/2026) | roadmap |
| RNF-01 | Endpoints exigem autenticação e contexto de tenant; sem token ⇒ 401 | ADR java 0005; ADR segurança 0001 |

## Regras de negócio

Nenhuma (feature técnica).

## Critérios de aceite

### CA-01 — Build com a plataforma (RF-01, RF-02, RF-03)
- **Dado** o repositório com o `build.gradle` atualizado
- **Quando** `./gradlew build` é executado
- **Então** todas as `lib-*` resolvem, o artefato se chama `cobranca-dividaativa` e o pacote raiz é `ai.attus.cobrancadividaativa`

### CA-02 — Aplicação sobe protegida (RNF-01)
- **Dado** a aplicação em execução no profile de teste
- **Quando** um endpoint é chamado sem token
- **Então** a resposta é 401

### CA-03 — Sem segredo versionado (RF-04)
- **Dado** o repositório
- **Quando** a varredura de segredos do pipeline roda
- **Então** nenhuma credencial é encontrada em arquivos versionados

### CA-04 — Migration multi-banco (RF-05)
- **Dado** a migration base na pasta `cobranca-dividaativa` do repo `scripts`
- **Quando** é aplicada em H2, PostgreSQL e Oracle
- **Então** roda sem erro nos três

### CA-05 — Sincronia send × commit comprovada (RF-06)
- **Dado** um método `@Transactional` que grava uma entidade e publica uma mensagem
- **Quando** a transação sofre rollback
- **Então** a mensagem não chega ao broker; e o resultado está registrado no `ARCHITECTURE.md` §10.3
- **Senão** (a mensagem chega) a decisão A15 é reaberta antes de qualquer feature publicar fatos

### CA-06 — DLQ central (RF-07)
- **Dado** um consumer de teste que lança exceção
- **Quando** recebe uma mensagem
- **Então** a mensagem é encaminhada a `attus.cmd.dead-letter.0`

### CA-07 — Build native decidido (RF-08)
- **Dado** o build native tentado
- **Quando** conclui ou falha
- **Então** o `ARCHITECTURE.md` §2 registra o resultado (sucesso, ou motivo e decisão por JVM)

### CA-08 — Dependências abertas (RF-11)
- **Dado** a lista E-01…E-17
- **Quando** a feature é concluída
- **Então** cada item tem card com dono e o link está no roadmap

### CA-09 — Base de testes disponível (RF-09)
- **Dado** o projeto após a configuração de testes
- **Quando** um teste de exemplo de cada classe-base (serviço e controller) e um teste com stub WireMock rodam
- **Então** os três passam usando H2 e o stub, sem depender de serviço externo

### CA-10 — Empacotamento e versão (RF-10)
- **Dado** uma tag de versão no repositório
- **Quando** o pipeline de build roda
- **Então** a imagem Docker é gerada com a versão derivada da tag e o relatório de cobertura jacoco é publicado para o sonar

## Fora de escopo

- Qualquer entidade, endpoint ou consumer de domínio.
- Renomear o repositório GitLab `att-cobranca` (decisão de infraestrutura, fora do código).

## Dependências e pendências

- **Features:** nenhuma.
- **Externas:** E-10 (catálogo de parâmetros) não bloqueia esta feature.
- **Em aberto:** nenhuma.
