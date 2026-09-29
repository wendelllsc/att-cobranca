# CLAUDE.md — cobranca

## Contexto do projeto
**Leia [`PROJECT.md`](PROJECT.md) antes de planejar ou implementar** — visão, escopo, fronteiras com outros serviços, gate CNJ 547 e decisões em aberto.

## Stack
Java 25 + Spring Boot 4.1.1 + **Gradle** (não Maven) — GraalVM native desejável, JVM aceitável

## Estrutura típica
```
src/
├── main/java/.../cobranca/
│   ├── controller/     # Controllers REST (entrada)
│   ├── component/      # Components (orquestração)
│   ├── service/        # Services (regras de negócio)
│   ├── repository/     # Repositories JPA
│   ├── mapper/         # Mappers Entity↔DTO
│   ├── entity/         # Entidades JPA
│   └── dto/            # DTOs (request/response)
└── test/
```

## Regras
- Subagent: `.claude/agents/arquitetos/arquiteto-java/SKILL.md`
- Nomenclatura PT-BR, verbos no infinitivo
- Hierarquia: Controller → Component → Service → Repository → Mapper
- BDD: `@Nested class Dado_*` / `@Test void Entao_*`
- Não faça inferências, se não tiver a resposta pergunte
- O resultado esperado desse serviço já existe em outros. A ideia é tirar essas responsabilidades desses outros serviços e concentrar neste.
- O que já existe em outros serviços pode fazer sentido, mas nem sempre. Então pergunte.
- Ao finalizar a implementação de uma feature, altere/crie o arquivo `PLAYBOOK.md` inserindo a regra de negócio. Esse arquivo deverá servir como documentação para o time de produto.

## Docs
- ADRs: `docs/rules/java/adrs/` (suplementos de `docs/rules/comum/adrs/`)
- Checklist: `docs/rules/java/guias/checklist-camadas.md`
- Base legal/normativa: `docs/base_legal_divida_ativa.md`

## Verificação local
```bash
./scripts/dev/verificacao/gate-pipeline.sh --projeto cobranca
```

## Manutenção
Atualizar quando estrutura mudar, novas ADRs forem publicadas ou regras do projeto alterarem.
