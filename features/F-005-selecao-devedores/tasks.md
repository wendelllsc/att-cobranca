# F-005 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Fonte das tarefas e do status desta feature.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-050 | Cliente da API de seleção + teste de contrato com stub | RF-01 | F-004 | PENDING (integração real: BLOCKED E-01) |
| T-051 | Paginação com gravação em lote | RF-02, RNF-01 | T-050 | PENDING |
| T-052 | Atualização do devedor pelas flags `higienizar`/`enriquecer` dos critérios | RF-03, RN-07 | T-051 | PENDING |
| T-053 | Validador de CPF/CNPJ | RF-04, RN-01 | T-051 | PENDING |
| T-054 | Validador de bloqueio de cobrança | RF-05, RN-02 | T-051 | PENDING |
| T-056 | Validador de endereço consistente (parametrizado; protesto/ajuizamento) | RF-08, RN-04 | T-051 | PENDING |
| T-057 | Critérios de devedor na consulta + validador coberto | RF-09, RN-05 | T-050 | PENDING (integração real: BLOCKED E-01) |
| T-058 | Conciliação da lista explícita de devedores: ausentes viram `DESCARTADO` com motivo | RF-10, RN-09 | T-050, T-057 | PENDING (integração real: BLOCKED E-01) |
| T-055 | Testes BDD da feature | todos | T-051–T-054, T-056–T-058 | PENDING |

---

### T-050 — Cliente de seleção
- **Atende:** RF-01 · CA-01, CA-02
- **Arquivos:** `client/DividaSelecaoClient.java`, `dto/selecao/PaginaDevedoresDto.java`, `dto/selecao/DevedorSelecionadoDto.java`
- **Testes:** WireMock com o contrato proposto (design); página vazia; cursor nulo encerra
- **Bloqueio parcial:** integração real depende de E-01

### T-051 — Paginação e gravação em lote
- **Atende:** RF-02, RNF-01 · CA-01, CA-12
- **Arquivos:** `component/selecao/SelecaoDevedoresComponent.java`, configuração de JDBC batch (`hibernate.jdbc.batch_size` ou equivalente da plataforma)
- **Testes:** 1.200 devedores / bloco 500 ⇒ 3 blocos; contagem de statements de insert agrupados

### T-052 — Atualização do devedor
- **Atende:** RF-03, RN-03, RN-07, RN-08 · CA-05, CA-06, CA-13, CA-14
- **Arquivos:** `service/selecao/AtualizacaoDevedorService.java`, `client/PessoaClient.java`; flags `higienizar`/`enriquecer` no DTO de critérios
- **Antes de implementar:** conferir `CobrancaDevedorAbstract.higienizarDevedor` no `divida` (campos do devedor que a resposta atualiza e tratamento de falha) e reproduzir o comportamento
- **Testes:** as duas flags desligadas (sem chamada); só `higienizar` (`enriquecer=false`); `enriquecer` ligado (`enriquecer=true`); falha no `pessoa` para o devedor em `ERRO` (`FALHA_ATUALIZACAO_DEVEDOR`) na qualificação de devedor, sem consultar dívidas nem gerar kit

### T-053 — CPF/CNPJ
- **Atende:** RF-04, RN-01 · CA-03
- **Arquivos:** `service/selecao/ValidadorDevedor.java`, `service/selecao/ValidadorDocumentoDevedor.java`, `service/selecao/DescarteItemService.java`
- **Testes:** CPF válido, CPF inválido, CNPJ válido, CNPJ inválido, documento vazio

### T-054 — Bloqueio de cobrança
- **Atende:** RF-05, RN-02 · CA-04, CA-07
- **Arquivos:** `service/selecao/ValidadorBloqueioCobrancaDevedor.java`, `client/DividaBloqueioClient.java`
- **Testes:** com e sem bloqueio; reservas liberadas no descarte

### T-056 — Endereço consistente
- **Atende:** RF-08, RN-04 · CA-08, CA-09, CA-10
- **Arquivos:** `service/selecao/ValidadorEnderecoDevedor.java`
- **Antes de implementar:** ler a regra atual no `divida` (inventário C5: sem principal, sem município, inválido para citação quando não "passível de ajuizamento") e reproduzir o mesmo comportamento
- **Testes:** parâmetro ligado × desligado; sem endereço principal; sem município; inválido para citação passível × não passível de ajuizamento; fase notificação ignora

### T-057 — Critérios de devedor
- **Atende:** RF-09, RN-05 · CA-11
- **Arquivos:** `service/selecao/ValidadorCriteriosDevedor.java`; campos no DTO da consulta
- **Testes:** montagem da consulta com os três critérios; validador só executa quando o devedor foi atualizado

### T-058 — Conciliação da lista de devedores
- **Atende:** RF-10, RN-09 · CA-15, CA-16, CA-17
- **Arquivos:** `service/selecao/ConciliadorListaInformada.java` (compartilhado com T-069), `client/DividaSelecaoClient.java` (busca por documento)
- **Testes:** ausente por critério de devedor, sem dívidas aptas e inexistente; execução sem lista não concilia; item guarda o documento informado

### T-055 — Testes BDD
- **Atende:** CA-01…CA-17
