# Especificação de Requisitos de Produto — Pipeline de RAG
## Assistente de Atendimento NovaTech (Logística)

**Versão:** 5.0
**Data:** 05/06/2026
**Base documental:** novatech-jornada-rag-v4.md + Anexo A — Documentação Simulada NovaTech
**Status:** Rascunho para validação com stakeholders

> **O que mudou nesta versão:** A v4 documentou três gaps pendentes (P8, P9, P10) sem input externo disponível. A v5 resolve os dois gaps resolvíveis com a documentação existente: (Gap 2) adicionada regra de comportamento para quando o tier do cliente não tem cobertura para o item consultado — o assistente cita o que o SLA define para o tier informado e nunca omite a resposta por ausência de cobertura parcial; (Gap 3) adicionada regra de comportamento para quando a data de abertura do chamado está ausente — o assistente solicita a data explicitamente antes de responder qualquer pergunta que dependa da versão da PROC-042, sem assumir nenhuma versão como padrão. Nenhum requisito existente foi alterado.

---

## Visão geral

Este documento define os requisitos funcionais do pipeline de RAG que alimentará o assistente de atendimento da NovaTech. O assistente apoia 45 atendentes que lidam com cerca de 320 chamados por dia, dos quais 60% envolvem consulta a documentos internos.

A documentação atual da NovaTech apresenta **cinco contradições diretas entre documentos** e **cinco gaps onde informação operacional relevante está ausente ou existe apenas no FAQ informal** — sem normativo correspondente. Todos os requisitos desta spec foram projetados para que o assistente lide corretamente com esses casos.

---

## 1. Fontes de dados — o que indexar e o que não indexar

### O que deve ser indexado

**SharePoint corporativo (~800 documentos)**
Contém os normativos oficiais: POL-001 (Política de Devolução), PROC-042 v1 e v2 (Frete Especial), SLA-2024 (Tabela de SLA por Tipo de Cliente) e outros. É a principal fonte de verdade da base. Documentos vigentes entram com peso total; documentos "em revisão" entram com peso reduzido e alerta obrigatório; documentos revogados são removidos — não apenas marcados.

*Situação de atenção imediata:* a PROC-042 v1 (emitida em março/2023) e a PROC-042 v2 (emitida em novembro/2023) coexistem no SharePoint **sem que nenhum dos dois esteja marcado como obsoleto**. Ambos precisam ser indexados com status distintos e regra de aplicação explícita (detalhada na seção 2). Nenhum dos dois pode ser removido unilateralmente.

**Confluence — wiki interna (~400 páginas)**
Deve ser indexada com aproveitamento do histórico de versões nativo do Confluence para extração automática de data de vigência. Páginas sem revisão há mais de 6 meses devem ser sinalizadas para avaliação antes da indexação.

**Planilhas da pasta de rede (tabelas mensais de frete)**
A PROC-042 v1 e v2 referenciam a tabela base de fretes em `\\novatech-fs\comercial\tabelas\frete-base-AAAAMM.xlsx`. Esse arquivo é atualizado mensalmente e não possui webhook nativo — o monitoramento é feito por comparação de hash com ciclo diário recomendado. Sem essa planilha, o assistente não consegue calcular o valor final do frete; tem apenas os multiplicadores, não o valor base.

**FAQ operacional (~50 itens, dos quais 47 estão no documento)**
Deve ser indexado com peso reduzido e marcação permanente de fonte não oficial. O FAQ nunca deve ser priorizado quando houver normativo conflitante. O FAQ-atendimento da NovaTech não tem responsável formal, não tem processo de revisão periódico e mistura conhecimento preciso com práticas informais não validadas — o que reforça a necessidade de tratamento como fonte secundária.

*Exemplos diretos do FAQ atual que ilustram o risco:*
- **Item 45** afirma que desconto automático se aplica a clientes com "mais de 10 fretes especiais por mês". A PROC-042 v2 (normativo vigente) define o limiar em 8 fretes. O FAQ está desatualizado neste ponto.
- **Item 8** orienta o atendente a "usar a v2 na dúvida, mas se o cliente reclamar pode ser que o contrato ainda esteja na tabela antiga" — uma heurística informal que não substitui a regra objetiva de aplicação por data de chamado.
- **Item 27** define parâmetros de alerta de rastreamento (Norte: até 10 dias úteis; Sul/Sudeste: alerta após 3 dias) sem respaldo em normativo — informação útil, mas sem fonte oficial.

### O que NÃO deve ser indexado — e por quê

**PROC-042 v1 como documento revogado**
A PROC-042 v1 **não pode ser tratada como revogada e removida**, mesmo que a v2 seja a versão atual para novos chamados. A seção 5 da PROC-042 v2 estabelece explicitamente que chamados abertos antes de 01/12/2023 ainda em processamento devem usar os multiplicadores da v1. Enquanto existirem chamados nessa condição — ou enquanto houver contratos que referenciem a v1 (ver P5 nas Perguntas em Aberto) — a v1 permanece obrigatória na base com status "coexistente".

**Comunicados temporários e e-mails internos**
Não representam posição oficial da empresa. Trocas de mensagem, decisões pontuais combinadas informalmente e avisos operacionais sem processo de aprovação não devem alimentar o assistente — o risco de inconsistência com os normativos vigentes é alto e a rastreabilidade é impossível.

**Rascunhos de documentos não aprovados**
Especificamente relevante para a PROC-043 (Frete de Cargas Perigosas), citada tanto na PROC-042 v1 quanto na v2, atualmente em revisão pelo Compliance. Qualquer versão provisória em uso interno não deve ser indexada até publicação formal.

**Contratos individuais de clientes**
Podem conter cláusulas que divergem das políticas gerais. O FAQ-atendimento já sinaliza esse risco no item 22 (percentuais de seguro diferentes para contratos anteriores a 2023) e no item 8 (multiplicadores diferentes para contratos antigos). A lógica de exceção contratual não deve ser tratada automaticamente pelo assistente.

---

## 2. Como lidar com documentos contraditórios

A documentação atual da NovaTech apresenta cinco contradições identificadas entre documentos formais ou entre documentos formais e o FAQ. Cada uma tem um tratamento diferente.

---

### Contradição 1 — Multiplicadores regionais: PROC-042 v1 vs v2

**O problema:**
Os multiplicadores regionais são diferentes nas duas versões e nenhum documento está marcado como obsoleto no SharePoint.

| Região | PROC-042 v1 | PROC-042 v2 | Diferença |
|--------|-------------|-------------|-----------|
| Sul | 1,2 | 1,3 | +8,3% |
| Sudeste | 1,0 | 1,1 | +10% |
| Centro-Oeste | 1,3 | 1,4 | +7,7% |
| Nordeste | 1,4 | 1,5 | +7,1% |
| Norte | 1,6 | 1,8 | +12,5% |

Usar a versão errada para uma carga de alto valor resulta em cobrança incorreta ao cliente — abaixo (prejuízo para a NovaTech) ou acima (cobrança indevida com risco de contestação).

**Como o assistente deve se comportar:**
A seção 5 da PROC-042 v2 define a regra de transição: chamados abertos antes de 01/12/2023 → aplicar v1; chamados abertos a partir de 01/12/2023 → aplicar v2. Essa regra deve ser implementada como metadado `regra_aplicacao` em cada fragmento indexado e executada automaticamente pelo sistema com base na data de abertura do chamado. O atendente não decide qual versão aplicar — o sistema decide.

**Comportamento quando a data de abertura do chamado não está disponível no contexto:**
Se a data de abertura do chamado não for informada pelo atendente ou pelo sistema de chamados, o assistente não deve assumir nenhuma versão nem tentar inferir a data a partir de outros dados. Deve interromper a resposta e solicitar a informação explicitamente: *"Para informar o cálculo correto de frete especial, preciso da data de abertura do chamado. Qual é a data?"* O assistente só prossegue com a resposta após receber a data. Essa regra se aplica a qualquer pergunta que envolva multiplicadores regionais, fatores de peso ou prazo adicional da PROC-042 — ou seja, toda pergunta onde a versão correta do documento é determinante para a resposta. Aplicar a v2 como padrão na ausência da data seria incorreto para chamados históricos ainda em processamento; aplicar a v1 como padrão seria incorreto para chamados novos. A única resposta segura é solicitar a data.

*Exemplo concreto:* chamado aberto em 15/01/2024 com carga acima de 3.000kg com destino ao Norte. O assistente deve aplicar v2: multiplicador 1,8 × fator de peso 1,4 = 2,52. Se aplicar a v1: 1,6 × 1,5 = 2,4. A diferença de 5% num frete de alto valor é material.

---

### Contradição 2 — Fatores de peso: PROC-042 v1 vs v2

**O problema:**
Além dos multiplicadores regionais, os fatores de peso por faixa também mudaram entre as versões:

| Faixa de peso | PROC-042 v1 | PROC-042 v2 | Diferença |
|---------------|-------------|-------------|-----------|
| 500kg – 1.000kg | 1,0 | 1,0 | Nenhuma |
| 1.001kg – 3.000kg | 1,2 | 1,15 | -4,2% |
| Acima de 3.000kg | 1,5 | 1,4 | -6,7% |

A v2 reduziu os fatores de peso ao mesmo tempo em que aumentou os multiplicadores regionais. Aplicar o fator de peso da versão errada produz um resultado incorreto independente do multiplicador aplicado corretamente.

**Como o assistente deve se comportar:**
Mesma regra de transição da Contradição 1. Os fatores de peso e os multiplicadores regionais fazem parte do mesmo documento — a versão aplicada deve ser integral (v1 ou v2), nunca uma combinação de partes das duas versões.

---

### Contradição 3 — Prazo adicional de frete especial: PROC-042 v1 vs v2

**O problema:**
A v1 define prazo adicional de +2 dias úteis para frete especial. A v2 define +3 dias úteis. A diferença é explicitamente reconhecida na v2: *"anteriormente era +2 dias na versão anterior".*

**Como o assistente deve se comportar:**
Mesma regra de transição. Informar o prazo errado a um cliente Gold — cujo SLA de incidente crítico é de 30 minutos — pode gerar penalidade contratual. O prazo de entrega é tão crítico quanto o valor do frete.

---

### Contradição 4 — Limiar de desconto de volume: FAQ vs PROC-042 v2

**O problema:**
O FAQ (item 45) instrui o atendente a informar desconto automático para clientes com "mais de 10 fretes especiais por mês". A PROC-042 v2 (normativo vigente) define o limiar em 8 fretes, com duas faixas:
- A partir de 8 fretes/mês: 5% de desconto automático sobre o multiplicador regional
- Acima de 15 fretes/mês: 10% de desconto

O FAQ não apenas usa o limiar errado (10 em vez de 8) como ignora completamente a segunda faixa de desconto (15+).

**Como o assistente deve se comportar:**
A PROC-042 v2 é normativo vigente; o FAQ é fonte secundária não validada. Quando há conflito direto, o normativo prevalece. O assistente deve responder com os valores da PROC-042 v2 e ignorar a instrução do FAQ para este ponto. O FAQ deve ser sinalizado para atualização pela área responsável.

---

### Contradição 5 — Frete expresso para carga perigosa: FAQ vs ausência de normativo

**O problema:**
O FAQ (item 32) afirma que é possível enviar carga perigosa com frete expresso "com autorização do Compliance". Não existe nenhum documento formal (POL ou PROC) que defina esse processo, seus critérios, prazos ou responsáveis. A informação pode refletir uma prática informal não documentada.

**Como o assistente deve se comportar:**
Sem normativo correspondente, o assistente não deve confirmar nem negar a possibilidade. Deve informar que não localizou procedimento formal para essa situação e recomendar contato com o Compliance diretamente. Não deve citar o FAQ como fonte para autorizar ou orientar a operação.

---

### Regra geral para contradições sem regra conhecida

Se dois documentos oficiais contradizem um ao outro e não há metadado de `regra_aplicacao` que permita a escolha automatizada — situação diferente dos casos da PROC-042, que têm regra explícita de transição — o assistente não escolhe por conta própria. Sinaliza a contradição ao atendente, recomenda escalada para o supervisor e registra o caso no fluxo de feedback como "gap documentado" para que a área responsável defina hierarquia de aplicação.

> ⚠️ **Pendência P8-A — Fluxo de feedback não especificado nesta spec**
> O comportamento acima pressupõe um fluxo de feedback operacional (tipos de sinal, triagem de volume, notificação às áreas responsáveis). Esse fluxo está descrito em detalhe no documento novatech-jornada-rag-v4.md, seção 2.4, mas ainda não foi incorporado formalmente como requisito desta spec. A incorporação depende de validação com o Product Owner para confirmar se o fluxo descrito reflete como o feedback será capturado na interface do assistente. **Enquanto essa validação não ocorrer, o comportamento de registro de feedback permanece indefinido para fins de implementação.** Ver P8 na tabela de Perguntas em Aberto.

---

## 3. Comportamento esperado quando não há resposta na base

**Regra fundamental:** o assistente jamais deve inventar ou inferir prazos, valores, multiplicadores ou procedimentos que não estejam na documentação indexada. Essa regra é inegociável.

### Cenário 1 — Nenhum fragmento relevante encontrado

O assistente informa ao atendente que não localizou resposta com confiança suficiente na base documental e recomenda escalada para o supervisor. A mensagem deve ser direta: não se trata de resposta com ressalvas, mas de ausência de resposta.

*Caso concreto que pode gerar este cenário:* cliente pergunta sobre frete padrão (abaixo de 500kg). A PROC-042 v1 e v2 cobrem apenas frete especial (acima de 500kg). Não há documento na base que cubra frete padrão — o assistente deve sinalizar o gap, não extrapolar a lógica da PROC-042.

### Cenário 2 — Fragmentos encontrados, mas abaixo do limiar de confiança

Mesmo que existam fragmentos relacionados, se a pontuação de relevância ficar abaixo do limiar configurado, o comportamento é idêntico ao Cenário 1. Nenhuma informação incerta chega ao cliente.

### Cenário 3 — Resposta encontrada apenas no FAQ, sem normativo correspondente

O assistente fornece a informação com sinalização obrigatória: *"Essa informação vem do FAQ operacional, que não foi validado por Compliance."* O atendente decide se usa com a ressalva ou escala.

**Gaps atuais que se enquadram neste cenário:**

| Item do FAQ | Informação | Status |
|---|---|---|
| Item 38 | Carga danificada em trânsito: 48h para registro, fotos, laudo, envio para `sinistros@novatech.com.br` | Sem normativo correspondente. Principal candidato a formalização. |
| Item 3 | Devolução de carga perigosa: orientar para ramal 4500 — Gestão de Riscos | POL-001 menciona o ramal 4500, mas não há procedimento documentado sobre o que a Gestão de Riscos faz com esses casos. |
| Item 27 | Rastreamento parado: Norte → até 10 dias úteis; Sul/Sudeste → alerta após 3 dias | Sem normativo. Parâmetros úteis mas sem validação formal. |
| Item 22 | Seguro de carga: 0,3% para cargas padrão, 0,8% para cargas perigosas | Sem normativo. Contratos anteriores a 2023 podem ter percentuais diferentes. |

**Gap adicional — frete padrão abaixo de 500kg:**
Nenhum documento na base cobre o cálculo de frete padrão. Perguntas sobre este tema devem ser direcionadas ao Comercial. O assistente deve sinalizar explicitamente que esse conteúdo não está disponível na base documental atual.

### O que não é aceitável em nenhum cenário

- Responder com valores aproximados sem citar fonte
- Extrapolar regras de um contexto para outro (ex: aplicar a lógica do frete especial para frete padrão)
- Silenciar sobre a incerteza e entregar uma resposta como se fosse confiável
- Citar o FAQ como fonte equivalente a um normativo oficial

---

## 4. Requisitos de atualização — da publicação do documento à disponibilidade no assistente

### Detecção de mudança

| Fonte | Mecanismo | Frequência |
|---|---|---|
| SharePoint | Webhook automático | Tempo real, a cada alteração |
| Confluence | Webhook via API | Tempo real, a cada alteração de página |
| Pasta de rede (planilhas de frete-base) | Monitoramento por hash | Diário (recomendado) |

A pasta de rede não possui webhook nativo. A planilha `frete-base-AAAAMM.xlsx` — referenciada explicitamente na PROC-042 v1 e v2 — é o componente mais crítico para cálculo de frete e o mais vulnerável à desatualização silenciosa: uma planilha nova pode ficar até 24h sem ser detectada. A frequência real de atualização (mensal conforme planejado, ou ad hoc) precisa ser confirmada com a equipe responsável (ver P6 nas Perguntas em Aberto).

### Prazos máximos por tipo de atualização

| Tipo de atualização | Prazo máximo alvo |
|---|---|
| Documento novo ou versão revisada (SharePoint/Confluence) | 4 horas após detecção da mudança |
| Planilha de frete-base da pasta de rede | 24 horas após a atualização |
| Revogação de documento | 2 horas — remoção tem prioridade sobre novas indexações |
| Publicação da PROC-043 revisada pelo Compliance | Segue o fluxo padrão de classificação e validação (seção acima). O alerta de "em revisão" só é removido das respostas sobre carga perigosa após a suite de validação passar integralmente — não no momento da publicação pelo Compliance. Se a PROC-043 publicada incluir período de transição ou coexistência de versões, aplica-se o mesmo tratamento da PROC-042: revisão humana obrigatória, definição de `regra_aplicacao` e atualização dos casos de teste antes da indexação. |
| Promoção de item do FAQ a normativo oficial | Mesmo prazo de documento novo, após validação formal |

### Fluxo após a detecção

1. Detecção da mudança (webhook ou hash)
2. Classificação de status — automática para documentos com metadado explícito; revisão humana para casos ambíguos (ver abaixo)
3. Re-indexação com metadados obrigatórios atualizados
4. Execução da suite de validação com os casos de teste definidos na seção 4a
5. Se todos os testes passam → publicação com registro
6. Se qualquer teste falha → bloqueio da publicação + notificação à área responsável

### Casos que sempre exigem revisão humana antes da indexação

- Documentos sem metadado de status explícito (como a PROC-042 v1, que não possui indicação formal de vigência ou obsolescência)
- Coexistência de versões sem regra de transição documentada
- Documentos movidos de pasta sem indicação de status
- Páginas do Confluence sem revisão há mais de 6 meses
- Qualquer nova versão de documento que contradiga um normativo vigente sem revogar explicitamente a versão anterior

### Suite de validação — casos derivados da documentação real (seção 4a)

A suite mínima de testes deve cobrir os seguintes casos, derivados diretamente das contradições e gaps identificados:

| Caso de teste | Resposta esperada | Risco se errar |
|---|---|---|
| Desconto de volume: a partir de quantos fretes especiais? | 8 fretes/mês — 5% automático (PROC-042 v2, seção 4) | Alto — FAQ item 45 diz 10 |
| Chamado aberto em 15/01/2024: qual versão da PROC-042 aplicar? | v2 — posterior a 01/12/2023 (PROC-042 v2, seção 5) | Alto — v1 coexiste no SharePoint |
| Carga de 3.500kg destino Norte, chamado atual: qual o multiplicador composto? | Fator de peso 1,4 × multiplicador regional 1,8 = 2,52 (PROC-042 v2) | Alto — v1 daria 1,5 × 1,6 = 2,4 |
| Prazo adicional para frete especial em chamado atual? | +3 dias úteis (PROC-042 v2, seção 3) | Médio — v1 diz +2 dias |
| Tempo de resposta para incidente crítico Gold? | Até 30 minutos (SLA-2024, tabela seção 2) | Médio — FAQ item 41 omite incidentes críticos |
| SLA Gold inclui gerente de conta dedicado? | Sim (SLA-2024, seção 2) | Médio — FAQ item 41 não menciona |
| Existe tier Platinum na NovaTech? | Não — descontinuado em 2022 (SLA-2024, nota seção 1; FAQ item 15) | Baixo — alinhado entre fontes |
| Prazo base para devolução padrão? | 7 dias úteis após recebimento confirmado (POL-001, seção 3.1) | Baixo |
| Carga perigosa pode ser devolvida pelo processo padrão? | Não — deve ser encaminhada ao ramal 4500 (POL-001, seção 3.2) | Médio — FAQ item 3 acrescenta nuances não documentadas |
| Pergunta sobre frete abaixo de 500kg | Assistente informa que não há documento na base cobrindo esse tema e recomenda contato com o Comercial | Alto — ausência de normativo pode levar a resposta inventada |
| Pergunta sobre política para carga danificada em trânsito | Assistente informa que a informação existe apenas no FAQ (item 38) e sinaliza que não foi validada por Compliance | Médio — FAQ cobre, mas sem normativo |

Regressão detectada em qualquer caso bloqueia a publicação.

> ⚠️ **Pendência P10-A — Responsável pelo pipeline e processo de aprovação indefinidos**
> Esta spec pressupõe, em múltiplos pontos, a existência de uma área ou pessoa com autoridade para aprovar publicações na base, decidir classificações de status em casos ambíguos e receber notificações de bloqueio. Nenhum dos documentos disponíveis define essa estrutura de governança. A definição requer uma decisão organizacional explícita — qual área é dona do pipeline, quem aprova publicações e qual o processo em caso de divergência entre áreas (ex: Operações quer publicar um documento que o Compliance ainda não validou). **Enquanto essa decisão não for tomada, os fluxos de aprovação, bloqueio e revisão humana descritos nesta spec não têm destinatário definido e não podem ser implementados.** Este é o gap com maior efeito de desbloqueio: sua resolução é pré-requisito para definir o limiar de confiança (P8) e acionar o Jurídico sobre retenção de dados (P9). Ver P10 na tabela de Perguntas em Aberto.

---

## 5. Requisitos de rastreabilidade — citação de fonte e trecho relevante

Toda resposta entregue pelo assistente deve ser rastreável até sua origem documental. Isso protege o atendente juridicamente, permite auditoria pelo Compliance e viabiliza identificar a origem de erros quando ocorrem.

### O que toda resposta deve incluir obrigatoriamente

- **Identificador do documento:** ex. PROC-042-v2, POL-001, SLA-2024
- **Versão do documento:** ex. v2, v3.1, 2024.1
- **Data de vigência:** a data a partir da qual o documento está em vigor
- **Seção de origem:** ex. "PROC-042 v2, seção 2.1" ou "POL-001, seção 3.2" — não apenas o documento, mas o trecho específico
- **Tipo de fonte:** normativo oficial (POL, PROC, SLA) ou FAQ operacional
- **Tier do cliente aplicado à resposta:** quando relevante para SLA ou condições contratuais

**Regra de comportamento quando o tier do cliente não tem cobertura para o item consultado:**
Quando o SLA-2024 define valores distintos por tier e o atendente consulta sobre um item que existe apenas para tiers superiores — como gerente de conta dedicado (exclusivo Gold) ou relatório mensal detalhado (Gold) vs. resumido (Silver) vs. sob demanda (Standard) — o assistente deve citar o que o SLA define especificamente para o tier do cliente informado, sem mencionar o que outros tiers recebem, a menos que o atendente pergunte explicitamente pela comparação. Se o tier do cliente não tem cobertura para aquele item (ex: cliente Silver perguntando sobre gerente dedicado), o assistente responde: *"Conforme SLA-2024, seção 2, esse benefício não se aplica ao tier [X]"* — e cita o que o tier do cliente efetivamente tem. O assistente nunca deve omitir a informação nem deixar o atendente sem resposta por ausência de cobertura parcial.

### Alertas obrigatórios que compõem a rastreabilidade

| Situação | Mensagem ao atendente |
|---|---|
| Fonte é o FAQ operacional | "Informação proveniente do FAQ operacional — não validada por Compliance." |
| Documento está em revisão pelo Compliance | "Este documento está em revisão e pode sofrer alterações." — aplicável especialmente à PROC-043 |
| Resposta cruzou múltiplos documentos | Cada documento deve ser citado individualmente com versão, data e seção |
| Assistente aplicou regra de transição da PROC-042 | Citar explicitamente: "Aplicando PROC-042 v[X] conforme data de abertura do chamado ([data])" |
| Baixa confiança na recuperação | "Não localizei resposta com confiança suficiente. Recomendo escalar para o supervisor." |

### Rastreabilidade para respostas que cruzam múltiplos documentos

Algumas perguntas exigem combinação de informações de documentos que não se referenciam entre si na base original. A resposta deve citar cada fonte separadamente.

*Exemplo concreto:* "Quanto tempo um cliente Gold tem para devolver uma carga de frete especial com destino ao Norte?"
A resposta exige três documentos:
- POL-001 seção 3.1: prazo base de 7 dias úteis
- PROC-042 v2 seção 3: +3 dias úteis para frete especial (aplicável para chamados a partir de 01/12/2023)
- SLA-2024 seção 2: confirmar que não há SLA diferenciado de devolução para tier Gold (a tabela cobre apenas tempo de resposta e resolução de chamados, não prazos de devolução)

Cada fonte deve aparecer citada na resposta com sua versão e seção.

### Rastreabilidade para auditoria interna

Além do que é exibido ao atendente, o sistema deve registrar internamente para cada consulta:

- Lista dos fragmentos recuperados com identificador, posição no documento original e pontuação de relevância
- Versão da base vetorial utilizada no momento da consulta
- Data e hora da consulta
- Se houve aplicação de regra de transição (ex: PROC-042), qual regra foi aplicada e com base em qual dado de contexto

Esse registro é necessário para auditar chamados históricos — especialmente os casos da PROC-042, onde a versão correta depende da data de abertura do chamado, que pode ser contestada pelo cliente.

### Publicação com registro auditável

Cada publicação da base deve gerar um registro permanente contendo:

- Data e hora da publicação
- Lista de documentos alterados e seus novos status
- Responsável pela aprovação
- Referência à versão anterior (arquivada, não deletada, acessível para auditoria)

A versão anterior nunca deve ser deletada — apenas arquivada. Isso garante que chamados históricos possam ser auditados com a base exatamente como estava no momento do atendimento.

> ⚠️ **Pendência P9-A — Prazo de retenção dos registros indefinido**
> Esta spec não define por quanto tempo os registros de auditoria e as versões arquivadas da base devem ser mantidos. Esse prazo tem implicação legal: envolve o prazo prescricional dos contratos de transporte e eventuais obrigações regulatórias sob a LGPD e resoluções da ANTT. A definição requer input do Jurídico da NovaTech e não pode ser determinada a partir da documentação operacional disponível. **Enquanto essa informação não estiver disponível, nenhuma política de exclusão ou expiração de registros deve ser implementada.** Ver P9 na tabela de Perguntas em Aberto.

---

## Perguntas em aberto — validação necessária antes da implementação

| # | Pergunta | Impacto direto na spec |
|---|---|---|
| P1 | Quem é o dono formal do FAQ-atendimento? Existe processo de revisão periódica? | O FAQ tem pelo menos duas informações erradas (limiar de desconto, prazo adicional de frete). Sem governança, qualquer correção pontual se deteriora. |
| P2 | O redirecionamento para o ramal 4500 é processo formal ou workaround? | POL-001 menciona o ramal, mas não há procedimento documentado. Define se o item 3 do FAQ deve virar normativo. |
| P3 | O processo do item 38 (carga danificada, 48h, `sinistros@novatech.com.br`) é reconhecido formalmente pelo Jurídico? | É o item mais útil do FAQ e o principal candidato a virar POL ou PROC. Sem confirmação, permanece como FAQ com alerta. |
| P4 | Qual o prazo para conclusão da revisão da PROC-043 pelo Compliance? Existe versão provisória em uso? | Enquanto não publicada, todo chamado envolvendo carga perigosa com frete especial acima de 500kg recebe alerta de instabilidade documental. |
| P5 | Existem contratos de clientes que referenciam explicitamente a PROC-042 v1? | A regra de transição por data de chamado pode não ser suficiente para esses clientes — eles podem ter direito contratual à v1 independente da data. |
| P6 | Quem atualiza a planilha `frete-base-AAAAMM.xlsx`? A atualização é realmente mensal ou ad hoc? | Determina a frequência mínima do monitoramento por hash e o risco real de janela de desatualização silenciosa. |
| P7 | Qual o percentual das ~400 páginas do Confluence ativamente mantido? Há páginas sem revisão há mais de 6 meses? | Páginas obsoletas indexadas com peso total são tão problemáticas quanto documentos revogados não removidos. |
| P8 | O fluxo de feedback descrito no novatech-jornada-rag-v4.md (seção 2.4) reflete como o feedback será capturado na interface do assistente? | Validação com o Product Owner. Enquanto não confirmado, o comportamento de registro de feedback permanece indefinido para implementação. Desbloqueado por P10. |
| P9 | Qual o prazo legal de retenção para registros de auditoria do pipeline e versões arquivadas da base? | Requer input do Jurídico: prazo prescricional dos contratos de transporte + obrigações regulatórias sob LGPD e ANTT. Enquanto não definido, nenhuma política de exclusão de registros deve ser implementada. Desbloqueado por P10. |
| P10 | Qual área é dona do pipeline de RAG? Quem tem autoridade para aprovar publicações na base, decidir classificações ambíguas e receber notificações de bloqueio? Qual o processo em caso de divergência entre áreas? | Decisão organizacional. Pré-requisito para P8 e P9. Sem isso, os fluxos de aprovação, bloqueio e revisão humana não têm destinatário e não podem ser implementados. |

---

*Documento produzido com base em novatech-jornada-rag-v4.md e Anexo A — Documentação Simulada NovaTech (05/06/2026). Contradições e gaps foram verificados diretamente nos textos de POL-001, PROC-042 v1 e v2, SLA-2024 e FAQ-atendimento. Validação com stakeholders necessária antes da implementação.*
