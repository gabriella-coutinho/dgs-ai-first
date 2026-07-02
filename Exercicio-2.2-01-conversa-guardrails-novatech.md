# Conversa — Guardrails do Assistente NovaTech

> Registro completo da conversa entre Product Specialist e assistente de IA, incluindo prompts e respostas.  
> Projeto: Assistente de IA NovaTech  
> Fase: Estruturação de guardrails  

---

## Turno 1

### Prompt

**Papel:** Product Specialist analisando a documentação da NovaTech para um projeto de assistente de IA.

**Cenário do projeto:**
O projeto foi aprovado, o discovery está concluído e a fase de entendimento produziu artefatos concretos. Agora o time precisa estruturar o ambiente, os padrões e os artefatos que vão governar o desenvolvimento.

O que foi definido na fase anterior:
- Modelo LLM: Azure OpenAI (GPT-4o) — escolhido pela integração com o ecossistema Microsoft da NovaTech e pela janela de 128K tokens (ADR-0001).
- Pipeline de RAG: Azure AI Search + Azure OpenAI. O protótipo open-source (ChromaDB + sentence-transformers) validou a abordagem e identificou problemas de chunking em tabelas (ADR-0004).
- Estratégia de contexto: context budget de ~4K tokens para system prompt + ~8K para chunks (5 chunks de ~1.500 tokens) + pergunta + histórico limitado a 3 turnos (ADR-0002).
- Documentos contraditórios: metadado de vigência no pipeline; prompt instrui o modelo a priorizar versão mais recente; documentos obsoletos marcados, não excluídos (ADR-0003).
- Integração: Microsoft Teams (bot) + painel web interno.
- Base documental: 847 documentos válidos consolidados (12 com contradições pendentes de resolução pelo Compliance), 63 descartados por obsolescência, ~340 eliminados como duplicatas.
- Arquitetura: 4 componentes — (1) pipeline de ingestão, (2) API do assistente (Azure Functions + Azure AI Search + Azure OpenAI), (3) interface no Teams via Bot Framework, (4) painel web interno.
- Stack: TypeScript (backend e bot), React (painel web), Bicep para infraestrutura como código.
- Time: 1 Tech Lead, 2 Desenvolvedores, 1 QA, 1 Product Specialist, 1 Delivery Manager.

**Inputs:**
- Guardrails: (1) Sempre citar fonte. (2) Nunca inventar prazos ou valores. (3) Quando não encontrar resposta, dizer explicitamente. (4) Responder em português formal.
- 3 incidentes simulados onde o assistente falhou durante testes internos:
  1. O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias, quando na verdade cargas perigosas NÃO podem ser devolvidas.
  2. O assistente citou 'PROC-042, seção 2' mas os multiplicadores informados eram da versão 1 (desatualizada), não da v2 (vigente).
  3. O assistente disse 'Não encontrei informação sobre isso' para uma pergunta sobre SLA Gold, mas o documento SLA-2024 estava indexado e continha a resposta.

**Critérios:**
- Os guardrails são específicos ao domínio da NovaTech, não genéricos.
- Cada guardrail é rastreável a um risco concreto (incidente).

**Tarefa:** Com base no cenário, nos inputs, critérios e na documentação da NovaTech em anexo, elabore um documento de guardrails organizado em:
1. DEVE (comportamentos obrigatórios).
2. NÃO DEVE (comportamentos proibidos).
3. QUANDO EM DÚVIDA (comportamentos de fallback).

---

### Resposta

# Documento de Guardrails — Assistente NovaTech

## 1. DEVE (comportamentos obrigatórios)

**G1. Verificar exclusões de elegibilidade antes de confirmar qualquer prazo de devolução.**
Antes de informar o prazo de 7 dias úteis (POL-001, seção 3.1), o assistente deve primeiro checar se a carga se enquadra nas exceções da seção 3.2 (cargas perigosas classes 1-6 ANTT, cargas refrigeradas com rompimento de cadeia de frio, cargas com lacre violado não documentado). Se a categoria da carga não estiver explícita na pergunta do usuário, o assistente deve perguntar a categoria antes de afirmar qualquer prazo.
*Rastreabilidade: Incidente 1 — assistente informou 7 dias para carga perigosa, que é inelegível.*

**G2. Resolver conflito de versão antes de citar PROC-042, usando o metadado de vigência (ADR-0003).**
Ao responder sobre frete especial, o assistente deve identificar qual documento é o vigente (PROC-042-v2, conforme metadado de vigência do pipeline) e citar explicitamente a versão consultada (ex.: "PROC-042-v2, seção 2.1"), nunca apenas "PROC-042". Se o metadado de vigência não estiver disponível ou for ambíguo para o chunk recuperado, o assistente deve declarar a existência de duas versões e indicar qual está sendo usada e por quê.
*Rastreabilidade: Incidente 2 — citação genérica "PROC-042, seção 2" levou a multiplicadores desatualizados (v1).*

**G3. Distinguir explicitamente entre "documento não encontrado" e "informação não recuperada pelo pipeline".**
Antes de declarar "não encontrei informação sobre isso", o assistente deve confirmar que a busca cobriu os 5 chunks recuperados e que nenhum deles contém a resposta. Para tópicos com documento normativo dedicado e conhecido (ex.: SLA-2024 para perguntas sobre SLA, POL-001 para devolução, PROC-042/v2 para frete especial), o assistente deve tratar a ausência de resposta como sinal de possível falha de recuperação, não de lacuna documental — e sinalizar isso internamente (log) em vez de simplesmente responder negativamente ao usuário sem essa verificação.
*Rastreabilidade: Incidente 3 — assistente disse "não encontrei" sobre SLA Gold apesar de SLA-2024 estar indexado.*

**G4. Citar a fonte com identificador de documento e versão em toda resposta baseada em conteúdo normativo.**
Toda resposta que use POL-001, PROC-042/v2, SLA-2024 ou FAQ-Atendimento deve indicar o nome do documento, a versão (quando aplicável) e, se possível, a seção.
*Rastreabilidade: guardrail genérico (1) do input, especializado para o domínio.*

**G5. Sinalizar a natureza informal do FAQ-Atendimento sempre que ele for usado como fonte.**
Quando o pipeline recuperar conteúdo do FAQ-Atendimento, o assistente deve explicitar que se trata de conhecimento prático não validado por Compliance/Operações e — quando o mesmo tópico existir em documento normativo (POL/PROC/SLA) — priorizar o normativo e usar o FAQ apenas como complemento, nunca como única fonte para prazos, valores ou regras de elegibilidade.
*Rastreabilidade: classificação do próprio FAQ como "não validado por Compliance ou Operações".*

**G6. Tratar documentos com contradições pendentes (12 documentos sinalizados) com aviso explícito de divergência.**
Quando a resposta depender de um dos 12 documentos com contradição pendente de resolução pelo Compliance, o assistente deve informar ao usuário que há divergência documental não resolvida, apresentar a versão vigente conforme metadado, e recomendar confirmação com a área responsável quando o valor envolvido for relevante (ex.: frete especial, seguro de carga).
*Rastreabilidade: ADR-0003 + Incidente 2.*

---

## 2. NÃO DEVE (comportamentos proibidos)

**N1. Não deve afirmar prazo de devolução para uma carga sem confirmar primeiro que ela não pertence a categoria de exceção.**
Nunca aplicar diretamente o prazo geral de 7 dias (seção 3.1 da POL-001) sem verificar a seção 3.2.
*Rastreabilidade: Incidente 1.*

**N2. Não deve citar PROC-042 sem especificar a versão, nem misturar multiplicadores/fatores de peso/prazos de versões diferentes em uma mesma resposta.**
É proibido apresentar, por exemplo, o fator de peso da v1 (1.0/1.2/1.5) junto com o prazo adicional da v2 (+3 dias úteis) ou vice-versa.
*Rastreabilidade: Incidente 2.*

**N3. Não deve declarar ausência de informação sem antes confirmar, dentro da própria resposta, que os chunks recuperados foram revisados quanto ao tópico perguntado.**
Está proibido responder "não encontrei informação" como resposta padrão para perguntas dentro do domínio coberto pelos 5 documentos-base (devolução, frete especial, SLA, atendimento) sem essa verificação.
*Rastreabilidade: Incidente 3.*

**N4. Não deve inventar prazos, valores, percentuais ou tiers que não constem explicitamente nos documentos recuperados.**
Inclui, mas não se limita a: tiers além de Gold/Silver/Standard (ex.: "Platinum" — ver FAQ item 15), percentuais de seguro de carga não confirmados em documento formal, e prazos de processos sem documentação normativa (ex.: carga danificada em trânsito, descrita apenas no FAQ item 38).
*Rastreabilidade: guardrail genérico (2) do input + gaps identificados na documentação (seguro de carga, carga danificada).*

**N5. Não deve apresentar conteúdo do FAQ-Atendimento com o mesmo peso de evidência de um documento normativo (POL/PROC/SLA), nem omitir que a fonte é informal.**
*Rastreabilidade: classificação do próprio FAQ como "não validado por Compliance ou Operações".*

**N6. Não deve resolver sozinho casos que a documentação explicitamente encaminha a outras áreas** (ex.: cargas perigosas → Gestão de Riscos ramal 4500; sinistros/carga danificada → sinistros@novatech.com.br; descontos fora da regra automática → Comercial). O assistente não deve simular uma decisão dessas áreas nem dar respostas definitivas em nome delas.
*Rastreabilidade: POL-001 seção 3.2, FAQ itens 3, 38, 45.*

---

## 3. QUANDO EM DÚVIDA (comportamentos de fallback)

**F1. Categoria de carga não informada em pergunta sobre devolução →** perguntar explicitamente a categoria da carga (perigosa, refrigerada, lacre violado, ou nenhuma das anteriores) antes de informar prazo ou condições.

**F2. Pergunta sobre frete especial sem indicação clara de qual versão usar (ex.: chamado aberto antes de 01/12/2023) →** informar ambas as regras (v1 e v2), indicar a data de transição definida na seção 5 do PROC-042-v2, e orientar o usuário a confirmar a data de abertura do chamado/contrato.

**F3. Recuperação retorna chunks de baixa confiança/relevância para um tópico que deveria ter documento normativo conhecido (SLA, devolução, frete) →** antes de responder negativamente, indicar que a busca não localizou trecho suficientemente relevante e oferecer reformulação da pergunta ou indicar o documento esperado (ex.: "consulte SLA-2024") em vez de simplesmente negar a existência da informação.

**F4. Pergunta cuja única fonte disponível é o FAQ-Atendimento, sem correspondência em documento normativo (ex.: seguro de carga, carga danificada, frete expresso para carga perigosa) →** responder com a informação do FAQ, mas explicitar claramente que é prática informal não validada, e recomendar confirmação com a área responsável (Comercial, Jurídico/sinistros, ou Compliance, conforme o caso) antes de qualquer decisão com impacto financeiro ou contratual.

**F5. Pergunta envolve um dos 12 documentos com contradição pendente de resolução pelo Compliance →** responder com base na versão vigente conforme metadado, sinalizar a existência de divergência documental não resolvida, e recomendar confirmação formal se o caso envolver valores ou compromissos com o cliente.

**F6. Pergunta foge totalmente do escopo da base documental (ex.: tier "Platinum", processos não documentados) →** declarar explicitamente que a informação não existe na documentação oficial da NovaTech, sem especular, e orientar o encaminhamento adequado (ex.: Comercial para verificação de contrato).

---

## Turno 2

### Prompt

Faça uma análise no documento de guardrails gerado e avalie se há informações erradas ou que podem gerar confusão.

---

### Resposta

Revisei o documento de guardrails comparando cada afirmação com o Anexo A. A estrutura geral está bem ancorada nos incidentes e na documentação, mas encontrei alguns pontos que geram inconsistência interna ou inferências não sustentadas pela fonte.

**1. Contradição entre G5 e F4 sobre o FAQ como fonte de valores.**
G5 afirma que o FAQ-Atendimento nunca deve ser usado "como única fonte para prazos, valores ou regras de elegibilidade." Já F4 instrui o assistente a responder usando o FAQ exatamente nesses casos — quando não há documento normativo equivalente. Essas duas regras se contradizem como estão escritas: G5 proíbe categoricamente o que F4 depois permite com ressalva. O documento precisa esclarecer a diferença entre "usar FAQ como única fonte disponível, com aviso explícito de informalidade" (o que F4 descreve corretamente) e "usar FAQ sem aviso, como se fosse normativo" (o que G5 quer proibir). Sugiro reescrever G5 para deixar claro que a restrição é sobre *omitir o caráter informal*, não sobre *usar o FAQ quando é a única fonte existente*.

**2. G6 assume uma equivalência não confirmada pela documentação.**
G6 trata a contradição PROC-042 v1/v2 como parte dos "12 documentos com contradições pendentes de resolução pelo Compliance" mencionados no cenário do projeto. Essa é uma inferência razoável, mas não há nada no Anexo A que confirme que o par PROC-042/PROC-042-v2 está necessariamente entre esses 12. Vale marcar isso explicitamente como suposição em vez de apresentar como fato, para não criar uma falsa precisão.

**3. N4 e F4 podem se sobrepor de forma confusa.**
N4 proíbe "inventar" prazos/valores/percentuais que não constem nos documentos — mas os percentuais de seguro de carga (0,3%/0,8%) e o prazo de 48h para carga danificada *constam* no FAQ, só que em documento informal. Como N4 não faz essa distinção, alguém lendo o guardrail isoladamente pode concluir que citar esses números do FAQ já seria "inventar", o que entra em conflito direto com F4. Recomendo adicionar uma ressalva em N4: "não inventar" refere-se a criar números sem qualquer base documental, não a citar valores do FAQ com o devido aviso de informalidade.

**4. Pequena imprecisão em G3 quanto à causa-raiz do Incidente 3.**
G3 trata o problema como uma questão de "verificação antes de declarar ausência de informação", o que é correto, mas o incidente real sugere uma falha de recuperação (retrieval), não necessariamente uma falha de verificação pós-recuperação. O guardrail como está pressupõe que os chunks corretos *foram* recuperados — o que pode não ter sido o caso no incidente original. Vale adicionar no F3 uma ponte mais explícita: G3 deveria remeter ao F3 como o comportamento de fallback correspondente.

**5. Pontos que conferi e estão corretos, sem erro factual:**
- Os fatores de peso e prazos citados em N2 e F2 (v1: 1.0/1.2/1.5, +2 dias; v2: 1.0/1.15/1.4, +3 dias) correspondem exatamente ao Anexo A.
- A data de transição (01/12/2023) em F2 está correta conforme a seção 5 do PROC-042-v2.
- As exceções de devolução em G1/N1 refletem fielmente a seção 3.2 da POL-001.
- A ausência do tier "Platinum" em F6 está corretamente embasada no FAQ item 15.
- O encaminhamento a áreas específicas em N6 está consistente com POL-001 e os itens 3, 38 e 45 do FAQ.

Em resumo, o maior risco de confusão real é o conflito G5 × F4 (merece correção antes de aprovar o documento), seguido pela inferência não sustentada em G6. Os demais pontos são ajustes de precisão, não erros factuais graves.

---

## Turno 3

### Prompt

Atualize o documento de guardrails inserindo ao final uma observação com esses pontos que geram inconsistência interna ou inferências não sustentadas pela fonte.

---

### Resposta

O documento foi atualizado. Além de adicionar a seção 4 com as quatro observações, foram feitos ajustes cirúrgicos nos guardrails afetados — sem alterar nenhum guardrail que estava correto — para que o documento seja internamente consistente:

- **G5** foi reescrito para separar a restrição (omitir o caráter informal) do comportamento permitido (usar o FAQ como única fonte com aviso).
- **G6** recebeu uma nota explícita de que a vinculação ao conjunto dos 12 documentos é inferência, não fato confirmado.
- **N4** foi reescrito para distinguir "inventar sem base alguma" de "citar fonte informal com aviso declarado".
- **G3 e F3** receberam referências cruzadas entre si, formalizando a relação que estava implícita.

Cada observação na seção 4 registra o guardrail afetado, o problema original, a correção aplicada e o status — o que facilita o rastreamento para o time de QA e para o Compliance.

> O documento atualizado está disponível como `guardrails-assistente-novatech.md`.

---

## Turno 4

### Prompt

Com base nos guardrails gerados, conecte cada guardrail a, ao menos, um dos 3 incidentes. Para cada guardrail responda a seguinte pergunta: qual incidente esse guardrail previne?

**Incidentes:**
1. "O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias, quando na verdade cargas perigosas NÃO podem ser devolvidas."
2. "O assistente citou 'PROC-042, seção 2' mas os multiplicadores informados eram da versão 1 (desatualizada), não da v2 (vigente)."
3. "O assistente disse 'Não encontrei informação sobre isso' para uma pergunta sobre SLA Gold, mas o documento SLA-2024 estava indexado e continha a resposta."

**Input:**
- Guardrails: "(1) Sempre citar fonte. (2) Nunca inventar prazos ou valores. (3) Quando não encontrar resposta, dizer explicitamente. (4) Responder em português formal."

**Critérios:**
- Os guardrails são específicos ao domínio da NovaTech, não genéricos.
- A classificação prompt vs código demonstra compreensão de que prompts são probabilísticos e código é determinístico.
- Cada guardrail é rastreável a um risco concreto (incidente).

---

### Resposta

Foi gerado o documento `guardrails-matriz-incidentes.md` com o mapeamento completo de cada guardrail aos incidentes, incluindo:

- **Incidente primário** que cada guardrail previne, com narrativa causal explicando o mecanismo.
- **Incidente secundário** que mitiga, quando aplicável.
- **Classificação de enforcement** (Prompt / Código / Ambos) com justificativa para cada guardrail.
- **Tabela resumo consolidada** com cobertura por incidente.

**Padrões que emergiram do mapeamento:**

O I-1 (carga perigosa / prazo errado) é o mais coberto, com 9 guardrails. Isso reflete que sua causa-raiz é estrutural: o modelo não tem como distinguir "regra geral" de "regra geral com exceções críticas" sem múltiplas camadas de instrução.

O I-2 (versão errada do PROC-042) é o único com guardrails de implementação predominantemente em **código** (G2, N2), porque a resolução de versão é uma decisão binária e objetiva — não faz sentido deixar para o modelo julgar quando o pipeline já tem o metadado de vigência disponível.

O I-3 (SLA Gold não encontrado) é o menos coberto (5 guardrails), consistente com o OBS-04: a raiz pode ser uma falha de retrieval que nenhum guardrail de comportamento resolve sozinho.

**Resumo consolidado:**

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

---

## Turno 5

### Prompt

Analise o documento criado "guardrails matriz incidentes" e avalie se nele estão: classificação de enforcement e rastreabilidade aos incidentes.

---

### Resposta

## Dimensão 1 — Classificação de enforcement (Prompt / Código / Ambos)

**O que o documento faz bem:**

A nota introdutória ("Nota sobre a coluna Implementação") estabelece o critério corretamente: prompts são probabilísticos, código é determinístico, e a distinção entre os dois é o que justifica a existência das três categorias. Cada guardrail tem uma tabela de implementação com justificativa textual, não apenas o rótulo.

**Problema 1 — G2 está classificado como "Código" mas a justificativa descreve "Ambos".**
A tabela de G2 diz `Código`, mas o texto da justificativa diz explicitamente: *"O prompt complementa com a instrução de sempre citar a versão na resposta."* A tabela resumo no final também registra G2 como `Código`, perpetuando a inconsistência. A classificação correta é `Ambos`.

**Problema 2 — F3 e F6 estão classificados como "Código" e "Prompt" respectivamente, mas ambos têm comportamentos que cruzam as duas camadas.**
F3 diz `Código` e justifica com o threshold de relevância — correto. Mas a instrução ao usuário ("oferecer reformulação ou indicar o documento esperado") é comportamental e precisa estar no prompt também. F6 diz `Prompt`, mas a detecção de "fora do escopo" via score baixo é código. Ambos deveriam ser `Ambos`.

**O que está correto:**
A separação entre N1 (`Código`), N2 (`Código`) e N6 (`Prompt`) demonstra compreensão real da distinção — N1 e N2 são verificações binárias e objetivas, enquanto N6 envolve julgamento contextual.

---

## Dimensão 2 — Rastreabilidade aos incidentes

**O que o documento faz bem:**

Todo guardrail tem pelo menos um incidente primário identificado, e vários têm incidentes secundários com explicação de por que a relação é indireta. A seção de cobertura por incidente permite verificar a cobertura de forma invertida. A narrativa causal em cada guardrail vai além da rastreabilidade formal e explica o mecanismo.

**Problema 3 — F4 tem rastreabilidade fraca para I-1.**
F4 é classificado com incidente primário I-1, mas a justificativa trata de seguro de carga, carga danificada e frete expresso — nenhum desses *é* o I-1. Eles *compartilham o mesmo mecanismo de falha*, mas classificar como incidente primário direto foi uma escolha que o próprio texto do guardrail contradiz.

**Problema 4 — G4 tem rastreabilidade invertida.**
O argumento mais sólido para G4 aparece nos incidentes *secundários*. Para I-2, G2 e N2 já cobrem de forma mais direta. O papel único de G4 está nos incidentes I-1 e I-3 (auditabilidade). A classificação de I-2 como primário subestima o papel real de G4.

---

## Resumo dos problemas encontrados

| # | Guardrail afetado | Dimensão | Problema | Severidade |
|---|-------------------|----------|----------|------------|
| P1 | G2 | Enforcement | Classificado como `Código`, mas justificativa descreve comportamento de `Prompt` também | Alta — afeta implementação |
| P2 | F3, F6 | Enforcement | Classificados com apenas uma camada, mas ambos têm as duas | Média — afeta implementação |
| P3 | F4 | Rastreabilidade | Rastreado como prevenção direta de I-1, mas a relação é de mecanismo compartilhado | Baixa — afeta precisão |
| P4 | G4 | Rastreabilidade | Incidente primário (I-2) é coberto por outros guardrails mais diretos; papel único está nos secundários | Baixa — afeta priorização |

Os problemas de enforcement (P1 e P2) têm impacto prático direto: um desenvolvedor lendo G2 como `Código` pode não implementar a instrução de citação no prompt, o que reproduz parcialmente o Incidente 2.

---

## Turno 6

### Prompt

Sendo você uma Product Specialist, quais melhorias e/ou ajustes podem ser feitos no documento? Os problemas encontrados acima são por ausência ou inconsistência de informações previamente passadas?

---

### Resposta

## Parte 1 — Os problemas são por ausência ou inconsistência?

Todos os quatro problemas são de **inconsistência interna**, não de ausência de informação. A evidência está dentro do próprio documento em cada caso:

**P1 (G2 classificado como `Código`):** a justificativa do próprio G2 diz literalmente *"o prompt complementa com a instrução de sempre citar a versão na resposta."* A informação de que havia uma camada de prompt estava lá — o rótulo da tabela simplesmente não foi atualizado para refletir isso.

**P2 (F3 e F6 com classificação incompleta):** a justificativa de F6 diz explicitamente *"o código pode sinalizar o cenário; o prompt define a resposta"* — duas camadas descritas, rótulo de uma só. Em F3, o comportamento de oferecer reformulação ao usuário é descrito no texto, mas a classificação ignora que isso é instrução de prompt.

**P3 (F4 rastreado diretamente ao I-1):** os três incidentes foram fornecidos com descrição precisa. O I-1 é sobre prazo de devolução de carga perigosa. F4 trata de seguro de carga, carga danificada e frete expresso — nenhum desses *é* o I-1. A conexão pelo mecanismo é legítima, mas classificar como incidente primário direto foi uma escolha de redação que o próprio texto do guardrail contradiz.

**P4 (G4 com incidente primário subestimado):** o argumento mais forte para G4 aparece nos incidentes *secundários* do próprio documento. A lógica estava correta — a hierarquia entre primário e secundário é que ficou invertida.

**Conclusão:** não falta nenhuma informação de entrada para corrigir esses pontos. O que falta é um passo de consistência entre o texto da justificativa e o rótulo da classificação.

---

## Parte 2 — Melhorias como Product Specialist

**Separar "previne diretamente" de "previne mecanismo equivalente" na rastreabilidade.**
Hoje o documento usa "incidente primário" e "incidente secundário", mas não distingue entre guardrails que bloqueiam exatamente o cenário do incidente e guardrails que previnem uma classe de falha com o mesmo mecanismo. Uma coluna ou marcação simples (`direto` / `mecanismo equivalente`) na tabela resumo resolveria isso sem reestruturar o documento.

**Adicionar responsável pela implementação na tabela de cada guardrail.**
A classificação Prompt/Código/Ambos diz *o quê* implementar, mas não *quem*. No contexto do projeto — TypeScript no backend, Bot Framework, system prompt gerenciado pelo Product Specialist — prompt e código têm donos diferentes. Uma coluna `Responsável` com valores como `Product Specialist (prompt)`, `Engenharia (pipeline)`, `Ambos` tornaria o documento diretamente acionável em planning.

**Adicionar criticidade de implementação.**
Nem todos os 18 guardrails têm o mesmo impacto se não forem implementados na primeira versão. G1, G2 e N1 endereçam falhas que já aconteceram em testes — são bloqueadores para produção. O documento atual trata todos os guardrails com o mesmo peso, o que dificulta a decisão de priorização do backlog. Uma coluna `Criticidade` com três níveis (`Bloqueador`, `Alta`, `Média`) daria ao Delivery Manager o insumo que falta para sequenciar a implementação.

**Documentar o critério de revisão dos guardrails.**
O documento é um artefato vivo: novos incidentes vão surgir, o Compliance vai resolver as 12 contradições pendentes, a PROC-043 está em revisão. Sem um critério explícito de quando revisar os guardrails, o documento envelhece em silêncio. Uma nota curta no final — definindo gatilhos de revisão como "novo incidente em produção", "resolução de contradição pelo Compliance", "atualização de documento normativo na base" — transforma o documento de entregável pontual em artefato de governança contínua.
