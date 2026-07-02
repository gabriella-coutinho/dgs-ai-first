# Nota de Handoff ao Tech Lead — Product Rules & Guardrails

**De:** Product Specialist
**Para:** Tech Lead
**Referente a:** Seção "Product Rules & Guardrails" do `AGENTS.md` (entregue em anexo/PR separado)
**Objetivo deste documento:** Registrar pendências e perguntas de ordem técnica identificadas durante a escrita da seção, que exigem decisão ou tradução por parte do Tech Lead/time de Dev antes (ou depois) da seção ser incorporada ao repositório.

> Este documento **não faz parte do `AGENTS.md`**. A seção "Product Rules & Guardrails" já está pronta e pode ser incorporada como está — os pontos abaixo são refinamentos e decisões que dependem de conhecimento técnico que eu, como Product Specialist, não tenho.

---

## 1. Formato e estrutura (machine-readability)

**O que fiz:**
Escrevi as regras de comportamento (seção 1), o glossário (seção 2) e as referências a specs (seção 4) em tabelas Markdown. A seção 3 (restrições de código) ficou em lista numerada com texto corrido, por não saber qual formato seria mais fácil de consumir pelo pipeline ou por ferramentas de automação de vocês.

**Perguntas para o Tech Lead:**
- A seção 3 deveria seguir o mesmo padrão de tabela das demais, ou faz mais sentido convertê-la para um formato estruturado (YAML/JSON) quando for integrada ao repositório?
- Os IDs que usei nas regras (D1–D8, N1–N9, F1–F7) foram pensados para referenciar os guardrails da matriz original (G1–G6, N1–N6, F1–F6). Percebi que criei N7, N8 e N9 como IDs novos, mas eles na verdade remetem a guardrails já existentes (N1/N4). Isso pode causar ambiguidade se alguém tentar cruzar os IDs automaticamente. Faz sentido eu renomear essas três regras como subitens (ex.: N4.1, N4.2) para manter rastreabilidade 1:1 com a matriz, ou vocês preferem tratar isso de outra forma no versionamento?
- Seria útil adicionar um cabeçalho de metadados (versão da seção, data da última atualização) para rastrear qual revisão foi de fato incorporada ao `prompts/system-prompt.md`? Não sei se isso é convenção do projeto ou se o Git já resolve isso sozinho.

**O que preciso do Tech Lead:** uma decisão sobre formato de ingestão e uma validação/ajuste do esquema de IDs.

---

## 2. Prescritividade das regras (DEVE/NÃO DEVE)

**O que fiz:**
Segui a estrutura DEVE / NÃO DEVE / QUANDO EM DÚVIDA pedida, com linguagem direta ("deve", "não deve", "nunca"), referenciando o guardrail e o incidente de origem em cada linha.

**Perguntas para o Tech Lead:**
- Não indiquei, na tabela de regras, se cada uma tem enforcement garantido em código ou se depende do modelo seguir a instrução no prompt (essa distinção existe na matriz de guardrails original, coluna "Implementação"). Vale a pena eu repetir essa informação na seção do `AGENTS.md`, ou isso geraria redundância desnecessária já que a matriz pode ser consultada à parte?
- Não defini o que o assistente deve fazer quando duas regras entram em conflito no mesmo caso — por exemplo, "sempre citar a fonte" (D1) versus "não afirmar nada sob baixa confiança" (F3/F7). Essa é uma decisão de prioridade de negócio (o que decidimos que importa mais) ou uma decisão de implementação (como o pipeline resolve tecnicamente o conflito)? Preciso saber de quem é essa call para documentar corretamente.

**O que preciso do Tech Lead:** confirmar se a camada de implementação (Código/Prompt/Ambos) deve aparecer explicitamente na seção de produto, e apontar quem decide a ordem de precedência em regras conflitantes.

---

## 3. Cobertura do glossário

**O que fiz:**
Selecionei os termos classificados como risco crítico (Grupo 1) e os termos inexistentes no domínio (Grupo 4) do documento de linguagem ubíqua — os que considerei mais propensos a gerar erro de maior impacto.

**Pergunta para o Tech Lead:**
- Deixei de fora cinco termos que também têm risco real de confusão pelo modelo: **Chamado**, **Operação (contagem para tier)**, **Coleta reversa**, **Devolução parcial** e **Frete reverso**. Não incluí porque não sei se ainda cabem dentro do orçamento de ~4K tokens do system prompt definido no ADR-0002. Vocês conseguem validar se há espaço para incluir esses termos, ou preciso priorizar com alguém do time quais ficam de fora se o budget for o limitante?

**O que preciso do Tech Lead:** validação de espaço no context budget (ADR-0002) e apoio para priorizar os termos, caso seja necessário cortar.

---

## 4. Restrições de código

**O que fiz:**
Descrevi as restrições em linguagem de produto — o que um campo deve conter, quais valores um enum deve aceitar, o que deve acontecer antes de uma resposta ser enviada — citando os arquivos do Anexo C que pareciam mais relacionados a cada regra.

**Perguntas para o Tech Lead:**
- Não sei escrever essas restrições como schema/tipo real. Alguém do time consegue traduzir as descrições abaixo para TypeScript/Zod (ou o padrão que vocês já usam em `validator.ts`)?
  - `source_document: { document_id, version, section }` — obrigatório em toda resposta, mesmo com baixa confiança.
  - `source_type: "normativo" | "informal"` — obrigatório em todo chunk indexado.
  - `status: "vigente" | "obsoleto"` — metadado de vigência por documento versionado.
  - `baixa_confianca: true` — flag quando o score de relevância do retrieval fica abaixo do threshold.
  - Tier como enum fechado: `"Gold" | "Silver" | "Standard"` (sem aceitar "Platinum" ou qualquer outro valor).
- Não sei onde essas configurações deveriam morar no código (ex.: o valor do threshold de relevância do retrieval, o formato do log estruturado de eventos de baixa confiança). Citei `search.ts` e `response-validator.ts` como pontos prováveis, mas não tenho certeza se `src/shared/config.ts` é o lugar certo para centralizar isso, nem como nomear as variáveis de ambiente.
- Seria muito útil se alguém (Tech Lead ou Dev Sênior) anexasse um exemplo de JSON de resposta real com todos esses campos preenchidos juntos (ex.: uma resposta de baixa confiança citando um documento com contradição pendente). Não tenho como validar sozinho se o formato que imaginei é implementável como está, e um exemplo concreto ajudaria o time a alinhar expectativas antes de começar a codificar.

**O que preciso do Tech Lead:** tradução das descrições para schema formal, decisão sobre localização de configs, e um exemplo de payload de resposta de referência.

---

## Resumo das decisões pendentes

| # | Pendência | Quem decide | Bloqueia a incorporação da seção ao AGENTS.md? |
|---|-----------|-------------|--------------------------------------------------|
| 1 | Formato da seção 3 (tabela vs. lista) | Tech Lead | Não — pode ser ajustado depois |
| 2 | Correção do esquema de IDs (N7–N9 duplicando N1/N4) | Tech Lead | Recomendado antes do versionamento |
| 3 | Explicitar camada de implementação (Código/Prompt/Ambos) na seção de produto | Tech Lead | Não |
| 4 | Regra de precedência entre guardrails conflitantes | Tech Lead + Product Specialist | Sim, para casos de conflito (D1 vs. F3/F7) |
| 5 | Ampliar glossário (5 termos faltantes) dentro do budget do ADR-0002 | Tech Lead (valida budget) + Product Specialist (prioriza) | Não |
| 6 | Tradução das restrições de código para schema/tipos reais | Tech Lead / Dev Sênior | Sim, antes da implementação (não do texto do AGENTS.md) |
| 7 | Localização de configs (threshold, log) no código | Tech Lead | Sim, antes da implementação |
| 8 | Exemplo de payload de resposta de referência | Tech Lead / Dev Sênior | Recomendado, não bloqueante |

---

**Próximo passo sugerido:** revisar este documento junto com a seção "Product Rules & Guardrails" na próxima reunião de sincronização do time, e registrar as decisões tomadas em `docs/adr/` (se gerarem escolha arquitetural) ou em `prompts/prompt-changelog.md` (se resultarem em mudança direta no system prompt).
