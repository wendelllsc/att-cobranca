# Inventário de paridade — `divida` · `cobranca` (legado) · `kitcobranca`

> Levantamento **read-only** do código em 25/09/2026. Critério de pronto da virada (PROJECT.md §8):
> paridade com os três legados + gate CNJ 547. Detalhe com evidência `arquivo:linha` nos anexos:
> [divida](anexo-divida.md) · [cobranca legado](anexo-cobranca-legado.md) · [kitcobranca](anexo-kitcobranca.md).

| Repo | Versão / branch | Stack |
|---|---|---|
| `divida` | 4.935.0 · master `8e647442b` | Java 17, Spring Boot 2.4.2 |
| `cobranca` | 1.70.0 · master `8b3299b` | Java 17, Spring Boot 2.4.2 |
| `kitcobranca` | v0.55.0 · master `c14af7d` | Spring Boot 4.0.3, `lib-action`, platform 1.339.0 |
| `kitajuizamento` | — | esqueleto (4 classes: server, configs, `MessageService`) — sem capacidade de negócio |

## 1. Achados que mudam o PROJECT.md

1. **Não são três fatias — são três implementações sobrepostas da mesma esteira.**
   - `divida` faz **ajuizamento e também tem via de protesto** (`TipoProcessamento.PROTESTO*`).
   - `cobranca` faz **protesto e também tem via judicial completa** (EF, kit, arquivo de retorno).
   - `kitcobranca` faz **ambas** via `lib-action`.
   - O roteamento real está no `agendador` (`CobrancaJob.java:73-91,136-147`): flag de admin
     `COBRANCA_VIA_PROCESSAMENTO_ACTION_HABILITADO` ligada → `kitcobranca`; desligada →
     `PROTESTO` vai para `cobranca`, o resto para `divida`. O `frontng` também chaveia rotas pela flag.
   - **Consequência:** "paridade" = união das regras, e as regras **divergem** entre as três (§4).
2. **Todos vão além de "iniciar o BPMN".** Antes do BPMN, os três:
   - **cadastram o processo** no `processo` (EF com partes, corresponsáveis, valores da pasta,
     taxa judiciária, unidade judicial — ou processo ADMINISTRATIVO para protesto);
   - **registram o protesto** `AGUARDANDO_ENVIO` no `protesto-svc`;
   - **gravam o `Ajuizamento` na dívida** (no `divida`), que é o que impede nova seleção.
   Nenhum gera documento nem protocola (isso já é BPMN / `integrajud`), o que bate com o PROJECT.md.
   Falta decidir se cadastro de processo/protesto e o vínculo dívida↔processo são do `cobranca`
   (antes do BPMN) ou viram etapa do BPMN.
3. **Nenhum retira dívida da esteira por evento** (pagamento/parcelamento/suspensão). Os três só
   filtram no momento da seleção (e revalidam por puxada, sob parâmetro). **Capacidade nova.**
4. **Nenhum tem gate CNJ 547.** Protesto prévio é só filtro opcional; a action de notificação do
   `kitcobranca` lança `NotImplementedException`; `ENVIO_SMS`/`CARTA_COBRANCA`/`NEGATIVACAO_SERASA`
   existem como tipo/job sem implementação. Confirma: régua amigável e gate são **novos**.
5. **A seleção depende do Elasticsearch do `divida`**, e esse índice guarda **estado de cobrança**
   (`loteProcessamentoId`, `loteAjuizamento`, `erroLoteProcessamento`, `ajuizamento`). O
   `DividaDocumentFactory` chega a consultar o `cobranca` legado. O novo serviço precisa de um
   modelo de leitura — é a maior dependência de fronteira.
6. **Há um fluxo com a Fazenda que o PROJECT.md não cita:** `LOTE_AJUIZAMENTO_EF` nasce na carga
   de arquivo do `divida` e termina com **arquivo de retorno TXT** (1 linha por kit + erros; a soma
   tem de bater com o arquivo). Existe no `divida` e no `cobranca`; o `kitcobranca` não gera.
7. **Hardcodes por tenant** que a paridade precisa decidir se viram parâmetro: PGEPA `sleep` 2 s,
   PGMCABO unidade judicial 179, tipos de participação 143/168; regra "ajuizamento 4.0" é
   específica de SP (UFESP, capital, interior).

## 2. Matriz de capacidades

Legenda: ✅ implementado · ◐ parcial/variante · — ausente. Códigos (C#) remetem ao anexo do repo.

| # | Capacidade | divida | cobranca | kitcobranca | Observação |
|---|---|---|---|---|---|
| P1 | Disparo por critérios (agendador + manual) | ✅ C1 | ✅ C1 | ✅ action via critérios | filas PADRAO/PRIORITARIO no divida e cobranca |
| P2 | Disparo por arquivo CSV (números de dívida) | ✅ C2 | ✅ C3-C4 | ✅ action via arquivo | kit agrupa por documento/raiz CNPJ |
| P3 | Lote da Fazenda (`LOTE_AJUIZAMENTO_EF`) + arquivo de retorno TXT | ✅ C3, C17 | ✅ C16 | — | acoplado à carga de débitos do `divida` |
| P4 | Seleção de devedores (≈15 filtros: valor, categoria, tributo, órgão, datas, prescrição, PF/PJ, município, qtd dívidas, `%` mínimo) | ✅ C4 | ✅ C6, C8 | ✅ | mesmo conjunto nos três |
| P5 | Validação do devedor (CPF/CNPJ, endereço, bloqueio, tipo, valor mínimo ajuizado, situação PJ) + higienização/enriquecimento | ✅ C5 | ✅ C7 | ✅ | higienização via `pessoa` |
| P6 | Seleção/validação de dívidas (24–26 critérios: situação, parcelamento, ajuizamento, protesto, prescrição, composição, sob defesa, negativação, irregularidade…) | ✅ C6 | ◐ C8, C17 | ✅ | cobranca **não** tem negativação/sob defesa/irregularidade (parâmetros mortos) |
| P7 | Preparar dívida (sincroniza SDA, PDF CDA, exigibilidade, revalida) | ✅ C7 (dono) | ◐ chama divida | ◐ chama divida | fica no `divida` como API |
| P8 | Exclusividade: dívida em um só processamento | ◐ lote ativo | ◐ erro `PROCESSAMENTO_EM_MAIS_DE_UM_LOTE` | ✅ `reserva_cobranca` | kitcobranca tem furos (complementação e excepcional sem reserva) |
| P9 | Agrupamento em kit judicial (chave órgão+…, máx. dívidas, faixa de valor, empacotamento, complementação por prescrição, menor CNPJ) | ✅ C8, C10 | ✅ C13 | ✅ | algoritmos diferentes (guloso × otimizado/linear) |
| P10 | Validação do kit (assunto CNJ, mesma data de atualização, mesmo processo de inscrição, erros no lote, janela de prescrição) | ✅ C9 | ✅ C14 | ✅ | |
| P11 | Ajuizamento 4.0 / grande porte (unidade judicial especial) | ✅ C10 | ✅ C14 | ✅ | específico de SP? (D-R9) |
| P12 | Montar e cadastrar processo EF (partes, corresponsáveis, valores da pasta, taxa judiciária) | ✅ C11-C12 | ✅ C14 | ✅ | ver achado 2 |
| P13 | Gravar `Ajuizamento` na dívida | ✅ C11 | ✅ via divida | ✅ `POST /dividas/divida/ajuizamento` | ver achado 2 e 5 |
| P14 | Iniciar BPMN `ajuizamento-ef` (idempotente) | ✅ | ✅ | ✅ | **fronteira do novo serviço** |
| P15 | Protesto: kit = 1 dívida, processo ADMINISTRATIVO, protesto `AGUARDANDO_ENVIO`, BPMN `protesto` | ◐ existe, uso incerto | ✅ C11-C12 | ✅ | |
| P16 | Modo simulação (não gera efeito) | — | ✅ `SIMULACAO_EXECUCAO_PROTESTO` | — (parâmetro morto) | |
| P17 | Cancelamento/compensação (exclui processo sem número/BPMN/protocolo) | ✅ C13 | ✅ C12, C14 | ✅ SAGA com retry | consulta `integrajud` só para decidir |
| P18 | Máquina de estados do lote, progresso (polling), métricas | ✅ C14 | ✅ C15 | ✅ lib-action + finalizer | legados avançam por **polling** de `processo`/`protesto` |
| P19 | Erros/alertas/avisos tipados, sumário, consultas de lote | ✅ C15 | ✅ C18 | ✅ resumos | |
| P20 | Reprocessamento (lote, devedor, kit com recorte) | ✅ C16 | ✅ C2, C18 | ✅ | |
| P21 | Descarte manual de item | — | — | ✅ | |
| P22 | Ajuizamento excepcional (1 dívida, sem lote/validadores) | ✅ C18 | — | ✅ | passa pelo gate CNJ 547? (D-B6) |
| P23 | CRUD de bloqueio de cobrança | ✅ C19 (dono) | consulta | consulta | provável que fique no `divida` |
| P24 | Prévia de dívidas por critérios do job | ✅ C20 | — | — | |
| P25 | Indicador mensal de cobranças | — | — | ✅ | |
| P26 | Demanda de conferência de endereço inválido | ✅ (demanda consome) | ◐ tópico sem consumidor | — (parâmetro morto) | |
| P27 | Encerrar/parar lotes (por job, tenant inteiro) | ✅ | ✅ | ✅ | |

**Consumidores das APIs atuais** (precisam migrar na virada): `agendador` (disparo, status,
parar), `frontng` (`DividaLoteProcessamentoController`, `CobrancaLoteProcessamentoController`,
`modules/kitcobranca`, bloqueios, excepcional), `demanda` (reenvio por devedor, fim de lote,
endereço inválido, desmonte de kit), `processo`, `divida` ↔ `cobranca`.

## 3. Parâmetros do `admin` envolvidos

Agrupados por tema; os nomes exatos e onde são lidos estão nos anexos.

- **Seleção:** `PERCENTUAL_MINIMO_SELECAO_DEVEDORES`, `FILTRO_GERACAO_COBRANCA_DIVIDAS_*` (negativação, sob defesa, prescrição futura, suspensão irregularidade), `USO_SITUACAO_COMPOSICAO_HABILITADA`, `FILTRO_DATA_CONSOLIDACAO_PROTESTO_HABILITADO`, `FEATURE_TOGGLE_FILTRO_DEVEDORES_APTOS_AJUIZAMENTO_HABILITADO`, `CONSULTAR_EXIGIBILIDADE_CREDITO`.
- **Validação do devedor:** `VALIDAR_CPF_CNPJ_GERACAO_KIT_AJUIZAMENTO`, `VALIDAR_ENDERECO_CONSISTENTE_GERACAO_KIT_AJUIZAMENTO`, `DEVE_GERAR_DEMANDA_CONFERENCIA_ENDERECO_DEVEDOR_INVALIDO`.
- **Sincronização:** `INTEGRACAO_ATUALIZA_TODA_DIVIDA`, `SINCRONIZAR_DIVIDA_COM_INTEGRACAO_AO_PREPARAR_COBRANCA`, `SINCRONIZAR_DIVIDA_COMPLETA_AO_ATUALIZAR_VALORES`.
- **Kit:** `QUANTIDADE_LIMITE_DE_DIVIDAS_MESMO_PROCESSO_KIT_AJUIZAMENTO`, `AGRUPAR_DIVIDAS_CONSIDERANDO_CORRESPONSAVEIS`, `DESCARTAR_AUTOMATICAMENTE_DIVIDAS_INVALIDAS_DO_KIT_HABILITADO`, `VALIDAR_DIVIDAS_MESMO_PROCESSO_INSCRICAO_KIT_AJUIZAMENTO`, `TABELA_ASSUNTOS_INSTITUICAO_HABILITADA`, `DEVE_GERAR_COBRANCA_QUANDO_DIVIDA_NO_LOTE_COM_ERRO`, `EXIBIR_ERROS_MONTAGEM_KITS_AO_USUARIO`, `AJUIZAR_DEVEDOR_MENOR_CNPJ`.
- **Processo EF / valores:** `INCLUSAO_CORRESPONSAVEIS_COMO_PARTES_EF`, `TIPOS_CORRESPONSAVEIS_FILTRO_EXECUCAO_FISCAL`, `TIPO_PARTICIPACAO_PARTE_CONTRARIA_EXECUCAO_FISCAL`, `USAR_NOME_PESSOA_DA_DIVIDA_PARA_AJUIZAMENTO_EF`, `VALOR_TIPO_ACAO`, `VALOR_TIPO_TAXA_JUDICIARIA`, `PORCENTAGEM_TAXA_JUDICIARIA`, `ID_INDICE_ATUALIZACAO_TAXA_JUDICIARIA`, `ID_INDICE_CALCULO_TAXA_JUDICIARIA`, `LIMITES_CALCULO_TAXA_JUDICIARIA`, `INCLUSAO_HONORARIOS_VALOR_CAUSA`, `VALOR_HONORARIOS_JA_INCORPORADO_VALOR_DIVIDA`, `PORCENTAGEM_HONORARIO`.
- **4.0 / grande porte:** `INSTITUICAO_PERMITE_AJUIZAMENTO_UNIDADE_JUDICIAL_ESPECIAL`, `UNIDADE_JUDICIAL_DEVEDOR_GRANDE_PORTE`, `ID_QUALIFICACAO_DEVEDOR_4_0`, `MUNICIPIO_CAPITAL`, `VALOR_MINIMO_UFESP_DEVEDOR_GRANDE_PORTE`, `PREFIXO_CATEGORIA_DEBITO_QUATRO_PONTO_ZERO`, `INDICE_MOEDA_INSTITUICAO`.
- **Protesto:** `CLASSE_PROTESTO`, `SIMULACAO_EXECUCAO_PROTESTO`, `ID_TIPO_PARTICIPACAO_CONTRARIA_PROCESSO_ADM`, `ID_TIPO_PARTICIPACAO_REPRESENTADA_PROCESSO_ADM`.
- **Operação:** `DEVE_ATUALIZAR_PROGRESSO_POR_DEVEDOR`, `TIPO_DOCUMENTO_ARQUIVO_RETORNO_AJUIZAMENTO`, `AJUIZAMENTO_EXCEPCIONAL_HABILITADO`, `COBRANCA_VIA_PROCESSAMENTO_ACTION_HABILITADO` (agendador).
- **Declarados e nunca lidos** (candidatos a descarte): no `cobranca`, `..._SOB_DEFESA_HABILITADO`, `..._NEGATIVACAO_HABILITADO`, `INTEGRACAO_ATUALIZA_TODA_DIVIDA`, `ANOS_PRESCRICAO_DIVIDA`; no `kitcobranca`, `ESTRATEGIA_ATUALIZACAO_DIVIDA_PARA_COBRANCA`, `DEVE_GERAR_DEMANDA_CONFERENCIA_ENDERECO_DEVEDOR_INVALIDO`, `SIMULACAO_COBRANCA_HABILITADO`, `FEATURE_TOGGLE_USO_EXECUTOR_INTEGRACAO_DIVIDA`. Critérios do DTO sem efeito: `diasUltimoSMS` (no kitcobranca aparenta nunca casar), `smsEnviado`, `cartaEnviada`, `diasUltimaCarta`, `negativada`, `ajuizamentoQuatroPontoZero`, entre outros.

## 4. Divergências de regra entre as implementações

A virada big bang exige **uma** regra. Cada linha precisa de decisão.

| # | Tema | divida | cobranca | kitcobranca |
|---|---|---|---|---|
| R1 | Protesto `PAGO` impede seleção? | impede (`ENVIADO_PARA_CARTORIO`/`PAGO` vetados) | seleção **aceita**, validador **veta** (contradição interna) | via critérios aceita; via arquivo veta |
| R2 | Sem `agrupaCategoria`, como agrupa? | chave por órgão (+ opcionais) | órgão [+bem/fato][+categoria] | **1 kit por dívida** (contradiz o próprio CLAUDE.md) |
| R3 | Valor mínimo no kit de protesto | — | registra erro **mas publica** | — |
| R4 | Dívida com bloqueio de cobrança | registra erro e **continua** no lote | registra erro e **continua** | valida (só com `INTEGRACAO_ATUALIZA_TODA_DIVIDA` ou recorte) |
| R5 | Revalidação após sincronizar | sob 2 parâmetros | — | sob 1 parâmetro |
| R6 | Algoritmo de empacotamento | guloso (valor decrescente) | menor valor + redução de kits | `OTIMIZADA`/`LINEAR` |

## 5. Lições do `kitcobranca` (reforçam PROJECT.md §7)

- A lib impôs **1 etapa = 1 mensagem Kafka + persistência por item** (~68 mil mensagens e ~19 mil
  linhas por execução de 5 mil devedores).
- **Estado de domínio em `Map<String,String>`** (`varchar(255)`, coleção EAGER → N+1). O kit não
  tem entidade própria.
- Chamadas remotas e `saveAndFlush` **item a item**; devedor rebuscado 3–4× no ES; filtros ES em
  `must` em vez de `filter`; consumer com `concurrency=1` (expulsão do consumidor confirmada).
- **Dual-write** (publica antes do commit); `catch (Exception)` sem DLQ; `Thread.sleep` no consumer.
- A mesma cadeia atende judicial e protesto ramificando por `if` no nome da action.
- **Segurança:** `clientSecret` em texto em `kitcobranca/src/main/resources/application.yml:19`.

## 6. Dúvidas para o time (não inferidas)

### Bloqueantes para o PROJECT.md (fronteira/virada)

| # | Pergunta | Origem |
|---|---|---|
| D-B1 | Em **cada tenant de produção**, qual caminho está ativo: lote nativo (`divida`/`cobranca`), `kitcobranca` ou ambos? (`COBRANCA_VIA_PROCESSAMENTO_ACTION_HABILITADO`) — as análises do próprio kitcobranca se contradizem | todos |
| D-B2 | Cadastro do processo (EF/administrativo) e do protesto `AGUARDANDO_ENVIO` continuam **antes** do BPMN, no `cobranca`, ou viram etapa do BPMN? | achado 2 |
| D-B3 | Quem **grava o `Ajuizamento`** na dívida após a virada (evento do `cobranca` → `divida` grava)? É ele que bloqueia nova seleção | divida D8 |
| D-B4 | Modelo de leitura: o `cobranca` consulta o ES do `divida` (como hoje) ou mantém projeção própria? E os campos de estado de cobrança no índice do `divida` saem de lá? | achado 5 |
| D-B5 | `LOTE_AJUIZAMENTO_EF` + arquivo de retorno à Fazenda entram no escopo do `cobranca`? | achado 6 |
| D-B6 | Ajuizamento excepcional passa pelo gate CNJ 547? | kit D11 |
| D-B7 | Via de protesto do `divida` e via judicial do `cobranca` legado estão em uso em algum tenant, ou são código morto? | divida D1, cobranca D1 |
| D-B8 | CRUD de bloqueio de cobrança fica no `divida`? | P23 |

### Regras (decidir antes de implementar)

| # | Pergunta | Origem |
|---|---|---|
| D-R1 | R1–R6 da §4: qual regra vale? | — |
| D-R2 | Kit de protesto continua **1 CDA = 1 protesto**? | kit D8 |
| D-R3 | Hardcodes (PGEPA sleep, PGMCABO 179, participações 143/168) viram parâmetro ou somem? | cobranca D8 |
| D-R4 | Diferenças por tenant de revalidação (`INTEGRACAO_ATUALIZA_TODA_DIVIDA`, `SINCRONIZAR_...`) devem ser reproduzidas? | divida D4, kit D4 |
| D-R5 | Ajuizamento 4.0 é só SP? | kit D9 |
| D-R6 | Parâmetros e critérios mortos (§3) entram na paridade ou são descartados (e removidos do admin/front)? | cobranca D10, kit D10 |
| D-R7 | Filas PADRAO/PRIORITARIO e modo simulação são requisitos? | P1, P16 |
| D-R8 | Tópicos sem consumidor (`cobranca.notificar.endereco.devedor.invalido.0`, `...registrar-erro-dividas-nao-selecionadas.0`, `cobranca.concluiu.*`) têm consumidor fora do Java (BPMN)? | cobranca D7 |

### Pontos técnicos suspeitos nos legados (informativo)

- `cobranca`: `cobrancaGerada` nunca marcado na via EF (D2); `MunicipioSemUnidadeJudicialVinculadaException` nunca lançada (D5); `assuntoCnj` não mapeado no protesto (D6); `attornatus.atualizacao.cobranca` sem leitor (D9).
- `divida`: agrupamento por raiz CNPJ ignora valor mínimo para PJ (D10); `GET /dividas/ajuizadas` sempre vazio (D11); endpoints de notificação à Fazenda sem chamador (D9).
- `kitcobranca`: `/actions/**` cai em `ROLE_ADMIN_ATTORNATUS` (D7); `LOTE_AJUIZAMENTO_EF`/`ENVIO_SMS`/`CARTA_COBRANCA`/`NEGATIVACAO_SERASA` caem na action administrativa (D2); réplicas em produção desconhecidas (D12).
