# Requisitos técnicos e funcionais para sistema de cobrança de dívida ativa de PGEs e PGMs: judicial, protesto e cobrança amigável por SMS/e-mail

Um sistema de cobrança de dívida ativa contratado hoje por uma PGE ou PGM precisa funcionar como uma esteira única e auditável. Nela, a cobrança amigável (notificação por SMS/e-mail com guia/PIX e parcelamento) e o protesto da CDA vêm primeiro. O ajuizamento eletrônico via MNI/PJe/e-SAJ fica como última etapa, para os créditos que tenham chance real de recuperação. A razão não é apenas de gestão: desde a Resolução CNJ nº 547/2024, ajustada em 2025 e 2026, a execução fiscal depende de prévia tentativa de solução administrativa e de prévio protesto, salvo justificativa. O sistema precisa, portanto, *gerar e guardar a prova* dessas etapas.

## TL;DR

- **Sequência obrigatória na prática:** notificação/solução administrativa (art. 2º da Res. CNJ 547/2024) → protesto da CDA (art. 3º, com dispensas) → execução fiscal (Lei 6.830/1980). O sistema deve registrar cada etapa como evidência juntável à petição inicial. Sem isso, a execução corre risco de extinção por falta de interesse de agir (Tema 1184/STF).
- **Núcleo funcional mínimo:** integração com o sistema de arrecadação/dívida ativa; cálculo de atualização; emissão de guia com código de barras e PIX; parcelamento/transação; régua de notificação multicanal com comprovação de entrega; remessa, confirmação, retorno, desistência e cancelamento de protesto via CRA/IEPTB; negativação em birôs; geração em lote de petição inicial + CDA; protocolo via MNI; recebimento de intimações; controle de prazos e de prescrição intercorrente; painéis gerenciais.
- **Requisitos não funcionais decisivos:** base legal LGPD de obrigação legal/política pública (art. 7º, II/III, e art. 23), e não consentimento; minimização dos dados nas mensagens; contratos de operador com gateways e birôs; certificado ICP-Brasil; trilhas de auditoria; comunicação de incidentes à ANPD; mensagens anti-golpe sem links encurtados. O mercado (Softplan SAJ Procuradorias, Attus/Attornatus, soluções de ERP municipal, Serasa/Boa Vista) cobre partes da esteira. Recomenda-se exigir prova de conceito (POC) com sorteio de funcionalidades.

## Key Findings

1. **O marco normativo mudou o desenho do sistema.** O STF (RE 1.355.208, Tema 1184, julgado em 19/12/2023) e a Resolução CNJ nº 547/2024 condicionaram o ajuizamento a duas providências prévias. A primeira é a tentativa de conciliação ou solução administrativa; a "notificação do executado para pagamento antes do ajuizamento" já configura solução administrativa (art. 2º, §2º). A segunda é o protesto do título, "salvo por motivo de eficiência administrativa, comprovando-se a inadequação da medida" (art. 3º). O próprio CNJ esclarece, em versão "linguagem simples", que o valor de R$ 10 mil **não é piso de ajuizamento**; é um dos critérios para extinguir execuções já em curso. O protesto é dispensado quando a dívida foi comunicada a serviços de proteção ao crédito, averbada em órgão de registro ou quando há indicação de bens à penhora.
2. **A Resolução CNJ nº 689/2026 (8/7/2026, publicada em 14/7/2026) impõe novas exigências técnicas.** Ela define "movimentação processual útil" (citação, intimação, constrição de bens, conforme o art. 921, §4º-A, do CPC). Os tribunais devem intimar o exequente em execuções com mais de 15 anos ou suspensas há mais de 6 anos, com prazo de 90 dias para manifestação. A resolução prevê identificação e alertas automáticos nos sistemas judiciais e permite incluir novos créditos de IPTU/IPVA na execução já ajuizada (art. 4º-A). O sistema da Procuradoria precisa responder em massa a essas intimações e montar pedidos incidentais de inclusão de CDAs.
3. **O protesto de CDA é pacífico juridicamente.** O parágrafo único do art. 1º da Lei 9.492/1997, incluído pela Lei 12.767/2012, inclui as CDAs entre os títulos protestáveis. O STF, na ADI 5135 (2016), fixou que o protesto de CDA "constitui mecanismo constitucional e legítimo" e não é sanção política. O STJ admite o protesto no Tema repetitivo 777 (REsp 1.686.659).
4. **O protesto tem eficiência comprovada.** Segundo notícia do Serpro sobre o Sistema de Protesto de CDA desenvolvido para a PGFN, "a arrecadação financeira a partir do protesto chega a 19% nos três primeiros meses de cobrança, um índice nove vezes maior do que o resultado obtido pela execução fiscal, que alcança aproximadamente 2%". Em notícia de 2016, a PGFN informou que, de março de 2013 a outubro de 2015, o índice chegou a 19,2%, o equivalente a 167.219 inscrições e R$ 728.260.828,54. Os considerandos da própria Res. CNJ 547/2024 citam as Notas Técnicas nº 06/2023 e 08/2023 do Núcleo de Processos Estruturais e Complexos do STF, segundo as quais "o custo mínimo de uma execução fiscal, com base no valor da mão de obra, é de R$ 9.277,00". O voto do Min. Luís Roberto Barroso no Ato Normativo 0000732-68.2024.2.00.0000 (CNJ, 20/02/2024) registra que "levantamento por amostragem do CNJ concluiu que mais da metade (52,3%) das execuções fiscais tem valor de ajuizamento inferior a R$ 10.000,00".
5. **Nos TRs, a integração com a CRA/IEPTB costuma vir por convênio, não por licitação.** Municípios e Estados firmam convênio com o IEPTB (por exemplo, IEPTB-SP/CRA) ou termo de cooperação (PGE-PB com IEPTB-PB/ANOREG). O software contratado precisa apenas implementar os arquivos e APIs da CRA.
6. **As contratações reais mostram três padrões de objeto:** (a) plataforma de procuradoria digital com IA integrada ao TJ e ao sistema de dívida ativa (PGM Joinville, PE 594/2022, 48 meses, cerca de R$ 2 milhões; PGM Ipojuca, PE 044/2022); (b) serviço de negativação/acionamento (PGM-SP com Serasa, R$ 1.392.000,00; PGE-PI com Boa Vista); (c) SaaS de mensageria de cobrança (por exemplo, pregão do CRECI/DF para disparo via API oficial do WhatsApp Business, sessão em 05/10/2026).

## Details

### (a) Base legal e normativa

| Norma / precedente | O que exige ou autoriza | Impacto no sistema |
|---|---|---|
| **Lei 6.830/1980 (LEF)** | Inscrição e requisitos do termo e da CDA (art. 2º, §§5º e 6º); petição inicial instruída com a CDA, que pode ser gerada por processo eletrônico (art. 6º); citação (art. 8º); suspensão e prescrição intercorrente (art. 40) | Validação automática dos requisitos da CDA; geração de petição + CDA; controle do prazo do art. 40 |
| **CTN, arts. 201-204** | Requisitos do termo de inscrição (art. 202); presunção de liquidez e certeza (art. 204) | Regras de validação antes da inscrição, do protesto e do ajuizamento |
| **CTN, art. 174** (alterado pela LC 208/2024, conforme nota técnica da CNM) | Causas interruptivas da prescrição, incluindo o protesto | Controle de prescrição com marcação da data do protesto |
| **CTN, art. 198, §3º, II** | Não é vedada a divulgação de informações sobre inscrições em dívida ativa | Dá suporte a protesto, negativação e notificações; demais dados fiscais seguem sob sigilo |
| **Lei 9.492/1997 + Lei 12.767/2012** | CDA protestável; intimação pelo tabelião (arts. 14-15), tríduo, edital | Módulo de protesto; enriquecimento de endereço/telefone para a intimação |
| **ADI 5135 (STF) e Tema 777 (STJ)** | Constitucionalidade e legalidade do protesto de CDA | Segurança jurídica para a remessa em massa |
| **Tema 1184 (STF) + Res. CNJ 547/2024, 617/2025 e 689/2026** | Prévia solução administrativa e prévio protesto; extinção de execuções de baixo valor paradas; novas intimações e automações | Dossiê probatório pré-ajuizamento; triagem de execuções antigas; resposta em lote |
| **Provimento CNJ 87/2019** (CENPROT) e Código Nacional de Normas (Prov. CNJ 149/2023) | Centrais eletrônicas de protesto | Integração com a CENPROT/CRA |
| **Res. Conjunta CNJ/CNMP 3/2013 (MNI)** | Padrão nacional de interoperabilidade; o schema do MNI 3.0 inclui CDA | Ajuizamento e intimações via web service |
| **LGPD (Lei 13.709/2018)** | Arts. 6º, 7º, 23, 26, 46-49 e 52 | Governança de dados nas três modalidades |
| **Leis locais** | Parcelamento, transação, honorários de protesto e valores mínimos (por exemplo, Lei SP 17.324/2020; Lei PR 18.292/2014) | Parametrização de regras por ente |

Cabe um alerta interpretativo: parte da jurisprudência afasta o CDC da relação tributária, porque o contribuinte não é consumidor. É o caso, por exemplo, da negativa de devolução em dobro do art. 42, parágrafo único, em repetição de indébito tributário. Mesmo assim, o padrão do art. 42 (vedação de exposição ao ridículo, constrangimento ou ameaça) deve ser tratado como **boa prática obrigatória na régua**. Ele coincide com os princípios da Administração (moralidade, proporcionalidade), e a cobrança vexatória pode gerar responsabilidade civil do Estado. O CDC incide diretamente quando a Procuradoria cobra créditos não tributários ligados a relações de consumo, ou quando empresas privadas contratadas atuam na cobrança.

### (b) Requisitos funcionais por modalidade

#### Módulos transversais (base comum)

- **Carga e integração de débitos:** importação por web service/API ou arquivo do sistema de arrecadação (SEFAZ/Secretaria de Finanças/órgãos de origem de multas). A integração deve ser bidirecional: pagamentos, parcelamentos, cancelamentos, suspensões e decisões administrativas voltam em tempo quase real. A Softplan, por exemplo, afirma integrar-se a "mais de 30 sistemas de dívida ativa".
- **Cadastro de CDA e devedor:** CDA com todos os requisitos do art. 2º, §5º, da LEF e do art. 202 do CTN; corresponsáveis; múltiplos endereços, telefones e e-mails, com origem e data de cada dado; agrupamento por devedor para reunião de dívidas.
- **Cálculo:** motor parametrizável de correção monetária, juros, multa, encargos legais e honorários (incluindo "honorários de protesto", como na PGE-PR), com índices por ente e por período, memória de cálculo e simulação por data.
- **Guias:** DAM/DAE/DAR/GR com código de barras e **PIX QR Code dinâmico**, conciliação automática da baixa e segunda via no portal do contribuinte.
- **Parcelamento e transação:** regras por lei/edital (por exemplo, a transação da PGM-SP com até 95% de desconto em juros e multa ou até 120 parcelas), termo eletrônico de adesão, controle de rompimento e retomada automática da cobrança.
- **Higienização e enriquecimento de dados:** validação de endereços na base dos Correios e enriquecimento de contatos. Essas exigências aparecem literalmente no edital de Ipojuca (itens 13.1.3 e 13.1.4) e no TR de Joinville (item 37).
- **Segmentação/rating da carteira:** classificação por probabilidade de recuperação, "grandes devedores" (item 147 do TR de Joinville), falência/recuperação judicial (item 86) e seleção automática da via de cobrança mais eficiente.
- **Workflow e BPMN:** filas de trabalho por perfil, automações configuráveis e modelos de documentos (item 5 de Joinville).
- **Painéis e relatórios:** estoque por fase, arrecadação por canal (SMS, protesto, judicial), taxa de conversão da régua, recuperação por safra, aging, prescrição iminente e custo por real recuperado.

#### 1. Cobrança judicial (execução fiscal)

- **Triagem de ajuizabilidade (gate CNJ 547):** o sistema bloqueia o ajuizamento sem (i) notificação prévia comprovada ou tentativa de conciliação e (ii) protesto efetivado ou justificativa formal de dispensa (negativação, averbação ou indicação de bens). A Softplan oferece função de "impedimento de ajuizamento" e "análise automatizada dos requisitos para execução fiscal".
- **Pesquisa patrimonial prévia:** registro de indícios de bens (imóveis, veículos, atividade econômica) para demonstrar a utilidade do ajuizamento e indicar bens na inicial, o que também dispensa o protesto.
- **Geração em lote de "kits de ajuizamento":** petição inicial + CDA em PDF assinado. Há fornecedores com lotes de até 50 petições por envio (Alternativa Soluções).
- **Protocolo eletrônico:** via MNI (web service SOAP, autenticação por certificado ICP-Brasil) para PJe e demais sistemas, e via integração específica para e-SAJ, eproc e Projudi, "retornando os ajuizamentos para o sistema da Dívida Ativa" (Ipojuca, item 13.1.5).
- **Recebimento de citações e intimações:** captura automática via MNI e Domicílio Judicial Eletrônico, com classificação por IA, distribuição a procuradores e contagem de prazos processuais.
- **Acompanhamento processual:** eventos do tribunal e da SEFAZ sincronizados (pagamento → pedido de extinção; parcelamento → pedido de suspensão), com geração automática da manifestação.
- **Gestão do art. 40 da LEF e da Res. 689/2026:** alertas de suspensão, arquivamento e prescrição intercorrente; resposta em lote às intimações para processos com mais de 15 anos ou suspensos há mais de 6; pedidos de inclusão de novos créditos de IPTU/IPVA na execução em curso (art. 4º-A).
- **Penhora e constrição:** registro de pedidos de SISBAJUD/RENAJUD/CNIB e de leilões (a PGM-Rio publica imóveis penhorados em leilão com emissão de guias).

#### 2. Cobrança administrativa via protesto extrajudicial

- **Seleção automática de CDAs protestáveis por regras**, com bloqueios obrigatórios: CDA prescrita, CDA já ajuizada (salvo extinção sem mérito), crédito com exigibilidade suspensa e devedor sem CPF/CNPJ válido. Essas regras aparecem, por exemplo, em decreto municipal de Ouroeste/SP.
- **Troca eletrônica de arquivos com a CRA/IEPTB**, nos cinco fluxos padronizados: **remessa** (apresentante → CRA); **confirmação** (protocolização pelo tabelionato); **retorno** (pago, protestado, retirado, irregular, sustado judicialmente); **desistência** (antes da lavratura); **cancelamento** (autorização/anuência após o protesto). A Cenprot-SP também oferece APIs.
- **Carta de anuência eletrônica automática** após o pagamento ou a adesão ao parcelamento. Referência de mercado: PGE-PR, que envia a anuência ao tabelionato em até 48 horas / 2 dias úteis da confirmação do pagamento, dispensando o devedor de imprimir documentos.
- **Conciliação financeira:** repasse pelo tabelião dos valores pagos no tríduo; baixa na dívida ativa; controle dos emolumentos (em regra a cargo do devedor; em muitos convênios, sem ônus antecipado para o ente).
- **Portal do devedor protestado:** consulta por CPF/CNPJ e emissão de guias do débito e dos honorários (modelo do "Sistema de Consulta e Emissão de Guias para Dívida Ativa Protestada" da PGE-PR).
- **Marcação da data do protesto** para controle de prescrição e como prova do art. 3º da Res. 547.
- **Negativação em birôs (Serasa, Boa Vista/SCPC, SPC):** inclusão, exclusão em prazo e retorno de status. É alternativa ou complemento ao protesto e hipótese de dispensa do protesto na Res. 547.

#### 3. Cobrança amigável (SMS, e-mail e outros canais)

- **Régua configurável** por perfil de devedor, tributo, faixa de valor e fase, com gatilhos: inscrição (a PGM de São Bernardo, por exemplo, notifica para pagamento em 5 dias, com edital de 15 dias como alternativa); pré-protesto (a PGE-MT avisa por SMS que o CPF/CNPJ será negativado em 10 dias); pré-ajuizamento; e aberturas de programas de parcelamento ou transação.
- **Multicanal com fallback:** SMS → e-mail → carta (a PGE-MT passa para e-mail e correspondência quando o SMS não localiza o contribuinte). Canais opcionais: WhatsApp Business API oficial, URA/voz.
- **Conteúdo mínimo:** identificação do órgão, número da CDA, valor atualizado, vencimento, meios oficiais de pagamento (linha digitável/PIX copia-e-cola), canal de atendimento e consequências (protesto/negativação/execução), sem linguagem intimidatória.
- **Comprovação de entrega:** status de entrega da operadora (DLR) para SMS; registro de entrega, abertura e bounce para e-mail; carimbo de tempo e armazenamento do conteúdo enviado. Isso compõe o **dossiê probatório** da "notificação prévia" do art. 2º, §2º, da Res. 547.
- **Autoatendimento:** link apenas para domínio oficial (gov.br ou domínio do ente) com autenticação (gov.br/CPF + validação), emissão de guia/PIX e adesão a parcelamento on-line.
- **Opt-out de canal** (sem extinguir a cobrança, que tem base legal) e gestão de contatos inválidos.
- **Métricas:** entrega, abertura, conversão em pagamento e parcelamento, arrecadação por mensagem e custo por disparo.

### (c) Requisitos não funcionais

**LGPD e sigilo**
- **Base legal:** tratamento para cumprimento de obrigação legal e execução de políticas públicas (art. 7º, II e III) e "para o atendimento de sua finalidade pública... com o objetivo de executar as competências legais" (art. 23). O consentimento não é a base adequada, já que a cobrança não depende da vontade do titular.
- **Transparência (art. 23, I):** publicar no portal da Procuradoria as hipóteses de tratamento, os canais usados (SMS/e-mail), os operadores e o encarregado. PGDF e PGE-MA já mantêm páginas explicando as mensagens.
- **Minimização:** no SMS, que pode ser lido por terceiros com acesso ao aparelho, evitar detalhar natureza da dívida, origem ou dados de terceiros; preferir "débito inscrito em dívida ativa" + número da CDA + valor + canal oficial. O art. 198, §3º, II, do CTN libera a informação da inscrição, mas não outros dados fiscais.
- **Operadores:** contratos com gateways de SMS/e-mail, birôs e fornecedores de software contendo cláusulas LGPD (instruções do controlador, vedação de uso secundário e de enriquecimento reverso, subcontratação só com autorização, eliminação ao fim do contrato, auditoria). O compartilhamento com birôs e cartórios (que são "Poder Público" para a LGPD, art. 23, §4º) deve seguir o art. 26.
- **RIPD** (relatório de impacto) para régua, negativação e enriquecimento de dados; registro das operações (art. 37).
- **Incidentes:** plano de resposta e comunicação à ANPD e aos titulares (art. 48 e regulamento de comunicação de incidentes da ANPD de 2024). Órgãos públicos não recebem multa da LGPD (art. 52, §3º), mas estão sujeitos às demais sanções e à responsabilização de agentes.

**Segurança da informação**
- Autenticação forte (MFA; certificado ICP-Brasil A1/A3 para assinatura e protocolo MNI), perfis por papel e segregação de funções (quem inscreve ≠ quem cancela ou concede desconto).
- Criptografia em trânsito (TLS 1.2+) e em repouso; trilhas de auditoria imutáveis para cálculo, desconto, cancelamento, anuência e envio; retenção de logs compatível com os prazos prescricionais.
- **Anti-fraude e anti-phishing:** remetente identificável (short code/sender ID dedicado; SPF, DKIM e DMARC no e-mail); sem links encurtados; PIX sempre com recebedor identificado como o ente. A própria Cenprot-SP alerta publicamente para mensagens falsas em seu nome, e golpes com "dívida ativa" são recorrentes.
- Hospedagem em nuvem com dados no Brasil (ou justificativa), backup, plano de continuidade, testes de intrusão periódicos e aderência a frameworks (ABNT NBR ISO/IEC 27001/27701; controles do PPSI/GSI para entes que o adotem).

**Integração e interoperabilidade**
- Arrecadação/dívida ativa (REST/SOAP ou mensageria, com idempotência); tribunais (MNI, PJe, e-SAJ, eproc, Projudi; Domicílio Judicial Eletrônico); CRA/CENPROT (layouts de remessa, confirmação, retorno, desistência e cancelamento; APIs); birôs (Serasa, Boa Vista, SPC); gateways de SMS/e-mail/WhatsApp com webhooks de status; PSP bancário/PIX (API PIX do BACEN) para baixa instantânea; gov.br para login.

**Desempenho e operação**
- Processamento em lote dimensionado para o estoque. Como referência, a Subprocuradoria Fiscal e Tributária da PGM de Florianópolis informa em sua página o "ajuizamento e acompanhamento dos aproximados 150.000 (cento e cinquenta mil) processos de execuções fiscais em trâmite no Poder Judiciário Catarinense", com dívida ativa ajuizada acima de R$ 450 milhões. O sistema deve suportar cargas diárias, milhares de protocolos por lote e disparos em massa com controle de vazão.
- Disponibilidade contratual (SLA) de 99,5% ou mais para o portal do contribuinte; janela de manutenção; tempo de resposta; suporte N1-N3 com prazos por severidade.
- Portabilidade: exportação integral dos dados em formato aberto ao fim do contrato, para evitar lock-in; migração inicial do legado.

### (d) Exemplos de mercado e editais

| Caso | Objeto e fatos | Requisitos relevantes |
|---|---|---|
| **PGM Joinville/SC – PE 594/2022** (SEI 22.0.232755-4, UASG 453230) | Solução de gestão da execução fiscal e do contencioso com IA, integrada a TJSC, TRF4, TRT12, SEI e à dívida ativa; 48 meses; contrato de cerca de R$ 2 milhões com a Attus/Attornatus; implantação a partir de jan/2023 | Anexo de requisitos com 237 itens; BPMN e petições automáticas; validação de endereços; rastreio de automações; peticionamento em lote em falências; **POC com 10 atividades sorteadas em sessão pública** |
| **PGM Ipojuca/PE – PE 044/PMI-PGM/2022** | Sistema com IA para execução fiscal e contencioso, integrado ao TJPE e à dívida ativa | Integração com a dívida ativa; higienização via Correios; enriquecimento; ajuizamento via MNI com retorno à dívida ativa; intimações via MNI; classificação por IA (itens transcritos em impugnação de concorrente — conferir no original) |
| **PGM Maceió/AL – PE 241/2022** | Software de gestão de processos da PGM, com migração, implantação, treinamento e 142 licenças | Exigência de POC |
| **PGM São Paulo – Contrato 003/PGM/2024** | Serasa: registro de preços para negativação de débitos de dívida ativa; R$ 1.392.000,00; vigência 24/02/2024-24/02/2025 | Negativação como via complementar |
| **PGE-PI – 2024** | PMI 01/2024 (estudo de segmentação da carteira) seguido de dispensa para "acionamento de devedores..., enriquecimento de base de contatos e... negativação (SPC/SERASA)"; execução pelo Banco Boa Vista (Portaria GAB 15/2025) | Modelagem prévia da carteira antes de contratar |
| **PGE-PB** | Protesto eletrônico de CDA via CRA por termo de cooperação com IEPTB-PB/ANOREG | Protesto por convênio, sem licitação |
| **CRECI/DF – pregão (sessão 05/10/2026)** | SaaS de comunicações de cobrança via API oficial do WhatsApp Business | Modelo de contratação de mensageria como SaaS (menor preço global; licença e suporte mensais) |

**Fornecedores e funcionalidades típicas (benchmarking):**
- **Softplan – SAJ Procuradorias:** integração com tribunais e com mais de 30 sistemas de dívida ativa; módulo de Gestão de Cobranças com filtros de protesto digital, execução fiscal, envio de SMS, enriquecimento, inconsistências e parcelamento; geração de kits de ajuizamento; impedimento de ajuizamento; consulta de CDA com tabelionato e situação do protesto; posicionado para a Res. 547. Caso divulgado pelo próprio fornecedor: Barueri teria passado de 1.235 para 8.727 ajuizamentos em um ano (dado de marketing, não auditado).
- **Attus/Attornatus (Procuradoria Digital):** vencedora em Joinville (a ata da Prova de Conceito do PE 594/2022 registra a empresa como "ATTORNATUS PROCURADORIA DIGITAL LTDA"); foco em IA, automação e integração MNI.
- **ERPs tributários municipais** (por exemplo, Alternativa Soluções em Sistemas): petição + CDA em PDF e ajuizamento em lote de até 50 iniciais dentro do próprio sistema de arrecadação.
- **Birôs (Serasa, Boa Vista):** negativação, notificação ao devedor e enriquecimento de contatos, contratados à parte.
- **Serpro:** desenvolveu para a PGFN o sistema que seleciona inscrições com perfil de protesto e as envia automaticamente aos cartórios; serve de referência de arquitetura para grandes entes.
- **Plataformas de régua de cobrança do setor privado:** têm funcionalidades maduras (lembretes antes e depois do vencimento, confirmação de pagamento), mas precisam de adaptação às exigências públicas (LGPD pública, domínio oficial, prova de notificação).

## Recommendations

1. **Estruture o TR em lotes ou módulos com interfaces abertas:** (i) núcleo de dívida ativa e cálculo, se não houver; (ii) procuradoria digital/judicial; (iii) mensageria. O protesto via CRA deve entrar como requisito de integração do núcleo, somado ao convênio com o IEPTB. Assim se evita pagar por intermediação do protesto e se reduz o lock-in.
2. **Transforme a Res. CNJ 547/2024 e a 689/2026 em requisitos testáveis:** "o sistema deverá impedir o protocolo de execução fiscal sem registro de notificação entregue e de protesto efetivado ou dispensa fundamentada, anexando automaticamente as evidências à inicial". Inclua também a resposta em lote às intimações de processos antigos.
3. **Exija POC por sorteio** (modelo Joinville), cobrindo ao menos: carga da dívida ativa, cálculo, emissão de PIX, disparo com status de entrega, remessa e retorno com a CRA de homologação, e protocolo MNI em ambiente de teste do TJ.
4. **Crie um caderno LGPD próprio no TR:** papel de operador, RIPD, minimização das mensagens, vedação de uso secundário e de revenda de dados enriquecidos, notificação de incidente em até 24-48 horas ao controlador, eliminação ou devolução ao término e auditoria.
5. **Adote uma política anti-golpe na régua:** remetente oficial fixo, nenhum link encurtado, divulgação no portal de modelos das mensagens verdadeiras e orientação para conferir a dívida pelo portal oficial.
6. **Meça resultado, não volume:** pagamento por disparo entregue (e não enviado); indicadores contratuais de conversão; relatório de recuperação por canal para recalibrar a régua e o limite de ajuizamento em lei local.
7. **Se a remuneração for por êxito**, submeta o modelo à análise jurídica: a cobrança de dívida ativa é atividade típica da Procuradoria, e a terceirização deve se limitar a suporte tecnológico e operacional.

## Caveats

- Não obtive o texto integral de nenhum TR de PGE/PGM. Os requisitos literais de Ipojuca vêm de impugnação feita por empresa concorrente, e os de Joinville vêm da ata da POC. É preciso conferir nos originais. Não localizei TR de PGE-SP, PGE-RJ ou PGE-MG para esse objeto específico.
- Os dados de eficiência do protesto (19% nos três primeiros meses, segundo o Serpro, e 19,2% entre março de 2013 e outubro de 2015, segundo a PGFN) são antigos e se referem à dívida ativa da União. Os números de Barueri são material de marketing do fornecedor.
- As alterações da Resolução 689/2026 são recentes (julho de 2026), e as especificações técnicas do CNJ para os tribunais estavam previstas para até 90 dias após a publicação. Os fluxos podem mudar.
- A alteração do art. 174 do CTN pela LC 208/2024 (protesto como causa interruptiva da prescrição) foi confirmada apenas indiretamente, via nota técnica da CNM. Valide a redação vigente.
- A inaplicabilidade do CDC às relações tributárias é majoritária na jurisprudência consultada, mas não afasta a responsabilidade civil do Estado por cobrança abusiva nem a incidência do CDC em créditos não tributários de natureza consumerista.