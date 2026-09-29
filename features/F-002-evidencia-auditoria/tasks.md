# F-002 — Tarefas

> Spec: [spec.md](spec.md) · Design: [design.md](design.md) · Status da feature: PENDING
> Fonte das tarefas e do status desta feature.

| ID | Tarefa | Atende | Depende de | Status |
|---|---|---|---|---|
| T-020 | Canonicalização + hash SHA-256 | RF-01, RN-02 | F-001 | PENDING |
| T-021 | Armazenamento de conteúdo com hash | RF-02 | T-020 | PENDING |
| T-022 | Repository append-only + teste de arquitetura | RF-03, RF-06, RNF-01 | F-001 | PENDING |
| T-023 | Auditoria das ações via `lib-auditoria` | RF-04 | F-001 | PENDING |
| T-024 | Mascaramento de destino | RF-05, RN-03 | F-001 | PENDING |

---

### T-020 — Canonicalização e hash
- **Atende:** RF-01, RN-02 · CA-01, CA-02
- **Arquivos:** `service/evidencia/CanonizadorEvidencia.java`, `service/evidencia/HashEvidenciaService.java`
- **Testes (BDD):** `Dado_registros_equivalentes` → `Entao_hash_igual`; `Dado_campo_alterado` → `Entao_hash_diferente`; datas em fusos diferentes com o mesmo instante ⇒ mesmo hash; `BigDecimal` `1.0` × `1.00` conforme escala definida
- **Verificação:** suíte verde; campo `versaoCanonizacao` presente

### T-021 — Armazenamento de conteúdo
- **Atende:** RF-02 · CA-03
- **Arquivos:** `service/evidencia/ArmazenamentoConteudoService.java`, `entity/evidencia/ReferenciaConteudo.java`
- **Testes:** gravar e ler de volta ⇒ hash confere; conteúdo adulterado no storage ⇒ divergência detectada
- **Verificação:** leitura gera registro de auditoria

### T-022 — Append-only
- **Atende:** RF-03, RF-06, RNF-01 · CA-04, CA-05, CA-08
- **Arquivos:** `repository/RepositorioAppendOnly.java`, `src/test/.../EvidenciaArquiteturaTest.java`
- **Testes:** ArchUnit com regra para pacotes `repository.evidencia..`; teste do registro de correção (`corrigeId`)
- **Verificação:** introduzir método `delete` num repository de teste faz o build falhar

### T-023 — Auditoria
- **Atende:** RF-04 · CA-06
- **Como:** mecanismo da `lib-auditoria` (conferir a API na versão fixada antes de usar — ADR agente 0011)
- **Testes:** ação anotada gera registro com autor, data e objeto

### T-024 — Mascaramento
- **Atende:** RF-05, RN-03 · CA-07
- **Arquivos:** `service/evidencia/MascaradorDestino.java`
- **Testes:** telefone com e sem DDD, e-mail curto e longo, valor nulo
