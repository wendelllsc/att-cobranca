# F-001 — Design

> Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md)

## Visão geral

Feature de infraestrutura. O desenho segue o único precedente com Spring Boot 4 + plataforma
Attus (`kitcobranca/build.gradle`), trocando o que era lição negativa daquele serviço (segredo no
`application.yml`, `lib-action` como núcleo).

## Build (`build.gradle`)

| Item | Valor | Atende |
|---|---|---|
| `rootProject.name` | `cobranca-dividaativa` | RF-01 |
| `group` | `ai.attus` | RF-01 |
| BOM | `mavenBom("ai.attus:attus-platform-bom:${attusPlatformVersion}")`, versão em `gradle.properties` | RF-02, RF-03 |
| Repositórios | `mavenCentral()` + Reposilite (`https://reposilite.dev.attus.ai/master`, credencial por propriedade/variável de ambiente) | RF-02, RF-04 |
| Plugins | `org.springframework.boot 4.1.1`, `io.spring.dependency-management`, `org.graalvm.buildtools.native`, `pl.allegro.tech.build.axion-release`, `jacoco`, `org.sonarqube` | RF-08, RF-10 |
| Dependências | `lib-core`, `lib-database`, `lib-messageria`, `lib-parametro`, `lib-security`, `lib-auditoria`, `lib-storage`, `lib-cache`, `lib-utils`; starters `web`, `data-jpa`, `validation`, `security`; `spring-kafka`; `spring-cloud-starter-openfeign`; Lombok | RF-02 |
| Teste | `spring-boot-starter-test`, H2, WireMock | RF-09 |

`lib-action` **não** entra nesta feature (`PROJECT.md` §6: só se a borda de lote precisar).

## Estrutura de pacotes

```
ai.attus.cobrancadividaativa
├── CobrancaDividaAtivaApplication
├── controller/  component/  service/  repository/  mapper/
├── entity/  dto/  client/  messageria/  config/
```

Cada camada ganha subpacotes por subdomínio quando a primeira classe do subdomínio nascer
(`execucao`, `selecao`, `regua`, `gate`, `kit`, `cobranca`, `evidencia`) — não criar pastas vazias.

## Configuração

| Arquivo | Conteúdo |
|---|---|
| `application.yml` | nome, Kafka (bootstrap por variável), Feign (timeouts), datasource por variável, Flyway desligado localmente se as migrations ficam no repo `scripts` |
| `application-test.yml` | H2, Kafka embutido ou Testcontainers conforme precedente da plataforma |

Credenciais só por variável de ambiente/secret do cluster (RF-04).

## Verificação da sincronia send × commit (RF-06)

Teste de integração dedicado (`SincroniaEnvioTransacaoIT`):

1. método `@Transactional` grava entidade de teste e publica em tópico de teste;
2. força exceção após o `send`;
3. consumidor de teste com `isolation.level=read_committed` confirma que nada chegou.

Resultado copiado para o `ARCHITECTURE.md` §10.3 com a versão da `lib-messageria`.

## Migrations (RF-05)

- Repo `scripts`, pasta nova `microservicos/cobranca-dividaativa/` (mesmo padrão do legado em
  `microservicos/cobranca/`).
- Migration base: apenas estrutura mínima de verificação (ex.: tabela de sequência se a estratégia
  de ID exigir, ADR java 0019). Tabelas de domínio nascem nas features.

## Decisões locais

| ID | Decisão | Alternativa descartada |
|---|---|---|
| DL-01 | Renomear pacote e artefato já na F-001 | Renomear depois: todo código novo nasceria no pacote errado |
| DL-02 | Não incluir `lib-action` agora | Incluir por paridade com o `kitcobranca`: contraria `PROJECT.md` §6 |
| DL-03 | Registrar a versão do BOM após teste real com Spring Boot 4.1.1 | Rebaixar para 4.0.3 sem testar |

## Riscos

- BOM incompatível com Spring Boot 4.1.1 → registrar e escolher (rebaixar Boot ou aguardar BOM).
- `lib-*` bloqueando native (reflexão) → JVM, sem hacks.
