# Validação de Respostas do Assistente de IA — NovaTech (Staging)

**Avaliador:** Product Specialist (validação pré-go-live)
**Referência:** Anexo A — Documentação Simulada NovaTech (POL-001, PROC-042 v1 e v2, SLA-2024, FAQ-Atendimento)

---

## Resumo das avaliações

| # | Pergunta (resumo) | Avaliação | Problema principal |
|---|---|---|---|
| 1 | Prazo de devolução — Standard | Parcialmente correta | Seção citada errada (3.2 em vez de 3.1/3.3) |
| 2 | Prazo de resolução — Silver | Parcialmente correta | Falta "úteis" e falta especificar "chamados gerais" vs. incidente crítico |
| 3 | Devolução de carga perigosa classe 3 | Parcialmente correta | Encaminhamento errado ("supervisor" em vez de Gestão de Riscos, ramal 4500) |
| 4 | Carga danificada em transporte | Incorreta | Alta confiança sem fonte; trata FAQ informal como política oficial |
| 5 | SLA do cliente "Enterprise" | Correta | — |
| 6 | Carga perigosa com frete expresso | Parcialmente correta | Confiança Alta incompatível com fonte informal; omite ressalva de prazo |

---

## Avaliação detalhada

### 1. "Qual o prazo de devolução para produtos standard?"

**Classificação: Parcialmente correta**

O conteúdo está correto: 7 dias úteis após o recebimento confirmado no tracking, abertura de chamado no portal, com fotos. Porém a **fonte citada está errada**. O assistente cita POL-001, seção 3.2 — mas a seção 3.2 trata das *exceções* ao prazo (cargas perigosas, refrigeradas, lacre violado), não do prazo em si. O prazo está na seção 3.1, e o procedimento com fotos está na 3.3. Uma citação de seção incorreta compromete a auditabilidade da resposta: se o atendente for conferir a fonte indicada, cai no trecho errado do documento.

### 2. "Meu cliente é Silver. Qual o prazo de resolução?"

**Classificação: Parcialmente correta**

O valor de 48h está correto para chamados gerais, conforme a tabela do SLA-2024. Mas a resposta omite duas qualificações relevantes:
- São **48h úteis**, não corridas — o relógio de SLA pausa fora do horário comercial (seção 5 do SLA-2024).
- A pergunta do cliente foi genérica, sem indicar se é chamado geral ou incidente crítico. Para incidente crítico, o prazo de resolução do Silver é de **8h**, não 48h. Sem essa qualificação, a resposta pode induzir a erro em um cenário de incidente crítico.

### 3. "Posso devolver carga perigosa classe 3?"

**Classificação: Parcialmente correta**

A regra central está certa: cargas classificadas nas classes 1 a 6 da ANTT — incluindo a classe 3 (líquidos inflamáveis) — não são elegíveis para devolução pelo processo padrão (POL-001, seção 3.2). Porém o encaminhamento sugerido está errado: a política não menciona "supervisor", e sim contato com **Gestão de Riscos, ramal 4500**, um canal específico para tratamento individual desses casos.

### 4. "Qual a política para carga danificada durante transporte?"

**Classificação: Incorreta**

Não existe nenhum documento normativo (POL ou PROC) sobre carga danificada em trânsito — isso é um gap explicitamente identificado na documentação. A única informação disponível está no FAQ-Atendimento, item 38, documento classificado como informal e **não validado por Compliance ou Operações**. O assistente:
- apresentou a informação como se fosse política oficial, sem qualquer ressalva;
- atribuiu **confiança Alta** citando **"Fonte: Nenhuma"** — combinação incompatível, típica de alucinação ou falha de recuperação no pipeline de RAG;
- omitiu que o processo real (segundo o FAQ) exige registro em até 48h e é tratado pelo **Jurídico** (sinistros@novatech.com.br), não pelo atendimento comum.

Esta é a resposta mais crítica da amostra e não deveria seguir para produção nesse estado.

### 5. "Qual o SLA do cliente Enterprise?"

**Classificação: Correta**

O tier "Enterprise" não existe na documentação (SLA-2024 confirma apenas Gold, Silver e Standard). O assistente não inventou um SLA, sinalizou **confiança Baixa** de forma coerente e recomendou confirmação/escalonamento. Vale notar que esta resposta foi construída por **inferência** (ausência do tier na lista de tiers documentados, combinada com a nota explícita de que não existem outros tiers) e não por recuperação direta de uma frase pronta — o que reforça a adequação da confiança Baixa.

### 6. "Posso enviar carga perigosa com frete expresso?"

**Classificação: Parcialmente correta**

A fonte citada (FAQ-Atendimento, item 32) é pertinente e corresponde diretamente à pergunta — o item 32 trata especificamente de envio de carga perigosa com frete expresso, respondendo que sim, mediante autorização do Compliance e documentação ANTT atualizada. O problema não é a existência ou relevância da fonte, mas sim:
- a fonte é um documento informal, explicitamente **não validado** por Compliance ou Operações — o que torna **confiança Alta inadequada**;
- a resposta **omite o caveat prático** do próprio FAQ: a autorização leva cerca de 2 dias, ou seja, o "expresso" na prática não é tão expresso — informação relevante para não gerar expectativa incorreta no cliente.

---

## Observação transversal

Os itens 4 e 6 indicam que a lógica de confiança do assistente não está diferenciando adequadamente documentos normativos (POL, PROC, SLA) de documentos informais (FAQ-Atendimento, explicitamente marcado como não validado). Recomenda-se, antes do go-live, que a lógica de scoring rebaixe automaticamente a confiança de respostas cuja única fonte seja o FAQ, e que a citação de fonte sempre inclua a seção específica (não apenas o identificador do documento), para reduzir erros como o do item 1.
