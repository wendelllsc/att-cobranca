# F-002 — Evidência e trilha de auditoria

> Roadmap: [F-002](../../ROADMAP.md) · Fase 0 · Status: PENDING
> Fontes: `PROJECT.md` §5 · `ARCHITECTURE.md` §3, §11.2 · base legal (evidência juntável à inicial) · LGPD

## Objetivo

Oferecer um mecanismo único para registrar evidências **imutáveis e verificáveis** (notificação,
entrega, gate, dispensa, snapshot de execução) e a trilha de auditoria das ações do serviço. É o que
sustenta, perante o juízo, que as condições prévias da Res. 547 foram cumpridas — ou que foram
dispensadas por bypass/excepcional, e por quem.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Calcular hash SHA-256 sobre a forma canônica de um registro; mesma informação ⇒ mesmo hash | `ARCHITECTURE.md` §11.2 |
| RF-02 | Armazenar conteúdo (texto enviado, payload do provedor, anexo) na `lib-storage`, gravando o hash do conteúdo junto ao registro | `ARCHITECTURE.md` §11.2 |
| RF-03 | Registros de evidência são **append-only**: não há atualização nem exclusão; correção é um novo registro que referencia o anterior | `ARCHITECTURE.md` §11.2 |
| RF-04 | Registrar auditoria de seleção, dispensa, bypass, excepcional, retirada e envio (quem, quando, o quê) | `PROJECT.md` §5 |
| RF-05 | Destino de notificação (telefone, e-mail) aparece mascarado em toda consulta; o valor completo só no storage, com acesso auditado | `ARCHITECTURE.md` §11.2; LGPD |
| RF-06 | Nenhum expurgo automático na 1ª entrega | `PROJECT.md` §5 |
| RNF-01 | A garantia de append-only é verificada automaticamente (teste de arquitetura), não por convenção | `ARCHITECTURE.md` §11.2 |

## Regras de negócio

- **RN-01** — Evidência nunca é alterada. Se um dado estiver errado, grava-se um registro de correção apontando para o original.
- **RN-02** — O hash cobre todos os campos com valor probatório do registro e o hash do conteúdo armazenado.
- **RN-03** — O destino da notificação é dado pessoal: exibir mascarado; o completo exige acesso auditado.

## Critérios de aceite

### CA-01 — Hash determinístico (RF-01)
- **Dado** dois registros com os mesmos valores, em ordens de campos diferentes
- **Quando** o hash é calculado
- **Então** os dois hashes são iguais

### CA-02 — Hash sensível a alteração (RF-01, RN-02)
- **Dado** um registro com hash calculado
- **Quando** qualquer campo probatório muda
- **Então** o hash muda

### CA-03 — Conteúdo verificável (RF-02)
- **Dado** um conteúdo armazenado na `lib-storage`
- **Quando** é lido de volta e seu hash recalculado
- **Então** confere com o hash gravado no registro

### CA-04 — Append-only garantido (RF-03, RNF-01)
- **Dado** um repository de evidência
- **Quando** alguém adiciona método de update ou delete
- **Então** o teste de arquitetura falha o build

### CA-05 — Correção por novo registro (RN-01)
- **Dado** uma evidência gravada
- **Quando** é necessário corrigi-la
- **Então** um novo registro referencia o anterior e o original permanece intacto

### CA-06 — Auditoria das ações (RF-04)
- **Dado** uma seleção, dispensa, bypass, excepcional, retirada ou envio
- **Quando** a ação ocorre
- **Então** existe registro de auditoria com autor, data e objeto afetado

### CA-07 — Mascaramento (RF-05, RN-03)
- **Dado** uma notificação por SMS para `11987654321`
- **Quando** consultada por qualquer endpoint
- **Então** o destino aparece mascarado (ex.: `11*****4321`) e o acesso ao completo gera auditoria

### CA-08 — Sem expurgo na 1ª entrega (RF-06)
- **Dado** evidências gravadas em qualquer data
- **Quando** qualquer rotina agendada, consumer ou endpoint do serviço é executado
- **Então** nenhuma evidência é removida; não existe job de expurgo registrado no `agendador` nem endpoint de exclusão de evidência

## Fora de escopo

- As entidades de evidência específicas (`Notificacao`, `Dispensa`, `AvaliacaoGate`…) — nascem nas features que as usam; esta feature entrega o mecanismo.
- Política de retenção/expurgo (E-15 (antiga Q-06), 2ª entrega).

## Dependências e pendências

- **Features:** F-001.
- **Externas:** nenhuma.
- **Em aberto:** E-15 (antiga Q-06) — retenção; não bloqueia (sem expurgo na 1ª entrega).
