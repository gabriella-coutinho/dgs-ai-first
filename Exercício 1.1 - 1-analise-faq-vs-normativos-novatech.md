# Análise cruzada: FAQ-atendimento vs documentos oficiais NovaTech

**Data da análise:** 05/06/2026  
**Documentos de referência:** FAQ-atendimento (não controlado), PROC-042 v1, PROC-042 v2, SLA-2024, POL-001  
**Status:** Rascunho para discovery com stakeholders

---

## Resumo executivo

| Categoria | Qtd |
|---|---|
| Contradições diretas | 3 |
| Informações desatualizadas | 2 |
| Gaps preenchidos pelo FAQ | 4 |
| Itens para validar com stakeholders | 6 |

O FAQ-atendimento cumpre dois papéis que nenhum documento oficial assume: manual de procedimento de atendimento e registro de casos de borda. Isso cria uma dependência oculta — parte do conhecimento operacional da empresa existe apenas nesse documento informal, sem dono, sem versionamento e sem processo de revisão.

---

## 1. Contradições com documentos oficiais

### Item 45 — Regra de desconto por volume
**Risco:** Financeiro direto

| | Conteúdo |
|---|---|
| **FAQ diz** | Desconto disponível a partir de 10 fretes especiais/mês; casos acima disso redirecionados ao Comercial. |
| **Normativo oficial (PROC-042 v2)** | Threshold mudou para 8 fretes (5% automático) e 15 fretes (10%). O FAQ ainda reflete a v1, substituída em dez/2023. |

**Impacto:** Atendente que segue o FAQ nega desconto a clientes que já têm direito a ele pela v2, ou redireciona ao Comercial um processo que deveria ser automático.

---

### Item 8 — Qual versão da PROC-042 usar
**Risco:** Cálculo de frete incorreto + orientação ambígua

| | Conteúdo |
|---|---|
| **FAQ diz** | "Na dúvida, use a mais recente (v2), mas se o cliente reclamar, pode ser que o contrato esteja na tabela antiga." |
| **Normativo oficial (PROC-042 v2, seção 5)** | A regra é baseada na data de abertura do chamado: antes de 01/12/2023 → v1; a partir de 01/12/2023 → v2. Não é julgamento do atendente. |

**Impacto:** O FAQ transforma uma regra objetiva em decisão subjetiva do atendente, aumentando inconsistência e risco de litígio com clientes.

---

### Item 41 — Tempos de SLA por tier
**Risco:** Comunicação incorreta de compromissos contratuais

| | Conteúdo |
|---|---|
| **FAQ diz** | Gold: 2h resposta / 24h resolução. Silver: 4h / 48h. Standard: 8h / 72h. |
| **Normativo oficial (SLA-2024)** | O FAQ cita apenas tempos de resposta e resolução para chamados gerais, omitindo: tempos para incidentes críticos, penalidades por descumprimento, disponibilidade do portal, gerente de conta dedicado (Gold) e relatórios de performance. |

**Impacto:** Atendente transmite ao cliente uma visão incompleta do SLA contratado. Clientes Gold, em especial, podem desconhecer direitos como gerente dedicado e relatório mensal detalhado.

---

## 2. Informações desatualizadas

### Item 32 — Carga perigosa com frete expresso
O FAQ orienta o fluxo com base na PROC-043 como vigente. A PROC-042 v2 informa que a PROC-043 está em revisão pelo Compliance. O atendente pode estar citando um processo que está prestes a mudar — ou que já mudou sem comunicação formal.

### Item 22 — Seguro de carga (0,3% e 0,8%)
Os percentuais citados não têm correspondente em nenhum documento oficial fornecido. Podem estar corretos, mas sem normativo de referência são impossíveis de validar e podem divergir por contrato ou data de assinatura.

---

## 3. Gaps que o FAQ preenche (e que os normativos não cobrem)

### Item 3 — Fluxo prático para devolução de carga perigosa
Nenhum documento oficial define o que o atendente faz na prática. O FAQ preenche isso com o ramal 4500 (Gestão de Riscos) e a possibilidade de exceção. Conhecimento operacional real que deveria ser formalizado em normativo.

### Item 27 — Parâmetro de alerta para rastreamento por rota
Nenhum documento define o que é "normal" para tempo em trânsito por região. O FAQ cria um parâmetro informal (Norte: até 10 dias úteis; Sul/Sudeste: alerta após 3 dias) que os normativos não cobrem.

### Item 38 — Processo para carga danificada em trânsito
Carga danificada é fluxo distinto de devolução, mas nenhum normativo trata disso. O FAQ descreve: prazo de 48h para registro, necessidade de fotos e laudo, encaminhamento ao Jurídico via `sinistros@novatech.com.br`. Informação crítica sem documento formal correspondente.

### Item 15 — Tier Platinum inexistente
O FAQ registra um comportamento real e recorrente de clientes confundindo tiers, incluindo contexto histórico (programa descontinuado em 2022). A base oficial ignora completamente esse caso de borda.

---

## 4. Veredicto por item: atendente seguindo o FAQ responderia certo?

| Item | Resposta do atendente via FAQ | O que deveria dizer | Veredicto |
|---|---|---|---|
| 45 | Desconto a partir de 10 fretes, via Comercial | A partir de 8 fretes: 5% automático; 15 fretes: 10% | ❌ Errado |
| 8 | "Na dúvida, use a v2" | Verificar a data de abertura do chamado para decidir a versão | ❌ Errado |
| 41 | Cita SLAs sem mencionar penalidades nem gerente dedicado | Confirmar valores no SLA-2024 e mencionar obrigações completas por tier | ⚠️ Incompleto |
| 32 | Orienta fluxo com base na PROC-043 como vigente | Avisar que a PROC-043 está em revisão; confirmar com Compliance antes de comprometer | ⚠️ Incompleto |
| 3 | Redireciona ao ramal 4500; não diz que é impossível | Alinhado com o espírito da POL-001. Resposta razoável. | ✅ Aceitável |
| 38 | 48h, fotos, laudo, sinistros@novatech.com.br | Sem normativo conflitante. Informação útil sem contradição identificada. | ✅ OK (sem conflito) |

---

## 5. Perguntas de discovery para stakeholders

**P1 — Governança do FAQ (prioridade máxima)**  
Quem é o dono do FAQ-atendimento? Existe algum processo de revisão periódica ou ele cresce organicamente sem controle?  
*Antes de qualquer correção pontual, é preciso saber se o problema é de conteúdo ou de governança. Se não houver dono, qualquer atualização feita agora vai se deteriorar novamente.*

**P2 — Validação dos números de SLA**  
Os valores do item 41 (2h/24h Gold, 4h/48h Silver, 8h/72h Standard) foram confirmados com o responsável pelo SLA-2024?  
*O FAQ cita valores específicos sem fonte. Se estiverem errados, o atendente está comunicando compromissos contratuais incorretos para o cliente.*

**P3 — Formalização do fluxo de sinistros**  
O processo do item 38 — prazo de 48h, e-mail sinistros@novatech.com.br — está documentado formalmente em algum lugar? O Jurídico reconhece esse fluxo?  
*É o item mais útil do FAQ e não tem normativo correspondente. Se for validado, deve virar política oficial.*

**P4 — Ramal 4500: processo formal ou workaround?**  
O redirecionamento ao ramal 4500 (Gestão de Riscos) para carga perigosa é um processo formal ou um workaround criado pelo time?  
*Se for workaround, a POL-001 precisa ser atualizada para cobrir o fluxo de exceção que está acontecendo na prática.*

**P5 — Origem dos percentuais de seguro**  
Os valores 0,3% e 0,8% para seguro de carga têm algum normativo de origem? Valem para todos os contratos ou apenas para novas contratações?  
*É informação financeira sem fonte rastreável. Um cliente com percentual diferente pode questionar e o atendente não terá documento para embasar.*

**P6 — Status da revisão da PROC-043**  
Qual é o prazo para conclusão da revisão pelo Compliance? Existe uma versão provisória em uso?  
*Dois documentos já apontam para a PROC-043 em estados diferentes (vigente vs em revisão). O time de atendimento está operando sem saber qual vale.*

---

## Diagnóstico sistêmico

O problema central não é a existência de informações erradas no FAQ — é a ausência de governança documental. O FAQ está preenchendo lacunas que os normativos oficiais deveriam cobrir, mas o faz de forma informal, sem versionamento e sem dono. Isso cria três riscos simultâneos:

1. **Risco de contradição silenciosa:** quando normativos são atualizados (como ocorreu com a PROC-042 v2 em dez/2023), o FAQ não é sincronizado automaticamente — e o time continua operando com as regras antigas.
2. **Risco de perda de conhecimento:** os itens 3, 27 e 38 contêm conhecimento operacional real e valioso. Se o FAQ for descontinuado ou simplesmente parar de ser mantido, esse conhecimento some sem registro.
3. **Risco de falsa segurança:** o FAQ inclui um aviso de que "pode conter informações desatualizadas", mas na prática é o documento consultado no dia a dia — o aviso não muda o comportamento de uso.

**Recomendação imediata:** Antes de corrigir itens individuais, definir um dono formal para o FAQ e estabelecer um ciclo de revisão vinculado às atualizações de POL e PROC.
