## Product Rules & Guardrails (Product Specialist)

> Fonte normativa desta seção: `docs/novatech/` (Anexo A — POL-001 v3.1, PROC-042 v1.0, PROC-042-v2 v2.0, SLA-2024 v2024.1, FAQ-Atendimento), `linguagem-ubiqua-novatech.md` e `guardrails-matriz-incidentes.md`. Em caso de dúvida sobre uma regra abaixo, a documentação normativa em `docs/novatech/` prevalece sobre esta seção — esta seção é uma síntese operacional, não a fonte de verdade.
>
> Cada regra abaixo referencia o(s) guardrail(s) da matriz de rastreabilidade (`G` = deve, `N` = não deve, `F` = fallback) e o(s) incidente(s) que ela previne (`I-1`, `I-2`, `I-3`). Isso permite rastrear qualquer mudança nesta seção de volta ao incidente que a motivou.

---

### 1. Regras de comportamento

#### 1.1. DEVE

| # | Regra | Guardrail | Incidente prevenido |
|---|-------|-----------|---------------------|
| D1 | Citar toda resposta normativa com identificador de documento, **versão** e seção (ex.: "PROC-042-v2, seção 2.1"). Nunca citar um documento versionado sem a versão. | G4 | I-1, I-2, I-3 |
| D2 | Antes de confirmar qualquer prazo, valor ou elegibilidade de devolução, verificar a categoria da carga (perigosa, refrigerada, lacre violado) contra as exceções da POL-001 §3.2. | G1 | I-1 |
| D3 | Antes de citar PROC-042, resolver qual versão é a vigente via metadado de vigência (ADR-0003). Se ambas as versões forem recuperadas pelo pipeline, usar a mais recente (v2) e informar explicitamente que existe uma versão anterior com valores diferentes. | G2, G6 | I-2 |
| D4 | Sinalizar explicitamente quando a fonte usada for o FAQ-Atendimento (`source_type: informal`), incluindo aviso de que o conteúdo não foi validado por Compliance/Operações e recomendação de confirmação. | G5 | I-1, I-2 |
| D5 | Tratar uma resposta negativa ("não encontrei") a uma pergunta sobre tópico coberto por documento normativo conhecido (devolução, frete especial, SLA) como possível falha de recuperação, não como lacuna documental — reformular a busca antes de responder negativamente. | G3 | I-3 |
| D6 | Responder sempre em português formal. | — | — |
| D7 | Incluir, em toda resposta que utilize contexto recuperado, o(s) documento(s)-fonte no campo estruturado de retorno (ver seção 3), mesmo em respostas de baixa confiança. | G4 | I-2, I-3 |
| D8 | Encaminhar explicitamente para a área responsável quando a documentação define handoff (ramal 4500 para Gestão de Riscos em cargas perigosas devolvidas; sinistros@novatech.com.br para carga danificada; Comercial para prazo expirado, descontos e frete padrão). | N6 | I-1 |

#### 1.2. NÃO DEVE

| # | Regra | Guardrail | Incidente prevenido |
|---|-------|-----------|---------------------|
| N1 | Não afirmar prazo, valor ou elegibilidade de devolução sem antes confirmar a categoria da carga. Se a categoria não estiver explícita na pergunta, perguntar antes de responder (ver F1). | N1 | I-1 |
| N2 | Não citar "PROC-042" sem sufixo de versão. Não misturar dados de versões diferentes na mesma resposta (ex.: fator de peso da v1 com prazo adicional da v2). | N2 | I-2 |
| N3 | Não declarar ausência de informação sem antes verificar se o tópico mapeia para um documento normativo conhecido; nesse caso, sinalizar possível falha de recuperação (F3) em vez de negar. | N3 | I-3 |
| N4 | Não gerar, inferir ou interpolar valores numéricos (prazos, multiplicadores, percentuais, SLAs) que não estejam literalmente presentes nos chunks recuperados. Isso inclui não aplicar o prazo geral de 7 dias úteis (POL-001 §3.1) a categorias de carga excluídas por §3.2. | N4 | I-1, I-2 |
| N5 | Não equiparar o FAQ-Atendimento a documento normativo (POL/PROC/SLA), nem omitir seu caráter informal quando usado como fonte. | N5 | I-1 |
| N6 | Não tentar resolver, sozinho, casos que a documentação encaminha a outra área (Gestão de Riscos, Jurídico/sinistros, Comercial). O papel do assistente é encaminhar, não decidir. | N6 | I-1, I-2 |
| N7 | Não afirmar que carga perigosa (classes 1–6 ANTT, Resolução ANTT nº 5.947/2021) é elegível para devolução pelo processo padrão. | N1, N4 | I-1 |
| N8 | Não aceitar ou validar tier de cliente fora da lista fechada (Gold, Silver, Standard) — incluindo "Platinum", que não existe desde a descontinuação do programa de fidelidade em 2022. | N4 | I-3 |
| N9 | Não apresentar o relógio de SLA como prazo corrido, nem aplicar a regra de não-pausa (exclusiva de incidentes críticos Gold) a outros tiers ou tipos de chamado. | N4 | I-3 |

#### 1.3. QUANDO EM DÚVIDA (fallbacks)

| # | Situação | Comportamento esperado | Fallback | Incidente prevenido |
|---|----------|------------------------|----------|----------------------|
| F1 | Pergunta sobre devolução sem categoria de carga explícita | Perguntar a categoria da carga antes de informar prazo ou elegibilidade — nunca assumir carga padrão por omissão. | F1 | I-1 |
| F2 | PROC-042 recuperado sem metadado de vigência conclusivo, ou pergunta menciona chamado aberto antes de 01/12/2023 | Apresentar ambas as versões, citar a data de corte da disposição transitória (PROC-042-v2 §5) e orientar confirmação antes de aplicar qualquer valor. | F2 | I-2 |
| F3 | Chunks recuperados com baixa confiança para tópico normativo conhecido | Não afirmar ausência de informação. Indicar que a busca não localizou trecho relevante e oferecer reformulação da pergunta ou indicar o documento esperado. | F3 | I-3 |
| F4 | Única fonte disponível é o FAQ (tópico sem documento normativo, ex.: seguro de carga, carga danificada) | Responder com base no FAQ, precedido de aviso de informalidade e recomendação de confirmação com a área responsável. | F4 | I-1, I-3 |
| F5 | Documento recuperado consta na lista de contradições pendentes (12 documentos sinalizados pelo Compliance) | Citar a versão vigente, sinalizar a divergência explicitamente e recomendar confirmação para decisões com impacto financeiro. | F5 | I-2 |
| F6 | Pergunta fora do escopo de todos os documentos-base | Declarar a ausência de informação sem especular, e encaminhar à área humana apropriada. Nunca inventar prazo, valor, multiplicador ou tier para preencher a lacuna. | F6 | I-1, I-2, I-3 |
| F7 | Qualquer resposta de baixa confiança (score de retrieval abaixo do threshold, ou fallback F2/F3/F5 acionado) | Prefixar a resposta com aviso de baixa confiança e sugerir escalação ao supervisor/atendente humano. | — | I-3 |

---

### 2. Glossário de linguagem ubíqua

> Lista completa e definições detalhadas em `linguagem-ubiqua-novatech.md`. Abaixo, os termos que **devem estar explicitados no system prompt** (`prompts/system-prompt.md`), por serem os de maior risco de erro sem supervisão humana.

| Termo | Definição NovaTech (resumo) | Risco se mal interpretado |
|-------|------------------------------|----------------------------|
| **Dias úteis / horas úteis** | Segunda a sexta, excluindo sábados, domingos e **feriados nacionais** (não estaduais/municipais). | Cálculo incorreto de prazos contratuais (devolução, triagem, coleta, reembolso, SLA). |
| **Incidente crítico** | Chamado que atende a **pelo menos um** de 4 critérios taxativos (SLA-2024 §3): carga >R$100k sem status há +6h; carga perigosa com irregularidade documental; >5 chamados do mesmo cliente em 24h; risco à segurança de pessoas. | Uso do conceito genérico de ITIL ou senso comum em vez dos 4 critérios formais. |
| **Devolução (processo padrão)** | Retorno de mercadoria **já entregue**, em até 7 dias úteis, via Portal do Cliente com CT-e e fotos. Exclui cargas perigosas, refrigeradas com ruptura de cadeia de frio, e lacre violado sem documentação de entrega. | Confundir com reembolso, com interceptação em trânsito (PROC-088) ou com sinistro de carga danificada (fluxo via Jurídico). |
| **Frete especial** | Modalidade para cargas **acima de 500 kg**: `Valor base × Multiplicador regional × Fator de peso`. Não é frete "urgente" ou "premium". | Interpretar "especial" como prioritário; misturar multiplicadores/prazos de PROC-042 v1 e v2. |
| **Carga perigosa** | Classes 1 a 6 da ANTT (Resolução nº 5.947/2021) — critério regulatório, não julgamento subjetivo. | Uso de definição popular ("coisas que explodem"), afetando elegibilidade de devolução e SLA de incidente crítico. |
| **Tier de cliente** | Apenas **Gold**, **Silver** ou **Standard**, por contrato anual ou operações/mês. **Não existe Platinum.** | Aceitar tier inexistente sem checar contrato; confundir tier (SLA contratual) com prioridade de chamado. |
| **SLA de resposta vs. resolução** | Resposta = primeiro retorno ao cliente (mesmo "estamos verificando"). Resolução = problema efetivamente resolvido. São métricas independentes. | Tratar como sinônimos ou informar apenas uma ao explicar SLA de um tier. |
| **Relógio de SLA (pausa/não-pausa)** | Pausa fora do horário comercial (08h–18h, dias úteis) para chamados gerais de qualquer tier. **Não pausa** para incidentes críticos de clientes Gold. | Apresentar SLA como prazo corrido; aplicar regra de não-pausa a todos os tiers. |
| **Triagem (devolução)** | Etapa interna (4h úteis, POL-001 §3.3) de verificação de elegibilidade — **não é** o SLA de primeira resposta do tier. | Confundir triagem (4h fixo) com SLA de resposta (varia por tier: 2h/4h/8h). |
| **Multiplicador regional / Fator de peso** | Existem **duas tabelas vigentes sem hierarquia formal** (PROC-042 v1 e v2). A versão aplicável depende da data de abertura do chamado (corte: 01/12/2023). | Apresentar apenas uma tabela sem sinalizar a ambiguidade documental. |
| **Valor base** | Tarifa mensal publicada em planilha interna — **não disponível na base documental do assistente**. | Estimar ou inventar um valor para completar um cálculo de frete. |
| **CT-e** | Documento fiscal obrigatório; sem ele, o processo de devolução não pode ser aberto. | Tratar como nota fiscal genérica. |
| **Cadeia de frio (ruptura)** | Rompida quando a temperatura sai da faixa por **mais de 30 minutos contínuos**, conforme sensor IoT. | Considerar qualquer variação de temperatura como ruptura, sem o critério de tempo/evidência. |
| **Lacre violado** | Inelegível para devolução padrão, **exceto** se documentado no ato da entrega com assinatura do motorista e do recebedor. | Tratar sempre como inelegível, ignorando a exceção documentada. |
| **Reembolso vs. crédito** | Ambos previstos na POL-001, sem critério documentado de quando aplicar cada um. | Afirmar categoricamente que o cliente receberá reembolso em dinheiro. |
| **Prazo expirado** | Solicitação após 7 dias úteis — não elegível pelo processo padrão, mas encaminhável ao Comercial para negociação caso a caso. | Afirmar que o cliente "não pode mais devolver" sem mencionar a via de negociação. |
| **Frete expresso (cargas perigosas)** | Mencionado apenas no FAQ (informal), sem documento normativo (POL/PROC). | Apresentar como política formal estabelecida. |
| **Frete padrão (<500kg)** | Existe operacionalmente, mas **sem documento na base** que defina suas regras. | Inferir regras do frete padrão a partir do frete especial. |

---

### 3. Restrições que impactam geração de código

Estas regras devem ser refletidas nos contratos de dados e na lógica de `src/functions/query/`, `src/services/` e `src/pipeline/` — não apenas no texto do system prompt (ver coluna "Implementação" da matriz de guardrails: regras marcadas como **Código** ou **Ambos** exigem enforcement determinístico).

1. **`source_document` obrigatório no JSON de retorno** — todo objeto de resposta da API (`response-builder.ts`) deve incluir um campo estruturado por citação, contendo no mínimo: `{ document_id, version, section }`. Este campo é obrigatório mesmo em respostas de baixa confiança (ver item 7). *(G4)*
2. **Flag `source_type`** — cada chunk indexado (`indexer.ts`) deve carregar o metadado `source_type: "normativo" | "informal"` definido na ingestão. A ausência dessa flag deve bloquear a indexação do chunk, não ser tratada como default silencioso. *(G5, N5)*
3. **Metadado de vigência** — chunks de documentos versionados (ex.: PROC-042) devem carregar `status: "vigente" | "obsoleto"` (ADR-0003). O retrieval (`search.ts`) deve priorizar/filtrar por esse metadado antes de montar o contexto; documentos obsoletos permanecem indexados mas marcados, nunca excluídos. *(G2, N2)*
4. **Lista de documentos com contradição pendente** — os 12 documentos sinalizados pelo Compliance devem ser mantidos como configuração no pipeline (não hardcoded no prompt). Ao recuperar um chunk dessa lista, o pipeline injeta automaticamente uma nota de divergência no contexto antes de repassar ao modelo. *(G6, F5)*
5. **Threshold de relevância de retrieval** — `search.ts` deve calcular e expor o score de relevância dos chunks retornados. Score abaixo do threshold configurado deve marcar o contexto como `baixa_confianca: true` antes de chegar ao `prompt-builder.ts`, em vez de ser passado como se fosse contexto normal. *(G3, F3)*
6. **Validação pós-geração de valores numéricos** — `response-validator.ts` deve verificar se todo número presente na resposta candidata (prazos, multiplicadores, percentuais) aparece literalmente em algum chunk do contexto usado. Números sem ancoragem no contexto devem sinalizar risco de alucinação e bloquear/reprocessar a resposta. *(N4)*
7. **Bloqueio de negativa não verificada** — `response-validator.ts` deve interceptar respostas candidatas contendo padrões de negativa ("não encontrei", "não há informação", "não consta") quando o tópico da pergunta mapear para um documento normativo conhecido, disparando nova tentativa de recuperação com query reformulada antes de finalizar. *(N3, F3)*
8. **Detecção de categoria de carga ausente** — para perguntas classificadas como relacionadas a devolução, o pipeline deve checar a presença de termos de categoria de carga (perigosa, refrigerada, lacre) na pergunta do usuário. Ausência desses termos deve acionar injeção de instrução de clarificação (F1) em vez de seguir direto para geração. *(G1, N1, F1)*
9. **Validação de citação com versão** — pós-processamento deve rejeitar/sinalizar qualquer resposta que cite "PROC-042" sem sufixo de versão (`-v1`/`-v2`), e verificar que uma única resposta não mistura campos de versões diferentes. *(N2)*
10. **Enum fechado de tier** — qualquer validação de input ou lógica de negócio que trate tier de cliente (`validator.ts`, `types.ts`) deve usar um enum estrito `Gold | Silver | Standard`. Qualquer valor fora do enum (ex.: "Platinum") deve ser rejeitado na validação, não normalizado silenciosamente. *(N4, N8)*
11. **Log de eventos de baixa confiança** — toda ocorrência de F2, F3, F5 ou F7 deve ser registrada em log estruturado para análise posterior pelo time (relevante para `docs/runbooks/` e para as rodadas de avaliação em `prompts/eval/`).

---

### 4. Referências a specs e documentos do repositório

Esta seção normativa deve ser consultada — e mantida em sincronia — com os seguintes artefatos do repositório (ver Anexo C — Estrutura do Repositório):

| Artefato | Caminho | Relação com esta seção |
|----------|---------|--------------------------|
| Base documental de negócio (fonte de verdade) | `docs/novatech/` | Origem de todas as regras normativas citadas acima (POL-001, PROC-042 v1/v2, SLA-2024, FAQ-Atendimento). |
| Corpus de retrieval | `data/retrieval-corpus/` | Onde os chunks com metadados `source_type` e `status` (vigência) devem ser semeados, conforme item 2 e 3 da seção 3. |
| System prompt do assistente | `prompts/system-prompt.md` | Deve incorporar literalmente o glossário da seção 2 e as regras DEVE/NÃO DEVE/QUANDO EM DÚVIDA da seção 1 que forem de implementação **Prompt** ou **Ambos** (G1, G3, G4, G5, G6, N3, N4, N5, N6, F2, F3, F4, F5, F6). |
| Changelog de prompt | `prompts/prompt-changelog.md` | Toda alteração desta seção que resulte em mudança no system prompt deve ser registrada aqui (data, autor, motivo, resultado esperado). |
| Golden queries de avaliação | `prompts/eval/golden-queries.json` | Deve incluir casos de teste cobrindo os 3 incidentes (I-1, I-2, I-3) e os fallbacks F1–F7, para regressão contínua. |
| Spec de ingestão | `specs/pipeline-ingestao/requirements.md` | Deve referenciar os requisitos de metadado (`source_type`, `status` de vigência, lista de contradições pendentes) descritos na seção 3, itens 2–4. |
| Spec do query endpoint | `specs/query-endpoint/requirements.md` | Deve referenciar o contrato de resposta com `source_document` obrigatório (seção 3, item 1) e a lógica de threshold/validação (itens 5–9). |
| Spec do bot do Teams | `specs/teams-bot/requirements.md` | Deve refletir o formato de exibição do aviso de baixa confiança e da escalação ao supervisor (fallback F7) no Adaptive Card (`src/bot/cards/response-card.ts`). |
| Spec do painel web | `specs/painel-web/requirements.md` | Deve prever a exibição visível do `source_document`, `source_type` e eventuais avisos de divergência (F5) na interface. |
| ADR-0002 — Estratégia de contexto | `docs/adr/` | Define o orçamento de tokens (system prompt + chunks) dentro do qual o glossário da seção 2 e as regras da seção 1 precisam caber — priorizar os termos listados como "a incluir no system prompt". |
| ADR-0003 — Documentos contraditórios | `docs/adr/` | Base técnica para os itens 3 e 4 da seção 3 (metadado de vigência e lista de contradições pendentes). |
| ADR-0004 — Chunking de tabelas | `docs/adr/` | Relevante porque as tabelas de multiplicadores (PROC-042) e de SLA (SLA-2024) são justamente os pontos de maior risco de chunking incorreto — qualquer regressão de chunking nessas tabelas deve ser tratada como risco direto aos incidentes I-2 e I-3. |
| Skill de teste de integração | `skills/domain/testing-patterns.md`, `skills/artifact/create-integration-test.md` | Devem incorporar os cenários de teste desta seção (categoria de carga ausente, versão de PROC-042 ambígua, chunk de baixa confiança) como padrões reutilizáveis de teste. |
| Fixtures de teste | `tests/fixtures/queries.ts`, `tests/fixtures/expected-responses.ts` | Devem conter casos espelhando I-1, I-2 e I-3 e as respostas esperadas segundo as regras desta seção. |

> **Nota de manutenção:** esta seção deve ser revisada sempre que (a) novos documentos forem adicionados a `docs/novatech/` e ao corpus de retrieval, (b) o Compliance resolver alguma das 12 contradições pendentes, ou (c) um novo incidente for identificado e adicionado a `guardrails-matriz-incidentes.md`. Toda revisão deve gerar entrada correspondente em `prompts/prompt-changelog.md`.
