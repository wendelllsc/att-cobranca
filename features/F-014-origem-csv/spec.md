# F-014 — Origem por arquivo CSV

> Roadmap: [F-014](../../ROADMAP.md) · Fase 6 · Status: PENDING
> Fontes: `PROJECT.md` §3 (item 7) · `ARCHITECTURE.md` §3, §4, §5.5, §11.2 · inventário P2

## Objetivo
Permitir que o procurador dispare uma execução a partir de um arquivo CSV cujas colunas são os critérios de
lista de hoje (`numero_cda`, `processo_inscricao`, `documento_devedor`). É paridade com o caminho ativo (P2 existe no `divida` e no `cobranca` legado)
e, por trazer lista explícita, a execução corre na fila prioritária.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Aceitar upload de CSV e criar uma `Execucao` com origem `MANUAL_ARQUIVO` | `ARCHITECTURE.md` §4; roadmap T-140 |
| RF-06 | Layout: 1ª linha = cabeçalho; **cada coluna é um critério de lista** — `numero_cda` (→ `numeroDividas`), `processo_inscricao` (→ `dividasProcessoInscricao`), `documento_devedor` (→ `documentoDevedores`); colunas opcionais (ao menos uma), listas independentes (a linha N de uma coluna não se relaciona com a linha N de outra) | Q-08, decisão de 28/09/2026 |
| RF-07 | As colunas preenchidas se combinam por **E** entre si e com os critérios da execução, como os critérios de lista no `divida` (`CriteriosSelecaoCobranca.java:521-522`) | Q-08 |
| RF-02 | A execução por arquivo é **prioritária** | `ARCHITECTURE.md` §3 (lista explícita ⇒ prioritária), §5.5 |
| RF-03 | Guardar o arquivo recebido na `lib-storage` com hash, como parte do snapshot da execução | `ARCHITECTURE.md` §11.2; roadmap T-140 |
| RF-04 | Valor mal formatado ou duplicado numa coluna gera aviso de upload sem abortar a execução; valor que não existe no tenant (dívida ou devedor) segue a conciliação da lista explícita e vira item `DESCARTADO` com motivo "dívida não encontrada" / "devedor não encontrado" (F-005 RF-10, F-006 RF-12) | roadmap T-141 |
| RF-05 | As dívidas do arquivo percorrem a mesma pipeline da fase escolhida (seleção, atualização, validação, gate quando aplicável) | `ARCHITECTURE.md` §4, §5 |

## Regras de negócio
- **RN-01** — Toda execução tem uma fase; a execução por arquivo também (notificação, protesto ou ajuizamento).
- **RN-02** — O arquivo restringe o universo; não dispensa filtros, validações nem o gate da fase (salvo bypass nos critérios).
- **RN-03** — O disparo exige o privilégio de disparo; bypass exige também o privilégio de bypass (`PROJECT.md` §5).
- **RN-04** — A reserva de exclusividade vale igualmente: dívida já em execução ativa sai `DESCARTADO` com motivo "dívida em outra execução" (F-003 RN-03), nunca item duplicado.

- **RN-05** — Cada coluna preenchida é uma lista explícita: os devedores e dívidas informados que não forem selecionados aparecem com motivo (F-005 RF-10, F-006 RF-12).
- **RN-06** — A paridade completa dos campos com os critérios de lista do legado é tarefa pendente (T-139); o layout inicial tem as três colunas acima.

## Critérios de aceite
### CA-01 — Execução criada a partir do arquivo (RF-01, RF-03)
- **Dado** um CSV válido com números de dívida e uma fase escolhida
- **Quando** o procurador envia o arquivo
- **Então** é criada uma `Execucao` `MANUAL_ARQUIVO` e o arquivo fica no storage com hash registrado no snapshot

### CA-02 — Execução prioritária (RF-02)
- **Dado** uma execução padrão grande em andamento
- **Quando** uma execução por arquivo é disparada
- **Então** seus blocos são consumidos pela fila prioritária sem esperar a execução padrão

### CA-03 — Valores inválidos não abortam (RF-04)
- **Dado** um CSV com um valor mal formatado, um valor duplicado e um número de dívida inexistente na coluna `numero_cda`
- **Quando** a execução é processada
- **Então** os dois primeiros geram aviso de upload consultável, o inexistente vira item `DESCARTADO` "dívida não encontrada" e os demais valores seguem

### CA-04 — Mesma pipeline da fase (RF-05, RN-02)
- **Dado** um arquivo com dívidas elegíveis e não elegíveis para a fase, sem bypass
- **Quando** a execução termina
- **Então** só as elegíveis viram itens encaminhados e as demais aparecem com motivo de descarte

### CA-05 — Colunas combinadas por E (RF-06, RF-07)
- **Dado** um CSV com `documento_devedor` = {A, B} e `numero_cda` = {101, 205}, em que a 101 é do devedor A e a 205 do devedor C
- **Quando** a execução seleciona
- **Então** só a 101 entra (A e 101); B aparece como devedor informado sem dívida selecionada e a 205 como dívida informada não selecionada, cada um com motivo

### CA-06 — Cabeçalho inválido (RF-06)
- **Dado** um CSV sem cabeçalho reconhecido
- **Quando** o procurador envia o arquivo
- **Então** o upload é recusado com a mensagem das colunas aceitas e nenhuma execução é criada

## Fora de escopo
- Arquivo com critérios não-lista (valores, datas, tributo…): esses continuam nos critérios da execução.
- `LOTE_AJUIZAMENTO_EF` e arquivo de retorno à Fazenda (descontinuado, `PROJECT.md` §3).

## Dependências e pendências
- **Features:** F-006 (seleção de dívidas), F-004 (motor e fila prioritária), F-002 (storage/hash)
- **Externas:** E-01 — seleção via `divida` restrita à lista; E-09 — tela de upload no `frontng`
- **Decidido (28/09/2026):** Q-08 — uma coluna por critério de lista, combinação por E (RF-06, RF-07).
- **Pendente:** T-139 — paridade completa dos campos com o legado; separador, encoding/BOM e limite de tamanho a fixar no `design.md` (referência: vírgula e 200 MB no `divida`, sem tratamento de BOM hoje)
