# Matriz de Rastreabilidade — Guardrails × Incidentes

## Referência rápida dos incidentes

| ID | Descrição |
|----|-----------|
| **I-1** | O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias, quando na verdade cargas perigosas NÃO podem ser devolvidas. |
| **I-2** | O assistente citou "PROC-042, seção 2" mas os multiplicadores informados eram da versão 1 (desatualizada), não da v2 (vigente). |
| **I-3** | O assistente disse "Não encontrei informação sobre isso" para uma pergunta sobre SLA Gold, mas o documento SLA-2024 estava indexado e continha a resposta. |

---

## Nota sobre a coluna "Implementação"

A análise distingue duas camadas de enforcement dos guardrails, porque prompts são probabilísticos e código é determinístico:

- **Prompt** — a instrução é passada ao modelo no system prompt; o modelo decide se e como aplicá-la a cada resposta. Apropriado para comportamentos que dependem de interpretação contextual (ex.: tom, priorização de fonte, aviso de informalidade).
- **Código** — a regra é aplicada programaticamente no pipeline (pré ou pós-processamento), independente do julgamento do modelo. Apropriado para verificações objetivas e binárias (ex.: "metadado de versão existe?", "score de relevância está acima do threshold?", "a resposta contém referência a um documento?").
- **Ambos** — a regra tem uma camada de detecção/enforcement no código e uma camada de instrução de comportamento no prompt. O código reduz a probabilidade de falha; o prompt define o que fazer quando o código sinaliza o problema.

---

## Seção 1 — DEVE

### G1 — Verificar exclusões de elegibilidade antes de confirmar prazo de devolução

**Incidente que previne: I-1**

O Incidente 1 aconteceu porque o assistente aplicou diretamente o prazo geral (POL-001, seção 3.1) sem verificar as exceções da seção 3.2. G1 obriga o assistente a confirmar a categoria da carga antes de informar qualquer prazo, impedindo que carga perigosa receba a mesma resposta de carga padrão.

**Incidente secundário que mitiga: I-2** (parcialmente)
Se o usuário perguntar sobre frete especial de carga perigosa, G1 aciona a verificação que levaria ao PROC-043, evitando que o assistente responda com multiplicadores do PROC-042 para um tipo de carga que tem tabela própria.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** detectar palavras-chave de categoria de carga (perigosa, refrigerada, lacre) na pergunta e, se ausentes em perguntas sobre devolução, bloquear a resposta e acionar pergunta de clarificação. **Prompt:** instrui o modelo a priorizar a verificação da seção 3.2 antes de qualquer afirmação sobre prazos. O código garante a triagem; o prompt garante o comportamento correto quando a categoria é informada mas é uma exceção. |

---

### G2 — Resolver conflito de versão antes de citar PROC-042

**Incidente que previne: I-2**

O Incidente 2 aconteceu porque o assistente citou genericamente "PROC-042, seção 2" sem especificar a versão, e os multiplicadores retornados eram os da v1. G2 obriga a resolução da versão via metadado de vigência antes de qualquer citação, e exige que a versão apareça explicitamente na resposta (ex.: "PROC-042-v2, seção 2.1").

| Implementação | Justificativa |
|---------------|---------------|
| **Código** | A resolução de versão deve ser determinística: o pipeline de recuperação deve filtrar ou ranquear chunks pelo metadado de vigência (ADR-0003) antes de servir o contexto ao modelo. Se ambas as versões forem recuperadas, o código deve marcar qual é a vigente e qual é a obsoleta antes de passá-las ao prompt. Não é seguro deixar essa resolução apenas para o modelo, pois ele pode não ter visibilidade do metadado ou pode ignorá-lo sob pressão de contexto. O prompt complementa com a instrução de sempre citar a versão na resposta. |

---

### G3 — Distinguir "documento não encontrado" de "informação não recuperada pelo pipeline"

**Incidente que previne: I-3**

O Incidente 3 aconteceu porque o assistente tratou uma falha de retrieval como ausência de informação na base, respondendo negativamente ao usuário quando o documento correto estava indexado. G3 instrui o assistente a tratar respostas negativas sobre tópicos com documentos normativos conhecidos como sinal de falha de recuperação, não de lacuna documental — e a redirecionar para o fallback F3 em vez de confirmar a ausência.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** implementar threshold de relevância no retrieval; se o score dos chunks retornados para perguntas sobre SLA, devolução ou frete ficar abaixo do threshold, o pipeline deve sinalizar "baixa confiança na recuperação" antes de passar o contexto ao modelo — e registrar o evento em log para análise posterior. **Prompt:** instrui o modelo a tratar ausência de chunks relevantes para tópicos cobertos pela base como sinal de falha de recuperação, não de lacuna documental, e a acionar o comportamento de F3 (oferecer reformulação ou indicar o documento esperado). |

---

### G4 — Citar fonte com identificador de documento e versão em toda resposta normativa

**Incidente que previne: I-2**

No Incidente 2, a citação genérica "PROC-042, seção 2" impediu o usuário de verificar qual versão estava sendo usada. G4 resolve isso na camada de resposta: a versão deve aparecer explicitamente (ex.: "PROC-042-v2"). A citação rastreável é também condição de auditabilidade para I-1 (permite identificar de qual seção veio a informação sobre prazo) e I-3 (permite identificar se o documento citado é o correto).

**Incidente secundário que mitiga: I-1 e I-3**
Respostas com citação rastreável são mais fáceis de auditar e corrigir. Se o assistente tivesse citado "POL-001, seção 3.1" no Incidente 1, a equipe teria identificado imediatamente que a seção 3.2 não foi consultada. Se tivesse citado "SLA-2024, seção 2" no Incidente 3, a resposta negativa teria sido imediatamente contraditória.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** pós-processamento pode validar se a resposta contém referência a pelo menos um dos documentos-base conhecidos (POL-001, PROC-042-v2, SLA-2024, FAQ-Atendimento) quando o contexto recuperado for de conteúdo normativo. Ausência de citação pode disparar log de alerta ou reprocessamento. **Prompt:** instrui o modelo sobre o formato esperado de citação (nome do documento, versão, seção). |

---

### G5 — Sinalizar natureza informal do FAQ sempre que usado como fonte

**Incidente que previne: I-1** (causa estrutural)

O Incidente 1 tem raiz em como o assistente trata fontes de diferentes pesos. O FAQ item 3 orienta a encaminhar carga perigosa ao ramal 4500, mas um modelo sem instrução explícita pode tratar o FAQ com o mesmo peso da POL-001 — e a POL-001 tem o prazo de 7 dias em posição mais saliente que as exceções. G5 mitiga isso ao forçar o modelo a priorizar documentos normativos e a sinalizar quando está usando o FAQ.

**Incidente secundário que mitiga: I-2**
O FAQ item 8 menciona a existência de duas versões da PROC-042 e orienta usar a v2. Se o assistente priorizasse o normativo (G5) sobre o FAQ, teria chegado diretamente à PROC-042-v2 com os multiplicadores corretos.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** o pipeline pode marcar cada chunk com metadado de tipo de fonte (normativo / informal) na ingestão. Chunks do FAQ recebem flag `source_type: informal`. O código passa essa flag ao prompt para que o modelo saiba, sem depender de inferência, que a fonte é informal. **Prompt:** instrui o modelo a incluir aviso de informalidade quando `source_type: informal` estiver presente no contexto. |

---

### G6 — Tratar documentos com contradições pendentes com aviso de divergência

**Incidente que previne: I-2**

O Incidente 2 é diretamente causado pela coexistência de PROC-042 v1 e v2 sem hierarquia clara. G6 complementa G2: enquanto G2 resolve qual versão usar (via metadado), G6 garante que o usuário seja informado sobre a existência da divergência documental sempre que ela for relevante para a resposta, reduzindo o risco de o usuário tomar decisões baseado em uma versão sem saber que existe outra.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** o pipeline pode manter uma lista de IDs de documentos com contradições pendentes (os 12 sinalizados pelo Compliance). Quando um chunk desses documentos for recuperado, o código injeta automaticamente uma nota de contexto informando a contradição antes de passar o chunk ao modelo. **Prompt:** instrui o modelo a incluir o aviso na resposta ao usuário quando a nota de contradição estiver presente no contexto. |

---

## Seção 2 — NÃO DEVE

### N1 — Não deve afirmar prazo de devolução sem confirmar a categoria da carga

**Incidente que previne: I-1**

É o espelho negativo de G1: enquanto G1 define o comportamento obrigatório (verificar antes de afirmar), N1 define o comportamento proibido (afirmar sem verificar). No Incidente 1, o assistente fez exatamente o que N1 proíbe — aplicou a regra geral a uma carga que é exceção explícita na mesma política.

| Implementação | Justificativa |
|---------------|---------------|
| **Código** | Detecção de padrão determinística: se a pergunta contém termos associados a devolução e a resposta candidata contém o prazo de 7 dias, o pipeline verifica se a categoria da carga foi confirmada como não-exceção. Se não foi, bloqueia a resposta e solicita clarificação. Mais confiável que depender do modelo para lembrar de não fazer algo. |

---

### N2 — Não deve citar PROC-042 sem versão, nem misturar versões

**Incidente que previne: I-2**

N2 é o espelho negativo de G2. O Incidente 2 produziu uma resposta que misturava a identificação genérica "PROC-042" com multiplicadores da v1 — exatamente o que N2 proíbe. A proibição de misturar versões (ex.: fator de peso da v1 com prazo adicional da v2) previne um cenário mais sutil que o Incidente 2 original: o assistente citar a v2 corretamente mas usar dados mistos das duas versões.

| Implementação | Justificativa |
|---------------|---------------|
| **Código** | Pós-processamento pode verificar se a resposta contém referência a "PROC-042" sem sufixo de versão, e rejeitar ou sinalizar para reprocessamento. Adicionalmente, se o contexto contiver chunks de ambas as versões, o pipeline deve filtrar os da versão obsoleta antes de passar ao modelo. |

---

### N3 — Não deve declarar ausência de informação sem verificar os chunks recuperados

**Incidente que previne: I-3**

N3 é o espelho negativo de G3. No Incidente 3, o assistente declarou ausência de informação sem essa verificação. N3 proíbe o atalho de responder negativamente como default para perguntas dentro do domínio coberto pelos documentos-base.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** se a resposta candidata contiver padrões de negativa ("não encontrei", "não há informação", "não consta") e o tópico da pergunta mapear para um documento normativo conhecido, o pipeline deve interceptar e acionar nova tentativa de recuperação com query reformulada antes de finalizar a resposta. **Prompt:** instrui o modelo a não emitir negativas sem sinalizar a possibilidade de falha de recuperação. |

---

### N4 — Não deve inventar prazos, valores, percentuais ou tiers sem base documental

**Incidente que previne: I-1 e I-2**

Para I-1: o prazo de 7 dias informado para carga perigosa não é "invenção" no sentido estrito (ele existe na POL-001, seção 3.1), mas a aplicação a uma categoria explicitamente excluída produz o mesmo efeito prático de um valor inventado — informação incorreta para aquele contexto. N4 cobre o caso em que o modelo geraria um prazo sem qualquer ancoragem documental.

Para I-2: se o modelo não tivesse acesso a nenhuma versão da PROC-042, poderia ter inferido multiplicadores baseado em padrões gerais — N4 proíbe esse comportamento.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** para valores numéricos críticos (prazos em dias, multiplicadores, percentuais), o pós-processamento pode verificar se o número presente na resposta aparece em algum chunk do contexto. Se não aparecer, sinaliza como risco de alucinação. **Prompt:** instrui o modelo a não inferir ou interpolar valores numéricos não presentes explicitamente nos documentos recuperados. |

---

### N5 — Não deve equiparar FAQ a documento normativo nem omitir seu caráter informal

**Incidente que previne: I-1**

O FAQ item 3 menciona que carga perigosa "oficialmente não pode" ser devolvida pelo processo padrão, mas abre exceção para "tratamento especial". Se o assistente tratasse o FAQ com o mesmo peso da POL-001, poderia usar essa ambiguidade do FAQ para suavizar a resposta correta — que é clara: carga perigosa não é elegível pelo processo padrão, e o cliente deve ligar ao ramal 4500. N5 impede que a fonte informal dilua a clareza da fonte normativa.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** metadado `source_type: informal` no chunk do FAQ (ver G5). O pipeline pode bloquear respostas que usam FAQ como fonte primária quando há chunk normativo disponível sobre o mesmo tópico. **Prompt:** instrui o modelo sobre a hierarquia de fontes. |

---

### N6 — Não deve resolver casos que a documentação encaminha a outras áreas

**Incidente que previne: I-1**

No Incidente 1, além do prazo errado, há um segundo risco: o assistente poderia ter tentado resolver o caso de carga perigosa dentro do fluxo de atendimento padrão, em vez de encaminhar ao ramal 4500 (Gestão de Riscos), conforme POL-001, seção 3.2. N6 proíbe esse comportamento — o assistente deve encaminhar, não decidir.

**Incidente secundário que mitiga: I-2**
Para casos de frete especial com contradição de versão não resolvida (ver G6), o assistente não deve tentar calcular o frete sozinho — deve informar a contradição e encaminhar ao Comercial.

| Implementação | Justificativa |
|---------------|---------------|
| **Prompt** | O encaminhamento correto (qual ramal, qual e-mail, qual área) depende de interpretação contextual — não é verificável deterministicamente pelo código sem regras de roteamento complexas. O prompt é o veículo mais adequado para instruir o modelo sobre os pontos de handoff definidos na POL-001 e no FAQ. Uma tabela de roteamento no system prompt (categoria → encaminhamento) reduz ambiguidade. |

---

## Seção 3 — QUANDO EM DÚVIDA (fallbacks)

### F1 — Categoria de carga não informada → perguntar antes de responder sobre devolução

**Incidente que previne: I-1**

F1 é o comportamento de fallback que G1 aciona: quando a categoria da carga não está explícita na pergunta, a resposta correta não é assumir que é carga padrão — é perguntar. No Incidente 1, o assistente assumiu e errou. F1 institucionaliza a pergunta de clarificação como comportamento padrão nesse cenário de ambiguidade.

| Implementação | Justificativa |
|---------------|---------------|
| **Código** | A condição de acionamento de F1 é determinística: ausência de termos de categoria de carga em perguntas sobre devolução. O código pode detectar esse padrão e injetar instrução de clarificação no prompt antes da geração da resposta, garantindo que o modelo faça a pergunta em vez de assumir. |

---

### F2 — PROC-042 sem indicação clara de versão → informar ambas e orientar confirmação

**Incidente que previne: I-2**

F2 trata o cenário em que G2 não resolve completamente a versão — por exemplo, quando o metadado de vigência é ambíguo ou quando o usuário menciona um chamado aberto antes de 01/12/2023 (data de transição da PROC-042-v2, seção 5). Nesse caso, a resposta correta não é escolher uma versão arbitrariamente, mas apresentar ambas com a data de corte e orientar o usuário a confirmar.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** quando o pipeline recuperar chunks de ambas as versões sem metadado de vigência conclusivo, ou quando a pergunta mencionar data anterior a 01/12/2023, injetar flag de ambiguidade de versão no contexto. **Prompt:** instrui o modelo a acionar o comportamento de F2 quando a flag estiver presente. |

---

### F3 — Chunks de baixa confiança para tópico normativo → oferecer reformulação, não negar

**Incidente que previne: I-3**

F3 é o comportamento de resposta ao usuário quando G3 detecta possível falha de recuperação. Em vez de "não encontrei informação" (comportamento do Incidente 3), o assistente deve indicar que a busca não localizou trecho relevante e oferecer alternativa (reformulação da pergunta ou indicação do documento esperado). A distinção é crítica: o problema no Incidente 3 não foi ausência de informação na base — foi falha de retrieval.

| Implementação | Justificativa |
|---------------|---------------|
| **Código** | O threshold de relevância (score mínimo dos chunks retornados) é uma decisão de engenharia, não do modelo. O código que calcula o score já sabe se o retrieval foi bem-sucedido; cabe ao pipeline usar essa informação para sinalizar ao modelo o nível de confiança da recuperação, em vez de simplesmente passar chunks de baixa relevância como se fossem bons. |

---

### F4 — Única fonte é o FAQ → responder com aviso de informalidade e recomendar confirmação

**Incidente que previne: I-1** (causa estrutural)

F4 previne que o silêncio documental vire uma lacuna não sinalizada. Para tópicos sem documento normativo (seguro de carga, carga danificada), o assistente deve usar o FAQ — mas sinalizando seu caráter informal. Sem F4, o modelo poderia ou responder com o FAQ sem aviso (risco de I-1: informação informal tratada como definitiva) ou negar a informação por não ter fonte normativa (risco de I-3).

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** quando o único chunk relevante recuperado tiver `source_type: informal` e não houver chunk normativo sobre o mesmo tópico, o pipeline injeta instrução de usar F4. **Prompt:** instrui o modelo sobre o texto do aviso de informalidade e a recomendação de confirmação. |

---

### F5 — Documento com contradição pendente → citar versão vigente e sinalizar divergência

**Incidente que previne: I-2**

F5 é o fallback de G6 para casos em que o assistente está diante de um dos documentos com contradição pendente. O comportamento esperado é idêntico ao que teria evitado o Incidente 2: citar a versão vigente (conforme metadado), alertar sobre a divergência, e recomendar confirmação para decisões com impacto financeiro.

| Implementação | Justificativa |
|---------------|---------------|
| **Ambos** | **Código:** a lista dos documentos com contradições pendentes é um dado gerenciável pelo pipeline (não pelo modelo). O código que recupera os chunks pode injetar automaticamente a nota de divergência quando o chunk pertence a um documento nessa lista. **Prompt:** instrui o modelo a incluir o aviso na resposta e a recomendar confirmação. |

---

### F6 — Pergunta fora do escopo documental → declarar ausência sem especular

**Incidente que previne: I-1, I-2 e I-3** (cobertura transversal)

F6 é o guardrail de último recurso: quando a pergunta não encontra nenhuma ancoragem nos documentos-base, o assistente não deve especular — deve declarar a ausência e encaminhar. Previne alucinações que poderiam reproduzir os três incidentes: inventar um prazo para carga perigosa (I-1), inventar multiplicadores sem base (I-2), ou fingir uma resposta sobre SLA para um tier inexistente (I-3).

| Implementação | Justificativa |
|---------------|---------------|
| **Prompt** | A definição de "fora do escopo" é semântica e contextual — o código pode detectar score muito baixo em todos os chunks retornados, mas a instrução de não especular e de oferecer o encaminhamento adequado é comportamental e deve estar no prompt. O código pode sinalizar o cenário; o prompt define a resposta. |

---

## Resumo consolidado

| Guardrail | Incidente primário | Incidente secundário | Implementação |
|-----------|-------------------|---------------------|---------------|
| G1 | I-1 | I-2 | Ambos |
| G2 | I-2 | — | Código |
| G3 | I-3 | — | Ambos |
| G4 | I-2 | I-1, I-3 | Ambos |
| G5 | I-1 | I-2 | Ambos |
| G6 | I-2 | — | Ambos |
| N1 | I-1 | — | Código |
| N2 | I-2 | — | Código |
| N3 | I-3 | — | Ambos |
| N4 | I-1, I-2 | — | Ambos |
| N5 | I-1 | — | Ambos |
| N6 | I-1 | I-2 | Prompt |
| F1 | I-1 | — | Código |
| F2 | I-2 | — | Ambos |
| F3 | I-3 | — | Código |
| F4 | I-1 | I-3 | Ambos |
| F5 | I-2 | — | Ambos |
| F6 | I-1, I-2, I-3 | — | Prompt |

**Cobertura por incidente:**
- I-1 (carga perigosa / prazo errado): G1, G5, N1, N4, N5, N6, F1, F4, F6
- I-2 (PROC-042 versão errada): G2, G4, G6, N2, N4, N6, F2, F5, F6
- I-3 (SLA Gold não encontrado): G3, G4, N3, F3, F4, F6
