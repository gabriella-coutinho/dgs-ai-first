# Análise completa da documentação NovaTech — Product specialist review

**Data:** 05/06/2026  
**Escopo:** Análise de 5 documentos internos da NovaTech para avaliação de assistente de IA  
**Documentos analisados:** POL-001, PROC-042 v1, PROC-042 v2, SLA-2024, FAQ-atendimento  
**Metodologia:** Três etapas progressivas — mapeamento por metadados → análise de contradições entre PROC-042 v1 e v2 → cruzamento integral com FAQ-atendimento

---

## Parte 1 — Mapeamento inicial da base documental

### 1.1 Temas cobertos por documento

| Documento | Temas principais | Status |
|---|---|---|
| POL-001 v3.1 | Devoluções, prazos (7 dias úteis), exceções de carga | Normativo oficial |
| PROC-042 v1 | Cálculo de frete especial >500kg, multiplicadores regionais, acréscimo de prazo (+2d) | ⚠️ Sem revogação formal |
| PROC-042 v2 | Multiplicadores revisados, fatores de peso ajustados, acréscimo de prazo (+3d) | ⚠️ Sem substituição formal da v1 |
| SLA-2024 | Tiers Gold/Silver/Standard, tempos de resposta/resolução, penalidades | Contratual |
| FAQ-atendimento | Prática do suporte, casos de borda, conhecimento operacional informal | ⚠️ Não validado |

### 1.2 Gaps identificados no mapeamento inicial

**Processo de acionamento de SLA:** Nenhum documento descreve como um cliente aciona formalmente os níveis de serviço ou escala um chamado por tier.

**Vínculo devolução ↔ frete especial:** POL-001 e PROC-042 não se referenciam. Não está claro como o prazo de frete especial (+2 ou +3 dias) interage com o prazo de devolução de 7 dias úteis.

**Rastreabilidade de versões:** Não há política de gestão de versões. A vigência da v2 está definida (01/12/2023), mas não há revogação formal da v1.

**Fluxo de exceções:** POL-001 lista exceções (carga perigosa, ruptura de frio, lacre violado) mas não define quem decide nem como se aciona o fluxo alternativo.

**Tier e cálculo de frete:** SLA-2024 diferencia clientes por tier, mas PROC-042 não menciona se Gold tem condições diferenciadas de frete ou prazo.

**Validação do FAQ:** FAQ-atendimento não possui data de revisão, responsável ou processo de atualização, criando risco sistêmico de desinformação.

### 1.3 Hipóteses de contradição identificadas no mapeamento

**PROC-042 v1 vs v2 — coexistência ambígua (risco crítico):** A v2 define vigência a partir de 01/12/2023 para novos chamados, mas a v1 não foi revogada. Chamados abertos antes dessa data, ou equipes sem acesso à atualização, podem usar multiplicadores e fatores de peso divergentes — gerando valores de frete diferentes para cargas idênticas.

**Prazo base de devolução vs acréscimo de frete especial:** POL-001 define 7 dias úteis para devolução; PROC-042 v1 acrescenta +2 dias e v2 acrescenta +3 dias ao prazo de entrega. Não está definido se esse acréscimo é somado ao prazo de devolução. O prazo total comunicado ao cliente pode variar entre 7, 9 ou 10 dias úteis dependendo do documento consultado.

**FAQ vs normativos — contradição silenciosa:** O FAQ foi produzido de forma informal e potencialmente antes da v2 do PROC-042 (vigência dez/2023). Respostas sobre cálculo de frete ou prazos no FAQ podem refletir regras da v1 já desatualizadas.

**SLA-2024 vs ausência de hierarquia de tier em outros documentos:** O SLA define obrigações distintas por tier, mas nenhum outro documento reconhece essa hierarquia. Uma política de devolução ou cálculo de frete uniforme pode conflitar com compromissos diferenciados assumidos contratualmente com clientes Gold.

---

## Parte 2 — Análise comparativa PROC-042 v1 vs v2

### 2.1 Inconsistências campo a campo

| Campo | V1 (mar/2023) | V2 (nov/2023) | Variação | Risco |
|---|---|---|---|---|
| Fator de peso — 1.001 a 3.000kg | 1,2 | 1,15 | ▼ −4,2% | Médio |
| Fator de peso — acima de 3.000kg | 1,5 | 1,4 | ▼ −6,7% | Alto |
| Multiplicador regional — Sul | 1,2 | 1,3 | ▲ +8,3% | Médio |
| Multiplicador regional — Sudeste | 1,0 | 1,1 | ▲ +10,0% | Médio |
| Multiplicador regional — Centro-Oeste | 1,3 | 1,4 | ▲ +7,7% | Médio |
| Multiplicador regional — Nordeste | 1,4 | 1,5 | ▲ +7,1% | Médio |
| Multiplicador regional — Norte | 1,6 | 1,8 | ▲ +12,5% | Alto |
| Prazo adicional de entrega | +2 dias úteis | +3 dias úteis | ▲ +1 dia | Médio |
| Threshold desconto de volume | ≥10 fretes/mês → negociação manual | ≥8 fretes → 5%; ≥15 → 10%; acima → Diretoria | Mudança estrutural | Alto |
| Status da PROC-043 | Referenciada como vigente | Em revisão pelo Compliance | Divergente | Médio |
| Disposições transitórias | Não existe | Presente (seção 5) — regula chamados em aberto | — | Informativo |

### 2.2 Vigência e datas

**V1:** Emitida em 03/03/2023. Sem cláusula de vigência ou expiração. Não há indicação de que foi substituída — coexiste no SharePoint sem hierarquia definida.

**V2:** Emitida em 10/11/2023. Seção 5 define vigência para chamados novos a partir de 01/12/2023. Chamados abertos antes disso mantêm as regras da v1 — mas a v1 não foi revogada formalmente.

**Zona de risco — chamados no limbo:** Existe uma janela entre nov/2023 (emissão da v2) e dez/2023 (entrada em vigor). Chamados abertos nesse intervalo não têm regra clara. A v2 já existia no sistema, mas ainda não era obrigatória — um atendente desavisado poderia aplicá-la antecipadamente.

### 2.3 Impacto prático ao usar a versão errada

**Frete cobrado incorretamente:** Para carga acima de 3.000kg com destino ao Norte, o frete calculado pela v1 resulta em fator 1,5 × multiplicador 1,6 = 2,4. Pela v2: 1,4 × 1,8 = 2,52. Diferença de aproximadamente 5% no valor final. Usar v2 onde deveria ser v1 faz o cliente pagar mais do que o contratado.

**Prazo prometido incorreto:** Diferença de 1 dia útil entre as versões. Usar v2 para um chamado que deveria usar v1 promete entrega com +3 dias quando o contrato prevê +2 — gerando expectativa errada e possível descumprimento de SLA.

**Desconto de volume aplicado indevidamente:** V1 exige negociação manual a partir de 10 fretes/mês. V2 aplica 5% automático a partir de 8. Um atendente usando v2 pode conceder desconto não previsto no contrato do cliente — impacto financeiro direto para a NovaTech.

**Referência a PROC-043 desatualizada:** Usar v1 para cargas perigosas direciona para uma PROC-043 que, segundo a v2, está em revisão pelo Compliance. O atendente pode aplicar regras de uma norma que está prestes a mudar.

### 2.4 Perguntas de discovery — PROC-042

1. A v2 substitui integralmente a v1 ou elas coexistem intencionalmente para tipos distintos de chamado? *(Define se precisamos de uma tabela de decisão ou se a v1 pode ser arquivada.)*
2. Existe um processo formal de revogação de versões? Quem é o responsável por marcar um documento como obsoleto no SharePoint? *(Identifica se o problema é pontual ou sistêmico na gestão documental.)*
3. Como o atendente sabe qual versão aplicar hoje? Existe algum treinamento ou comunicado oficial sobre a transição? *(Avalia o risco real de erro operacional no dia a dia.)*
4. Os descontos de volume da v2 já foram comunicados aos clientes? Há contratos que ainda referenciam as regras da v1? *(Determina o risco contratual de aplicar v2 retroativamente.)*
5. Qual é o status atual da revisão da PROC-043 pelo Compliance? Há prazo previsto? *(A PROC-043 é referenciada por ambas as versões — se estiver desatualizada, o problema se propaga para cargas perigosas.)*
6. O FAQ-atendimento foi atualizado após a entrada em vigor da v2? Quem é responsável por mantê-lo alinhado com os normativos? *(Fecha o ciclo: mesmo que os PROCs sejam corrigidos, o FAQ continuará sendo fonte de desinformação se não houver um dono definido.)*

---

## Parte 3 — Cruzamento integral: FAQ-atendimento vs documentos oficiais

### 3.1 Resumo quantitativo

| Categoria | Qtd |
|---|---|
| Contradições diretas com normativos | 3 |
| Informações desatualizadas ou sem fonte | 2 |
| Gaps preenchidos pelo FAQ | 4 |
| Itens que requerem validação com stakeholders | 6 |

### 3.2 Contradições diretas

**Item 45 — Regra de desconto por volume**
- FAQ diz: desconto disponível a partir de 10 fretes especiais/mês; processos acima disso redirecionados ao Comercial.
- Normativo oficial (PROC-042 v2): threshold mudou para 8 fretes (5% automático) e 15 fretes (10%). O FAQ ainda reflete a v1, substituída em dez/2023.
- Impacto: atendente que segue o FAQ nega desconto a clientes que já têm direito pela v2, ou redireciona ao Comercial um processo que deveria ser automático.

**Item 8 — Qual versão da PROC-042 usar**
- FAQ diz: "Na dúvida, use a mais recente (v2), mas se o cliente reclamar, pode ser que o contrato esteja na tabela antiga."
- Normativo oficial (PROC-042 v2, seção 5): a regra é objetiva — data de abertura do chamado determina a versão. Antes de 01/12/2023 → v1; a partir de 01/12/2023 → v2.
- Impacto: o FAQ transforma uma regra objetiva em decisão subjetiva do atendente, aumentando inconsistência e risco de litígio.

**Item 41 — Tempos de SLA por tier**
- FAQ diz: Gold: 2h resposta / 24h resolução. Silver: 4h / 48h. Standard: 8h / 72h.
- Normativo oficial (SLA-2024): o FAQ omite incidentes críticos, penalidades por descumprimento, disponibilidade do portal e gerente de conta dedicado (Gold). Os valores citados podem corresponder apenas aos chamados gerais, ignorando obrigações adicionais.
- Impacto: atendente transmite visão incompleta do SLA contratado; clientes Gold podem desconhecer direitos como gerente dedicado e relatório mensal detalhado.

### 3.3 Informações desatualizadas ou sem fonte verificável

**Item 32 — Carga perigosa com frete expresso:** O FAQ orienta o fluxo com base na PROC-043 como vigente. A PROC-042 v2 informa que a PROC-043 está em revisão pelo Compliance. O atendente pode estar citando um processo prestes a mudar — ou que já mudou sem comunicação formal.

**Item 22 — Seguro de carga (0,3% e 0,8%):** Os percentuais citados não têm correspondente em nenhum documento oficial. Podem estar corretos, mas sem normativo de referência são impossíveis de validar e podem divergir por contrato ou data de assinatura.

### 3.4 Gaps que o FAQ preenche (conhecimento sem normativo correspondente)

**Item 3 — Fluxo prático para devolução de carga perigosa:** Nenhum documento oficial define o que o atendente faz na prática. O FAQ preenche com o ramal 4500 (Gestão de Riscos) e a possibilidade de exceção. Conhecimento operacional real que deveria ser formalizado em normativo.

**Item 27 — Parâmetro de alerta para rastreamento por rota:** Nenhum documento define o que é "normal" para tempo em trânsito por região. O FAQ cria um parâmetro informal (Norte: até 10 dias úteis; Sul/Sudeste: alerta após 3 dias) que os normativos não cobrem.

**Item 38 — Processo para carga danificada em trânsito:** Fluxo distinto de devolução sem nenhum normativo correspondente. O FAQ descreve: prazo de 48h para registro, necessidade de fotos e laudo, encaminhamento ao Jurídico via `sinistros@novatech.com.br`. Informação crítica sem documento formal.

**Item 15 — Tier Platinum inexistente:** O FAQ registra comportamento real e recorrente de clientes confundindo tiers, incluindo contexto histórico (programa descontinuado em 2022). A base oficial ignora esse caso de borda completamente.

### 3.5 Veredicto por item: atendente seguindo o FAQ responderia certo?

| Item | Resposta do atendente via FAQ | O que deveria dizer | Veredicto |
|---|---|---|---|
| 45 | Desconto a partir de 10 fretes, via Comercial | A partir de 8 fretes: 5% automático; 15 fretes: 10% | ❌ Errado |
| 8 | "Na dúvida, use a v2" | Verificar a data de abertura do chamado para decidir a versão | ❌ Errado |
| 41 | Cita SLAs sem mencionar penalidades nem gerente dedicado | Confirmar valores no SLA-2024 e mencionar obrigações completas por tier | ⚠️ Incompleto |
| 32 | Orienta com base na PROC-043 como vigente | Avisar que a PROC-043 está em revisão; confirmar com Compliance antes de comprometer | ⚠️ Incompleto |
| 3 | Redireciona ao ramal 4500; não diz que é impossível | Alinhado com o espírito da POL-001. Resposta razoável. | ✅ Aceitável |
| 38 | 48h, fotos, laudo, sinistros@novatech.com.br | Sem normativo conflitante. Informação útil sem contradição identificada. | ✅ OK (sem conflito) |

### 3.6 Perguntas de discovery prioritárias — FAQ e governança

**P1 — Governança do FAQ (prioridade máxima)**  
Quem é o dono do FAQ-atendimento? Existe processo de revisão periódica ou ele cresce organicamente sem controle?  
*Antes de qualquer correção pontual, é preciso saber se o problema é de conteúdo ou de governança. Se não houver dono, qualquer atualização feita agora vai se deteriorar novamente.*

**P2 — Validação dos números de SLA**  
Os valores do item 41 foram confirmados com o responsável pelo SLA-2024?  
*O FAQ cita valores específicos sem fonte. Se estiverem errados, o atendente está comunicando compromissos contratuais incorretos ao cliente.*

**P3 — Formalização do fluxo de sinistros**  
O processo do item 38 — prazo de 48h, `sinistros@novatech.com.br` — está documentado formalmente? O Jurídico reconhece esse fluxo?  
*É o item mais útil do FAQ e não tem normativo correspondente. Se validado, deve virar política oficial.*

**P4 — Ramal 4500: processo formal ou workaround?**  
O redirecionamento ao ramal 4500 para carga perigosa é processo formal ou workaround criado pelo time?  
*Se for workaround, a POL-001 precisa ser atualizada para cobrir o fluxo de exceção que está acontecendo na prática.*

**P5 — Origem dos percentuais de seguro**  
Os valores 0,3% e 0,8% para seguro de carga têm algum normativo de origem? Valem para todos os contratos?  
*É informação financeira sem fonte rastreável. Um cliente com percentual diferente pode questionar e o atendente não terá documento para embasar.*

**P6 — Status da revisão da PROC-043**  
Qual é o prazo para conclusão da revisão pelo Compliance? Existe uma versão provisória em uso?  
*Dois documentos já apontam para a PROC-043 em estados diferentes (vigente vs em revisão). O time de atendimento está operando sem saber qual vale.*

---

## Diagnóstico consolidado

### Problema central: ausência de governança documental

A análise das três etapas revela que o problema não é a existência de documentos errados — é a ausência de um sistema que mantenha os documentos corretos ao longo do tempo. Três manifestações concretas disso:

**1. Coexistência sem hierarquia (PROC-042 v1 e v2):** A v2 define uma regra de transição clara na seção 5, mas sem revogação formal da v1, qualquer agente que não conheça a v2 opera com regras desatualizadas. O sistema documental atual não tem mecanismo para sinalizar isso.

**2. Dependência oculta do FAQ:** O FAQ-atendimento está preenchendo lacunas que os normativos deveriam cobrir (sinistros, rastreamento por rota, fluxo para carga perigosa), mas o faz de forma invisível à organização. Se o FAQ for descontinuado, parte do conhecimento operacional da empresa some sem registro. Se continuar sendo mantido sem dono, continuará se defasando silenciosamente.

**3. Ilhas documentais:** POL-001, PROC-042 e SLA-2024 funcionam como documentos independentes sem referências cruzadas. Um atendente que consulta apenas um deles tem uma visão parcial — e não tem como saber que está com uma visão parcial.

### Mapa de risco por documento

| Documento | Confiabilidade atual | Principal risco |
|---|---|---|
| POL-001 v3.1 | Alta — normativo recente e controlado | Não cobre exceções com detalhamento suficiente |
| PROC-042 v1 | Baixa — desatualizado desde dez/2023 | Ser consultado em vez da v2 |
| PROC-042 v2 | Média — correto, mas sem revogação formal da v1 | Não ser encontrado ou reconhecido como substituto |
| SLA-2024 | Alta — documento contratual recente | Não ser integrado ao fluxo de atendimento |
| FAQ-atendimento | Variável — itens úteis coexistem com erros | Ser tratado como fonte de verdade |

### Recomendação imediata para o discovery

Antes de corrigir itens individuais, duas ações estruturais devem ser priorizadas:

1. **Definir um dono formal para o FAQ** e estabelecer um ciclo de revisão vinculado às atualizações de POL e PROC. Qualquer correção de conteúdo feita sem resolver a governança vai se deteriorar novamente.

2. **Arquivar formalmente a PROC-042 v1** com nota de substituição apontando para a v2, ou publicar uma tabela de decisão unificada que elimine a ambiguidade sobre qual versão usar em cada contexto.

---

*Documento gerado como consolidado de análise para fins de discovery. Todas as informações derivam exclusivamente dos documentos internos fornecidos. Validação com stakeholders necessária antes de qualquer ação corretiva.*
