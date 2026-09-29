# F-011 — Agrupamento e empacotamento de kits

> Roadmap: [F-011](../../ROADMAP.md) · Fase 5 · Status: PENDING
> Fontes: `PROJECT.md` §3 item 4 · `ARCHITECTURE.md` §3, §5.3, §5.5 · inventário P9, P10, P11

## Objetivo

Agrupar as dívidas aptas de uma execução de notificação, protesto ou ajuizamento em kits — a unidade
enviada ao BPMN ou, na notificação, a unidade notificada pela régua (28/09/2026) —, segundo a chave de agrupamento e a estratégia de empacotamento escolhidas, respeitando os
limites do kit e registrando por que cada dívida ficou de fora.

## Requisitos

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Kit é entidade própria com estado explícito | `ARCHITECTURE.md` §3 |
| RF-02 | Chave fixa de agrupamento: `devedorId` + `orgaoOrigemId` | `ARCHITECTURE.md` §5.3 (R2) |
| RF-03 | Flags opcionais do job ampliam a chave: `agrupaIdentificadorDebito`, `agrupaCategoria`, `agrupaAutoInfracao` | `ARCHITECTURE.md` §5.3 (R2) |
| RF-04 | Com `agrupaRaizCnpj`, `devedorId` sai da chave e entra a raiz do CNPJ; o processo sai contra o menor CNPJ | `ARCHITECTURE.md` §5.3 (R2) |
| RF-05 | Restrições do kit: quantidade máxima de dívidas, valor mínimo e valor máximo | `ARCHITECTURE.md` §5.3 (R6) |
| RF-06 | Estratégia de empacotamento escolhida no critério do job: `OTIMIZADA` (padrão) ou `LINEAR` | `ARCHITECTURE.md` §5.3 (R6) |
| RF-07 | Dívidas que ficam de fora saem com motivo "não atingiu valor mínimo do kit" | `ARCHITECTURE.md` §5.3 (R6) |
| RF-08 | Kit de protesto contém exatamente 1 CDA | `PROJECT.md` §3 item 4 · `ARCHITECTURE.md` §5.3 |
| RF-12 | Na execução de notificação, o agrupamento gera kit com via `NOTIFICACAO`, com a mesma chave (RF-02 a RF-04) e a mesma regra de dívida em falha (RF-11); o kit segue para a régua (F-008), não para gerar cobrança. Limites e estratégias (RF-05 a RF-07) **não se aplicam** na notificação: um kit por chave (Q-15, 28/09/2026) | `ARCHITECTURE.md` §6.2 (A23) |
| RF-09 | Validação do kit judicial com as **mesmas regras e parâmetros do `divida`**: mesmo processo de inscrição (`VALIDAR_DIVIDAS_MESMO_PROCESSO_INSCRICAO_KIT_AJUIZAMENTO`), assunto (`TABELA_ASSUNTOS_INSTITUICAO_HABILITADA`), mesma data de atualização e janela de prescrição quando o filtro está ativo. Kit reprovado fica `INVALIDO` com o motivo da regra, reprocessável ou desmontável (F-016) | inventário P10, `anexo-divida.md` C9; decisão de 28/09/2026 |
| RF-10 | Ajuizamento 4.0 / grande porte: kit judicial que cumpre as condições da regra do `divida` recebe a unidade judicial especial (`UNIDADE_JUDICIAL_DEVEDOR_GRANDE_PORTE`), só em tenant com `INSTITUICAO_PERMITE_AJUIZAMENTO_UNIDADE_JUDICIAL_ESPECIAL` ligado | inventário P11, C10 · `ARCHITECTURE.md` §5.3 (A19) · roadmap T-208 |
| RF-11 | Dívida em falha = item com status `ERRO` ou `AVISO` gravado nas etapas anteriores (F-003 RF-04), como a falha na atualização da dívida (F-006 RN-04). Falha na atualização do **devedor** não chega aqui: para o devedor na qualificação (F-005 RN-08). Com `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` ligado, é descartada com o motivo da falha e o kit segue com as demais; desligado, o kit nasce `INVALIDO` com todas as dívidas, não segue para gerar cobrança e fica para o procurador reprocessar ou desmontar (F-016). A invalidez é do kit (motivo agregado); cada dívida mantém o próprio status — só as com problema ficam `ERRO`/`AVISO`, com o motivo concreto | `ARCHITECTURE.md` §5.3 (A20) · `kitcobranca` `AgrupadorDividasService.java:140-155`, `ValidadorKitDividasComErro.java` |
| RNF-01 | Agrupamento determinístico: mesmas entradas geram os mesmos kits | roadmap F-011 |
| RNF-02 | Agrupamento processado por bloco, com partição pela chave de agrupamento | `ARCHITECTURE.md` §5.5 |

## Regras de negócio

- **RN-01** — `OTIMIZADA`: ordena por valor decrescente; com valor mínimo, monta cada kit até atingir
  o mínimo e acomoda as sobras nos kits válidos (mais kits válidos, menos dívidas de fora); sem valor
  mínimo, concentra em menos kits.
- **RN-02** — `LINEAR`: enche o kit na ordem de chegada até a quantidade ou o valor máximo e abre o
  próximo; sobras abaixo do mínimo ficam de fora.
- **RN-03** — Paridade de agrupamento segue o `divida` (caminho ativo do ajuizamento); as estratégias
  e o descarte automático de dívida em falha (RN-06) vêm do `kitcobranca` por escolha explícita.
- **RN-04** — Hardcodes por tenant do legado (Q-02, 28/09/2026): nenhum fica no agrupamento — 179 e a espera da PGEPA não são portados; 143/168 viram parâmetros no processo ADMINISTRATIVO (F-012).
- **RN-05** — Ajuizamento 4.0 entra na 1ª entrega (Q-03, decidida em 27/09/2026). As condições são as
  do `divida` (`RegraAjuizamentoQuatroPontoZero`, inventário C10): instituição habilitada, PJ fora da
  União, agrupamento por raiz de CNPJ, categoria com o prefixo parametrizado, matriz não ativa na capital
  e filial no interior, e valor em UFESP acima do mínimo (ou devedor com a qualificação 4.0). O detalhe
  de cada condição e dos parâmetros vai para o `design.md`, conferido contra o código do `divida`.
- **RN-06** — Kit com dívida em falha segue `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` (decidido em 27/09/2026; não reabrir). O
  `DEVE_GERAR_COBRANCA_QUANDO_DIVIDA_NO_LOTE_COM_ERRO` do `divida` não é portado. `ERRO` (algo deu errado, mas pode ser corrigido)
  e `AVISO` (não há o que fazer, é só um aviso) chegam ao agrupamento; descarte por regra (dívida inapta) sai na etapa que o detectou e não afeta o kit — inclusive a dívida já ajuizada detectada na geração da cobrança (F-012 RF-11, Q-11).
  A decisão fica no agrupamento para permitir **reprocessar o kit** depois de resolvida a causa.

## Critérios de aceite

### CA-01 — Chave fixa (RF-02)
- **Dado** duas dívidas do mesmo devedor e do mesmo órgão de origem
- **Quando** o agrupamento roda sem flags opcionais
- **Então** as duas ficam no mesmo kit

### CA-02 — Órgãos diferentes (RF-02)
- **Dado** duas dívidas do mesmo devedor e de órgãos diferentes
- **Quando** o agrupamento roda
- **Então** ficam em kits diferentes

### CA-03 — Flag opcional (RF-03)
- **Dado** `agrupaCategoria` ligado e duas dívidas do mesmo devedor/órgão com categorias diferentes
- **Quando** o agrupamento roda
- **Então** ficam em kits diferentes

### CA-04 — Raiz do CNPJ (RF-04)
- **Dado** `agrupaRaizCnpj` ligado e dívidas de duas filiais da mesma raiz, mesmo órgão
- **Quando** o agrupamento roda
- **Então** ficam no mesmo kit, com o menor CNPJ como parte

### CA-05 — Limite de quantidade (RF-05)
- **Dado** quantidade máxima 10 e um grupo de 25 dívidas
- **Quando** o empacotamento roda
- **Então** nenhum kit tem mais de 10 dívidas

### CA-06 — OTIMIZADA reduz sobras (RF-06, RN-01)
- **Dado** o mesmo grupo e as mesmas restrições com valor mínimo
- **Quando** empacotado com `OTIMIZADA` e com `LINEAR`
- **Então** `OTIMIZADA` deixa de fora no máximo as dívidas que `LINEAR` deixaria

### CA-07 — Estratégia padrão (RF-06)
- **Dado** critérios sem estratégia informada
- **Quando** o empacotamento roda
- **Então** usa `OTIMIZADA`

### CA-08 — Dívidas de fora (RF-07)
- **Dado** sobras que não atingem o valor mínimo
- **Quando** o empacotamento termina
- **Então** essas dívidas saem da execução com o motivo "não atingiu valor mínimo do kit"

### CA-09 — Protesto 1 CDA (RF-08)
- **Dado** uma execução de protesto com três dívidas do mesmo devedor
- **Quando** o agrupamento roda
- **Então** são gerados três kits de uma CDA cada

### CA-19 — Kit de notificação (RF-12)
- **Dado** uma execução de notificação com três dívidas do mesmo devedor e do mesmo órgão de origem
- **Quando** o agrupamento roda sem flags opcionais
- **Então** é gerado um kit com via `NOTIFICACAO` contendo as três dívidas, encaminhado à régua

### CA-10 — Validação do kit judicial (RF-09)
- **Dado** um kit com dívidas de processos de inscrição diferentes
- **Quando** a validação do kit roda com `VALIDAR_DIVIDAS_MESMO_PROCESSO_INSCRICAO_KIT_AJUIZAMENTO` ligado
- **Então** o kit fica `INVALIDO` com motivo "dívidas de processos de inscrição diferentes" e não segue para gerar cobrança; com o parâmetro desligado, a regra não se aplica

### CA-11 — Determinismo (RNF-01)
- **Dado** as mesmas dívidas e restrições
- **Quando** o agrupamento roda duas vezes
- **Então** os kits resultantes são idênticos

### CA-12 — Kit com estado consultável (RF-01)
- **Dado** um bloco agrupado em kits
- **Quando** um kit é consultado
- **Então** ele existe como registro próprio com via, chave de agrupamento, dívidas e estado explícito (`MONTADO`)

### CA-13 — Partição pela chave de agrupamento (RNF-02)
- **Dado** dívidas do mesmo devedor (ou da mesma raiz de CNPJ, com `agrupaRaizCnpj`) espalhadas em blocos diferentes
- **Quando** o agrupamento processa os blocos
- **Então** todas caem na mesma partição e resultam nos mesmos kits que resultariam num bloco único

### CA-14 — Ajuizamento 4.0 (RF-10, RN-05)
- **Dado** tenant com `INSTITUICAO_PERMITE_AJUIZAMENTO_UNIDADE_JUDICIAL_ESPECIAL` ligado e um kit judicial agrupado por raiz de CNPJ que cumpre as condições da regra do `divida`
- **Quando** o agrupamento roda
- **Então** o kit recebe a unidade judicial de `UNIDADE_JUDICIAL_DEVEDOR_GRANDE_PORTE`

### CA-15 — Ajuizamento 4.0 desligado (RF-10)
- **Dado** o mesmo kit num tenant com `INSTITUICAO_PERMITE_AJUIZAMENTO_UNIDADE_JUDICIAL_ESPECIAL` desligado
- **Quando** o agrupamento roda
- **Então** o kit segue a regra de foro comum, sem unidade judicial especial

### CA-16 — Descarte automático ligado (RF-11, RN-06)
- **Dado** `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` ligado e um grupo de 10 dívidas, 2 com status `ERRO` por falha na atualização
- **Quando** o agrupamento roda
- **Então** as 2 saem `DESCARTADO` com motivo "falha na atualização" e o kit é montado com as outras 8

### CA-17 — Descarte automático desligado (RF-11, RN-06)
- **Dado** `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` desligado e o mesmo grupo
- **Quando** a validação do kit roda
- **Então** o kit fica `INVALIDO` com o motivo "2 dívidas com erro"; as 2 seguem `ERRO` com "falha na atualização" e as 8 seguem `SUCESSO`; nada segue para gerar cobrança

### CA-18 — Dívida com aviso também invalida (RF-11, RN-06)
- **Dado** `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO` desligado e um grupo de 5 dívidas, 1 com status `AVISO`
- **Quando** a validação do kit roda
- **Então** o kit fica `INVALIDO`, como no caso de `ERRO`; com o parâmetro ligado, a dívida com `AVISO` sai `DESCARTADO` e o kit segue com 4

## Fora de escopo

- Cadastro de processo/protesto e início do BPMN (F-012).

## Dependências e pendências

- **Features:** F-010 (gate, para ajuizamento), F-006 (seleção, para notificação e protesto)
- **Externas:** nenhuma nova (o 4.0 usa as APIs que o `divida` já consome: `pessoa` por raiz de CNPJ, unidade judicial no `processo`, índice no `calculo`)
- **Em aberto:** nenhuma (Q-02 decidida em 28/09/2026)
