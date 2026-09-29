# F-007 — Configuração da régua

> Roadmap: [F-007](../../ROADMAP.md) · Fase 3 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 2, §5 · `ARCHITECTURE.md` §3, §6.1

## Objetivo

Permitir que o procurador cadastre e mantenha as réguas do tenant — a pipeline da fase de
notificação (passos, canais, intervalos, fallback e modelo de mensagem) —, garantindo já no
cadastro as regras de conteúdo exigidas pela LGPD e pela política anti-golpe.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Um tenant pode ter várias réguas; cada uma tem nome e vigência | `PROJECT.md` §3 item 2 · `ARCHITECTURE.md` §3, §6.1 |
| RF-02 | Cada régua tem passos ordenados; cada passo define canal, dias após o passo anterior, modelo de mensagem e canal de fallback | `ARCHITECTURE.md` §6.1 |
| RF-03 | Cadastro, consulta, alteração e inativação de régua pelo procurador na tela, via API REST. **Inativar** encerra os kits `EM_REGUA` que a usam (kit `DESCARTADO` "régua inativada", itens `DESCARTADO`, reservas liberadas); **alterar** vale para os passos ainda não executados desses kits, e é **recusada** se levar algum desses passos a uma data já passada; passo removido deixa de ser executado nos kits em curso (Q-18) | `PROJECT.md` §3 item 2 · `ARCHITECTURE.md` §6.1 |
| RF-04 | O modelo de mensagem é validado ao salvar: conteúdo mínimo (órgão, número da CDA, valor, canal oficial) — direção provável: **regra por canal** (Q-17) | `PROJECT.md` §5 · `ARCHITECTURE.md` §6.1 |
| RF-05 | Modelo de SMS não pode conter natureza da dívida nem dado de terceiro | `PROJECT.md` §5 |
| RF-06 | Links no modelo só para domínio oficial (lista no `admin`), nunca encurtados; validados ao salvar | `PROJECT.md` §5 · `ARCHITECTURE.md` §6.1 |
| RNF-01 | Operações de escrita exigem privilégio de configuração de régua; isolamento por tenant | `ARCHITECTURE.md` §2 · ADR segurança 0001 |
| RNF-02 | Controller com no máximo 12 endpoints | ADR java 0012 |

## Regras de negócio

- **RN-01** — A régua pertence à fase de notificação; não contém protesto nem ajuizamento.
- **RN-02** — Gatilhos de mercado (inscrição, pré-protesto, pré-ajuizamento, campanhas) são réguas
  distintas usadas em jobs distintos; não há mecanismo de gatilho próprio (`ARCHITECTURE.md` §6.1).
- **RN-03** — A lista de domínios oficiais é limite global no `admin`; a estrutura da régua fica em
  tabelas deste serviço (A10).
- **RN-04** — A mesma validação de conteúdo e de links é reaplicada no envio (coberta em F-008).
- **RN-05** — Tom sem linguagem intimidatória é boa prática obrigatória (CDC art. 42); não há regra
  automática definida para verificá-lo.

## Critérios de aceite

### CA-01 — Cadastrar régua válida (RF-01, RF-02, RF-03)
- **Dado** um procurador com privilégio de configuração
- **Quando** cadastra uma régua com nome, vigência e dois passos (SMS no dia 0; e-mail 8 dias depois, com fallback)
- **Então** a régua é persistida no tenant do usuário com os passos na ordem informada

### CA-10 — Inativar encerra os ciclos em curso (RF-03)
- **Dado** a régua "Inscrição" em uso por kits `EM_REGUA`
- **Quando** o procurador a inativa
- **Então** esses kits ficam `DESCARTADO` com motivo "régua inativada", seus itens `DESCARTADO`, as reservas liberadas, nenhum passo pendente é enviado e as dívidas podem entrar num novo ciclo

### CA-11 — Alterar vale para os passos futuros (RF-03)
- **Dado** kits `EM_REGUA` que já executaram o passo WhatsApp do dia 0, com o e-mail previsto para o dia 15
- **Quando** o procurador antecipa o e-mail para o dia 10 e altera o texto dele
- **Então** os kits seguem na mesma régua, o e-mail sai no dia 10 com o texto novo e o passo do dia 0 não é refeito

### CA-12 — Alteração para data passada é recusada (RF-03)
- **Dado** um kit `EM_REGUA` que executou o WhatsApp em 01/10, com o e-mail previsto para 16/10 (15 dias depois)
- **Quando** em 08/10 o procurador tenta mudar o intervalo do e-mail para 5 dias (o que daria 06/10)
- **Então** a alteração é recusada com a indicação do passo e dos kits afetados, e a régua fica como estava

### CA-13 — Passo removido não é executado (RF-03)
- **Dado** kits `EM_REGUA` com o passo SMS do dia 5 ainda não executado
- **Quando** o procurador remove esse passo da régua
- **Então** os kits em curso não enviam o SMS e seguem para o passo seguinte na data prevista

### CA-02 — Várias réguas no mesmo tenant (RF-01)
- **Dado** um tenant com a régua "Inscrição" cadastrada
- **Quando** o procurador cadastra a régua "Pré-protesto"
- **Então** as duas ficam disponíveis para escolha em execuções de notificação

### CA-03 — Modelo sem conteúdo mínimo (RF-04)
- **Dado** um passo cujo modelo não referencia o número da CDA
- **Quando** o procurador salva a régua
- **Então** o cadastro é rejeitado indicando o item ausente

### CA-04 — Natureza da dívida ou dado de terceiro no SMS (RF-05)
- **Dado** um passo de canal SMS cujo modelo inclui a natureza da dívida
- **Quando** o procurador salva a régua
- **Então** o cadastro é rejeitado

### CA-05 — Link fora do domínio oficial (RF-06)
- **Dado** a lista de domínios oficiais do tenant no `admin`
- **Quando** o modelo contém link para domínio fora da lista
- **Então** o cadastro é rejeitado

### CA-06 — Link encurtado (RF-06)
- **Dado** um modelo com link de encurtador
- **Quando** o procurador salva a régua
- **Então** o cadastro é rejeitado

### CA-07 — Sem privilégio (RNF-01)
- **Dado** um usuário sem privilégio de configuração de régua
- **Quando** tenta cadastrar ou alterar uma régua
- **Então** recebe 403

### CA-08 — Isolamento de tenant (RNF-01)
- **Dado** uma régua do tenant A
- **Quando** um usuário do tenant B a consulta pelo id
- **Então** a régua não é retornada

### CA-09 — Granularidade do controller (RNF-02)
- **Dado** o código do serviço
- **Quando** o teste de arquitetura roda
- **Então** falha se o controller da régua expuser mais de 12 endpoints

## Fora de escopo

- Envio das mensagens e evidência (F-008).
- Tela no `frontng` (E-09).
- Verificação automática de tom da mensagem.

## Dependências e pendências

- **Features:** F-002 (evidência/auditoria)
- **Externas:** E-09 — telas no `frontng`; E-10 — catálogo do parâmetro de domínios oficiais no `admin`
- **Decidido (28/09/2026):** Q-18 — inativar encerra os kits em régua e libera as dívidas; alterar vale para os passos futuros (RF-03).
- **Pendência externa:** E-16 (antiga Q-17) — templates por canal; a validação de conteúdo mínimo é **por canal e configurável**, sem bloquear a implementação. Afeta RF-04 (regra de conteúdo mínimo por canal) e RF-02 (o canal de fallback precisará de modelo próprio se o conteúdo for por canal)
