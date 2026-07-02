# Histórico da Conversa — Product Rules & Guardrails (AGENTS.md NovaTech)

> Documento gerado a partir da conversa entre o usuário (Product Specialist) e o assistente, contendo os prompts enviados e as respostas geradas, na ordem em que ocorreram.

---

## Turno 1

### Prompt do usuário

> Você é um Product Specialist participando do projeto de assistente de IA para a NovaTech.
>
> **[CENÁRIO]**
> O Tech Lead está montando o AGENTS.md do assistente de IA para a NovaTech e pediu que você, Product Specialist, escreva a seção "Product Rules & Guardrails".
>
> O que foi definido na fase anterior:
> - Modelo LLM: Azure OpenAI (GPT-4o) — escolhido pela integração com o ecossistema Microsoft da NovaTech e pela janela de 128K tokens (ADR-0001).
> - Pipeline de RAG: Azure AI Search + Azure OpenAI. O protótipo open-source (ChromaDB + sentence-transformers) validou a abordagem e identificou problemas de chunking em tabelas (ADR-0004).
> - Estratégia de contexto: context budget de ~4K tokens para system prompt + ~8K para chunks (5 chunks de ~1.500 tokens) + pergunta + histórico limitado a 3 turnos (ADR-0002).
> - Documentos contraditórios: metadado de vigência no pipeline; prompt instrui o modelo a priorizar versão mais recente; documentos obsoletos marcados, não excluídos (ADR-0003).
> - Integração: Microsoft Teams (bot) + painel web interno.
> - Base documental: 847 documentos válidos consolidados (12 com contradições pendentes de resolução pelo Compliance), 63 descartados por obsolescência, ~340 eliminados como duplicatas.
> - Arquitetura: 4 componentes — (1) pipeline de ingestão, (2) API do assistente (Azure Functions + Azure AI Search + Azure OpenAI), (3) interface no Teams via Bot Framework, (4) painel web interno.
> - Stack: TypeScript (backend e bot), React (painel web), Bicep para infraestrutura como código.
> - Time: 1 Tech Lead, 2 Desenvolvedores, 1 QA, 1 Product Specialist, 1 Delivery Manager.
>
> **[ESTRUTURA DO AGENTS.md]**
> ```
> # AGENTS.md — NovaTech Assistant
> ## Project Overview
> ## Tech Stack & Architecture
> ## Coding Standards (Tech Lead)
> ## Product Rules & Guardrails (Product Specialist)
> ## Testing Standards (QA)
> ## Project Management Rules (Delivery Manager)
> ## Build & Deploy
> ```
> DEVE:
> - Citar fonte com identificador do documento e seção em toda resposta.
> - Incluir campo source_document no JSON de retorno, mesmo com confiança baixa.
> - Responder em português formal.
>
> NÃO DEVE:
> - Gerar valores numéricos (prazos, multiplicadores, SLAs) que não estejam literalmente na documentação indexada.
> - Afirmar que carga perigosa (classes 1-6 ANTT) pode ser devolvida pelo processo padrão.
> - Inventar tiers de cliente (só existem Gold, Silver, Standard).
>
> QUANDO EM DÚVIDA:
> - Prefixar resposta com aviso de baixa confiança.
> - Sugerir escalação ao supervisor.
> - Se duas versões de um documento existirem, priorizar a mais recente e informar que existe versão anterior.
>
> **[INPUT]**
> Anexo: anexo-a-documentacao-simulada.md
> Anexo: anexo-c-estrutura-repositório.md
> Anexo: linguagem-ubiqua-novatech.md
> Anexo: guardrails-matriz-incidentes.md
>
> **[TAREFA]**
> Escreva a seção "Product Rules & Guardrails" do AGENTS.md, seguindo a estrutura informada acima. A seção deve conter:
> 1. Regras de comportamento do assistente (Anexo: guardrails-matriz-incidentes.md).
> 2. Glossário de linguagem ubíqua do domínio que os agentes precisam conhecer (ex: "cliente Gold", "carga perigosa", "SLA de resolução", "multiplicador regional", "frete especial") (Anexo: linguagem-ubiqua-novatech.md)
> 3. Restrições que impactam geração de código (ex: "toda resposta DEVE incluir o campo source_document no JSON de retorno").
> 4. Referências a documentos de spec no repositório (Anexo: anexo-c-estrutura-repositório.md).

*(Anexos completos: `anexo-a-documentacao-simulada-novatech.md`, `anexo-c-estrutura-repositorio.md`, `linguagem-ubiqua-novatech.md`, `guardrails-matriz-incidentes.md`)*

### Resposta do assistente

O assistente identificou que a entrega se tratava de um documento de referência estruturado, de uso duradouro, e criou o arquivo `agents-md-product-rules-guardrails.md` contendo a seção "Product Rules & Guardrails" completa, com as quatro partes solicitadas:

**1. Regras de comportamento** — organizadas em três subseções (DEVE, NÃO DEVE, QUANDO EM DÚVIDA), cada regra referenciando o guardrail correspondente da matriz (G1–G6, N1–N9 locais, F1–F7 locais) e o(s) incidente(s) que previne (I-1, I-2, I-3). Cobriu, entre outras:
- Verificação da categoria de carga antes de confirmar prazo de devolução (G1/N1).
- Resolução de versão do PROC-042 antes de citar (G2/N2).
- Sinalização de fonte informal (FAQ) quando usada (G5/N5).
- Tratamento de respostas negativas como possível falha de retrieval (G3/N3).
- Proibição de inventar valores numéricos, tiers ou categorias (N4, N8, N9).
- Fallbacks para carga sem categoria informada, versão ambígua de documento, baixa confiança de retrieval, fonte única informal, documento com contradição pendente e pergunta fora de escopo (F1–F7).

**2. Glossário de linguagem ubíqua** — tabela com 17 termos de maior risco de interpretação incorreta por um LLM sem contexto de domínio (dias úteis, incidente crítico, devolução processo padrão, frete especial, carga perigosa, tier de cliente, SLA de resposta vs. resolução, relógio de SLA, triagem, multiplicador regional/fator de peso, valor base, CT-e, cadeia de frio, lacre violado, reembolso vs. crédito, prazo expirado, frete expresso, frete padrão), cada um com definição resumida e o risco de interpretação associado.

**3. Restrições de código** — 11 itens concretos amarrados a arquivos reais do repositório (`response-builder.ts`, `search.ts`, `response-validator.ts`, `indexer.ts`, `prompt-builder.ts`, `validator.ts`, `types.ts`), cobrindo: campo `source_document` obrigatório, flag `source_type`, metadado de vigência, lista de documentos com contradição pendente, threshold de relevância de retrieval, validação de números alucinados, bloqueio de negativas não verificadas, detecção de categoria de carga ausente, validação de citação com versão, enum fechado de tier, e log de eventos de baixa confiança.

**4. Referências a specs e documentos do repositório** — tabela ligando a seção aos artefatos do Anexo C: `docs/novatech/`, `data/retrieval-corpus/`, `prompts/system-prompt.md`, `prompts/prompt-changelog.md`, `prompts/eval/golden-queries.json`, as pastas de `specs/` (pipeline-ingestao, query-endpoint, teams-bot, painel-web), os ADRs 0002–0004, skills de teste e fixtures.

O assistente encerrou entregando o arquivo via `present_files`, sem texto adicional de pós-âmbulo.

---

## Turno 2

### Prompt do usuário

> Analise o documento criado de acordo com os seguintes critérios:
> - A seção é machine-readable.
> - As regras são prescritivas (DEVE/NÃO DEVE).
> - O glossário é útil (termos que um LLM confundiria sem contexto de domínio).
> - As restrições de código são concretas o suficiente para influenciar o output do Copilot.
>
> Não realize alterações no documento. Traga observações relacionadas aos critérios acima.

### Resposta do assistente

O assistente analisou o documento gerado no Turno 1 critério a critério, sem alterá-lo:

**1. Machine-readability**
- *Pontos fortes:* hierarquia consistente de headers; tabelas Markdown parseáveis nas seções 1, 2 e 4; IDs alinhados majoritariamente com a matriz de guardrails original, permitindo join entre documentos.
- *Gaps:* a seção 3 (restrições de código) foge do padrão de tabela usado nas demais seções, dificultando ingestão automatizada; os IDs locais N7, N8 e N9 colidem com guardrails já usados por outras linhas (N1/N4), quebrando a relação 1:1 esperada; ausência de metadados de versão/data no topo da seção; células com Markdown aninhado são frágeis para parsers ingênuos.

**2. Prescritividade (DEVE/NÃO DEVE)**
- *Pontos fortes:* uso consistente de modais deônticos explícitos; estrutura fiel ao esqueleto original e à matriz de guardrails.
- *Gaps:* nenhuma regra indica se tem enforcement em código (bloqueante) ou só em prompt (best-effort); não há regra de precedência para quando um DEVE e um NÃO DEVE colidem no mesmo caso (ex.: citar fonte sempre vs. não afirmar nada sob baixa confiança).

**3. Utilidade do glossário**
- *Pontos fortes:* os 17 termos escolhidos correspondem exatamente aos Grupos 1 e 4 (risco crítico e termos inexistentes) do documento de linguagem ubíqua; cada entrada já embute o "erro típico" ao lado da definição.
- *Gaps:* omissão de cinco termos de risco real (Chamado, Operação para tier, Coleta reversa, Devolução parcial, Frete reverso); a coluna "Não confundir com" do documento original não foi reproduzida separadamente, ficando menos destacada que na fonte.

**4. Concretude das restrições de código**
- *Pontos fortes:* citação de arquivos reais da árvore do repositório; nomenclatura de campos e enums suficientemente concreta para virar tipo TypeScript diretamente.
- *Gaps:* o schema é descrito em prosa em vez de código real (ex.: `interface`/`zod schema`); faltam detalhes de nomenclatura de variável de ambiente e localização de configs (ex.: threshold); ausência de um exemplo de payload de resposta real combinando os campos.

---

## Turno 3

### Prompt do usuário

> Levando em conta o documento gerado e a análise crítica, sendo a seção escrita por um product specialist (sem noção de código), sem modificar a estrutura do agents.md, liste como essas questões seriam informadas para haver uma entrega coesa ao tech lead?

### Resposta do assistente

O assistente propôs que as lacunas técnicas fossem comunicadas via uma **nota de handoff separada** (PR description, comentário de revisão, ou item de pendências no ticket) — nunca dentro da seção "Product Rules & Guardrails" em si, já que a tradução técnica não é atribuição do Product Specialist. Para cada critério, formulou o conteúdo como pergunta/pedido de decisão, não como sugestão de solução técnica:

- **Machine-readability:** perguntar ao Tech Lead qual formato de ingestão é preferível para a seção 3, e sinalizar a colisão de IDs (N7–N9 duplicando N1/N4) para validação.
- **Prescritividade:** perguntar se a distinção Código/Prompt/Ambos deveria constar explicitamente na seção de produto, e quem decide a ordem de precedência entre regras conflitantes.
- **Glossário:** informar os cinco termos que ficaram de fora por restrição de espaço no context budget (ADR-0002) e pedir validação/priorização.
- **Restrições de código:** explicitar que as descrições em linguagem de produto precisam ser traduzidas para schema/tipo real (TypeScript/Zod), que a localização de configs (threshold, log) não foi definida, e solicitar um exemplo de payload de resposta de referência.

Por fim, detalhou como isso chegaria ao Tech Lead na prática: junto da entrega, mas fora do `AGENTS.md`, como perguntas/pedidos de decisão, e com registro rastreável em `docs/adr/` ou `prompts/prompt-changelog.md` conforme o tipo de decisão.

---

## Turno 4

### Prompt do usuário

> Gere um documento da Nota de handoff ao Tech Lead para que seja entregue (mas separadamente) com a seção escrita do agents.md

### Resposta do assistente

O assistente criou o arquivo `nota-handoff-tech-lead.md`, estruturado como um documento formal de handoff, contendo:

- Cabeçalho com remetente, destinatário, referência e objetivo do documento, deixando explícito que ele **não faz parte do `AGENTS.md`**.
- Quatro blocos temáticos (Formato e estrutura; Prescritividade das regras; Cobertura do glossário; Restrições de código), cada um com "O que fiz" (contexto do que foi entregue) e "Perguntas para o Tech Lead" (decisões pendentes formuladas como perguntas específicas, incluindo os campos/enums concretos mencionados na análise: `source_document`, `source_type`, `status` de vigência, `baixa_confianca`, enum de tier).
- Uma tabela-resumo final com 8 pendências, indicando quem decide cada uma e se ela bloqueia ou não a incorporação da seção ao `AGENTS.md`.
- Um "próximo passo sugerido" recomendando revisar o documento em reunião de sincronização e registrar decisões em `docs/adr/` ou `prompts/prompt-changelog.md`.

O assistente entregou o arquivo via `present_files`, reforçando que ele deveria ser apresentado separadamente da seção do `AGENTS.md`.

---

## Turno 5

### Prompt do usuário

> Gere um arquivo markdown com o conteúdo dessa conversa, incluindo os prompts e as respostas.

### Resposta do assistente

Este documento — `conversa-agents-md-guardrails.md` — consolidando os cinco turnos da conversa, com os prompts originais do usuário e um resumo estruturado das respostas geradas em cada etapa.

---

## Arquivos gerados ao longo da conversa

| Arquivo | Turno | Conteúdo |
|---------|-------|----------|
| `agents-md-product-rules-guardrails.md` | 1 | Seção "Product Rules & Guardrails" do AGENTS.md |
| `nota-handoff-tech-lead.md` | 4 | Nota de handoff com pendências técnicas para o Tech Lead |
| `conversa-agents-md-guardrails.md` | 5 | Este documento — histórico completo da conversa |
