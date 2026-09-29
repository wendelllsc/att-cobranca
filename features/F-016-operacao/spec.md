# F-016 — Operação de execuções e kits

> Roadmap: [F-016](../../ROADMAP.md) · Fase 6 · Status: PENDING
> Fontes: `PROJECT.md` §3 · `ARCHITECTURE.md` §4, §7 · inventário P19, P20, P24, P26

## Objetivo
Dar ao procurador e ao suporte a visibilidade e as ações operacionais que o caminho ativo oferece
hoje: erros e avisos tipados, sumário, consultas, reprocessamento (devedor e kit), desmontagem de kit
inválido, reprocessamento de compensação, prévia de seleção e a demanda de conferência de endereço inválido.

## Requisitos
| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Status `ERRO`/`AVISO`/`DESCARTADO` com motivo tipado por item e kit | inventário P19; roadmap T-147; `ARCHITECTURE.md` A21 |
| RF-02 | Sumário por execução: selecionadas, descartadas por motivo, encaminhadas, com erro | roadmap T-147 |
| RF-03 | Consultas paginadas de execuções, itens e kits | roadmap T-148; ADR comum 0004 |
| RF-04 | Reprocessar **devedor** com `ERRO` **que ainda não gerou kit** na execução: volta à qualificação de devedor e refaz a consulta das dívidas no `divida`. Devedor com kit não é reprocessável: o pedido é recusado com exceção ("o devedor não pode ser reprocessado porque já existem kits gerados para ele") e nenhum kit é desfeito; o caminho é reprocessar o kit | inventário P20; roadmap T-149; `ARCHITECTURE.md` §7.1 (A22) |
| RF-05 | Prévia das dívidas que os critérios de um job selecionariam | inventário P24; roadmap T-150 |
| RF-06 | Com `DEVE_GERAR_DEMANDA_CONFERENCIA_ENDERECO_DEVEDOR_INVALIDO` ligado, devedor com endereço inválido fica `ERRO` ("endereço inválido — aguardando conferência") na qualificação de devedor, **não segue** para as dívidas e o serviço publica `…invalidou.enderecos-devedores.0` para a `demanda` abrir a conferência. Ao concluir a demanda, o BPMN chama o **reprocessamento do devedor** na execução original (RF-04), que reabre a execução | inventário P26; Q-09, decisão de 28/09/2026; E-17 |
| RF-07 | Reprocessar **kit `INVALIDO`** (F-011 RF-11): volta à qualificação de dívida com **as mesmas dívidas** (nunca novas), menos as que o procurador excluir (ex.: as com `AVISO`), sempre revalidadas; o kit antigo fica `REPROCESSADO` e o reagrupamento cria kit novo ligado a ele | `ARCHITECTURE.md` §7.1 (A22) |
| RF-08 | **Desmontar** kit `INVALIDO`: dívidas `DESCARTADO` com motivo "kit desmontado pelo procurador", reserva liberada, kit `DESCARTADO` | `ARCHITECTURE.md` §7.1; inventário P21 |
| RF-09 | **Reprocessar compensação** de kit `COMPENSACAO_PENDENTE`: repete a exclusão do processo; é a única ação desse kit | `ARCHITECTURE.md` §7 (A22) |
| RF-10 | Notificar a falha da compensação (kit `COMPENSACAO_PENDENTE`) | `ARCHITECTURE.md` §7 |
| RF-12 | Em execução com **lista explícita** de devedores ou dívidas (critérios ou CSV), o devedor ou a dívida informada e não selecionada — no fluxo ou filtrado pela consulta (F-005 RF-10, F-006 RF-12) — fica **visível no sumário** e, quando a dívida pertencia a um kit, **no kit**, com o motivo (ex.: "1 de 3 dívidas informadas descartada: já ajuizada no processo P1") | Q-11, decisão de 28/09/2026 |
| RF-11 | Kit `INVALIDO` de execução encerrada é **desfeito e descartado** quando outra execução reserva alguma de suas dívidas: kit e itens `DESCARTADO` com motivo "dívida tomada pela execução X". Verificação no serviço de reserva, 1 consulta por bloco (índice `(tenant, divida_id)`); concorrência resolvida pela restrição única da reserva e pelo `@Version` do kit | `ARCHITECTURE.md` §7.1; decisão de 27/09/2026 |

## Regras de negócio
- **RN-01** — Só devedor com `ERRO` e kit `INVALIDO` são reprocessáveis. **Dívida sozinha não é reprocessável**: dívida com erro pertence a um kit.
  Kit que falhou na geração da cobrança não é reprocessado: o retry automático e a compensação são da F-012, e o kit compensado é final.
- **RN-02** — Todo descarte de item tem motivo tipado, que alimenta o sumário.
- **RN-03** — A prévia não cria execução, reserva nem efeito externo.
- **RN-04** — Consultas respeitam isolamento de tenant e mascaramento de destino (`ARCHITECTURE.md` §11.2).
- **RN-05** — O reprocessamento reabre a execução original (F-003 RF-11) e volta a reservar as dívidas. Ao terminar, a execução liberou as reservas (F-003 RN-05), então outra execução pode ter pegado alguma: dívida em outra execução ativa sai `DESCARTADO` com motivo "dívida em outra execução" (F-003 RN-03).
- **RN-06** — Reprocessar kit sempre revalida as dívidas: no intervalo, outro processamento pode tê-las ajuizado.
- **RN-07** — Reprocessar dívida com `AVISO` não surte efeito ("não há o que fazer"); por isso o procurador pode excluí-las ao reprocessar o kit.
- **RN-08** — Privilégio: quem pode iniciar a execução pode reprocessar devedor e kit, desmontar e reprocessar a compensação.
- **RN-10** — Ter kit significa que o devedor passou pela qualificação com sucesso; por isso devedor com kit não volta à qualificação de devedor (Q-14). Falha na atualização do devedor para o devedor em `ERRO` antes das dívidas (F-005 RN-08), então ele nunca tem kit e é sempre reprocessável.
- **RN-09** — Desfazer o kit (RF-11) mantém a aba de kits inválidos só com kits reprocessáveis; a proteção contra duplicidade continua sendo a reserva e a revalidação (RN-05, RN-06).

## Critérios de aceite
### CA-01 — Sumário explica cada descarte (RF-01, RF-02)
- **Dado** uma execução concluída com itens descartados por motivos diferentes
- **Quando** o sumário é consultado
- **Então** os totais por motivo somam o total de descartes e cada item tem o motivo tipado

### CA-02 — Consultas paginadas (RF-03)
- **Dado** uma execução com mais itens que o tamanho de página
- **Quando** os itens são consultados
- **Então** a resposta é paginada e só traz dados do tenant do usuário

### CA-03 — Kit compensado não tem ação (RN-01)
- **Dado** um kit `DESCARTADO` após compensação bem-sucedida
- **Quando** o procurador consulta o kit
- **Então** nenhuma ação de reprocessar ou desmontar está disponível

### CA-04 — Reprocessar um devedor (RF-04)
- **Dado** uma execução concluída com falhas em vários devedores
- **Quando** o reprocessamento é pedido para um devedor
- **Então** a execução volta a `PROCESSANDO` e só aquele devedor volta à qualificação de devedor

### CA-18 — Devedor com kit não é reprocessável (RF-04, RN-10)
- **Dado** um devedor com `ERRO` cujas dívidas formaram um kit `INVALIDO` na execução
- **Quando** o reprocessamento do devedor é pedido
- **Então** o pedido é recusado com exceção ("o devedor não pode ser reprocessado porque já existem kits gerados para ele"), nenhum kit é alterado e a ação disponível é reprocessar o kit

### CA-19 — Informados e não selecionados ficam visíveis (RF-12)
- **Dado** uma execução com lista explícita dos devedores A e B e das CDAs 101 e 102, em que B não tem dívida apta e a 102 já está ajuizada
- **Quando** o procurador consulta o sumário da execução e o kit de A
- **Então** o sumário mostra B com "sem dívidas aptas para a fase" e a 102 com "dívida já ajuizada", e o kit mostra "1 de 2 dívidas informadas descartada"

### CA-05 — Prévia sem efeito (RF-05, RN-03)
- **Dado** os critérios de um job
- **Quando** a prévia é solicitada
- **Então** a lista de dívidas é devolvida e nenhuma execução, reserva ou chamada de efeito é criada

### CA-06 — Endereço inválido (RF-06)
- **Dado** o parâmetro habilitado e um devedor com endereço inválido na validação
- **Quando** a execução o processa
- **Então** o devedor fica `ERRO` "endereço inválido — aguardando conferência", suas dívidas não são consultadas, nenhum kit é gerado para ele e o fato de endereço inválido é publicado

### CA-20 — Conclusão da conferência reprocessa o devedor (RF-06, RF-04)
- **Dado** a demanda de conferência concluída com o endereço corrigido
- **Quando** o BPMN chama o reprocessamento do devedor
- **Então** a execução original reabre e o devedor volta à qualificação de devedor

### CA-21 — Parâmetro desligado (RF-06)
- **Dado** o parâmetro desligado e um devedor com endereço inválido
- **Quando** a execução o processa
- **Então** vale a regra de endereço da F-005 (RF-08): descarte com motivo, sem demanda

### CA-07 — Reprocessar kit invalidado por falha na atualização (RF-07, RN-05, RN-06)
- **Dado** um kit `INVALIDO` com 10 dívidas, 2 com `ERRO` por falha na atualização de saldo porque o serviço estava fora, e o serviço já restabelecido
- **Quando** o procurador reprocessa o kit
- **Então** as 10 voltam à qualificação de dívida e são revalidadas; sem erro restante, o kit antigo fica `REPROCESSADO` e um kit novo com as 10 segue para gerar cobrança

### CA-08 — Reprocessar devedor com falha na atualização (RF-04)
- **Dado** um devedor com `ERRO` porque o `pessoa` estava fora, e o `pessoa` já restabelecido
- **Quando** o procurador reprocessa o devedor
- **Então** o devedor volta à qualificação de devedor, a atualização é refeita, as dívidas são consultadas de novo no `divida` e seguem o fluxo

### CA-09 — Reprocessar kit excluindo dívidas com aviso (RF-07, RN-07)
- **Dado** um kit `INVALIDO` com 6 dívidas, 1 com `AVISO`
- **Quando** o procurador reprocessa o kit excluindo a dívida com `AVISO`
- **Então** essa dívida sai `DESCARTADO` com motivo "excluída no reprocessamento" e as outras 5 voltam à qualificação de dívida

### CA-10 — Reprocessamento não traz dívidas novas (RF-07)
- **Dado** um kit `INVALIDO` com 4 dívidas e uma 5ª dívida do mesmo devedor que ficou apta depois
- **Quando** o kit é reprocessado
- **Então** só as 4 dívidas originais são reprocessadas

### CA-11 — Revalidação no reprocessamento (RF-07, RN-06)
- **Dado** um kit `INVALIDO` em que uma dívida foi ajuizada por outro processamento no intervalo
- **Quando** o kit é reprocessado
- **Então** essa dívida sai `DESCARTADO` com motivo "já ajuizada" e as demais seguem

### CA-12 — Desmontar kit (RF-08)
- **Dado** um kit `INVALIDO` com 3 dívidas
- **Quando** o procurador desmonta o kit
- **Então** as 3 ficam `DESCARTADO` com motivo "kit desmontado pelo procurador", as reservas são liberadas e o kit fica `DESCARTADO`

### CA-13 — Reprocessar compensação (RF-09, RF-10)
- **Dado** um kit `COMPENSACAO_PENDENTE`, com a falha notificada
- **Quando** o procurador reprocessa a compensação e a exclusão do processo tem sucesso (ou o processo já não existe)
- **Então** o kit e as dívidas ficam `DESCARTADO`, sem ação disponível

### CA-14 — Privilégio (RN-08)
- **Dado** um usuário sem o privilégio de iniciar execução
- **Quando** tenta reprocessar, desmontar ou reprocessar a compensação
- **Então** a ação é negada

### CA-15 — Dívida pega por outra execução antes do reprocessamento (RN-05, RN-06)
- **Dado** a execução A concluída com um kit `INVALIDO` de 5 dívidas e uma delas reservada pela execução B, ainda ativa
- **Quando** o procurador reprocessa o kit em A
- **Então** essa dívida sai `DESCARTADO` com motivo "dívida em outra execução" e as outras 4 voltam à qualificação de dívida

### CA-16 — Kit inválido desfeito quando outra execução pega uma dívida (RF-11, RN-09)
- **Dado** a execução A concluída com um kit `INVALIDO` de 10 dívidas
- **Quando** a execução B reserva uma dessas dívidas
- **Então** o kit em A e seus 10 itens ficam `DESCARTADO` com motivo "dívida tomada pela execução B", e o kit sai da aba de kits inválidos

### CA-17 — Corrida entre reprocessamento e outra execução (RF-11)
- **Dado** o mesmo kit e o procurador reprocessando A no mesmo instante em que B reserva uma das dívidas
- **Quando** as duas tentam reservar a dívida
- **Então** só uma fica com a reserva: se for B, o kit em A é desfeito; se for A, o item em B fica `DESCARTADO` ("dívida em outra execução")

## Fora de escopo
- Reprocessar dívida isolada ou a execução inteira (só devedor e kit, RN-01).
- Indicador mensal (P25): `NOT_REQUIRED` (só existia no `kitcobranca`, inativo). O descarte manual (P21) entra só como desmontar kit inválido (RF-08).
- Modo simulação (F-020).

## Dependências e pendências
- **Features:** F-012 (SAGA), F-003/F-004 (execução e contadores), F-006 (seleção, para a prévia)
- **Externas:** E-09 — telas no `frontng`; E-01 — prévia depende da API de seleção do `divida`
- **Decidido (28/09/2026):** Q-14 — devedor com kit não é reprocessável; o pedido é recusado com exceção, nenhum kit é desfeito (RF-04, RN-10)
- **Em aberto:** Q-13 decidida (28/09/2026): a notificação tem kit, e reprocessar/desmontar kit `INVALIDO` vale também na notificação; contrato da demanda de conferência levantado (Q-09, 28/09/2026): a `demanda` precisa assinar o fato deste serviço e chamar o reprocessamento aqui (E-17)
