# requirements.md — Query Endpoint
## Assistente de IA NovaTech

**Componente:** Query Endpoint (API do assistente — Azure Functions + Azure AI Search + Azure OpenAI)
**Fase:** Estruturação do ambiente de desenvolvimento
**Responsável pelo documento:** Product Specialist
**Versão:** v3 — reestruturado para separar requisitos de produto de decisões técnicas pendentes
**Data:** Junho/2026
**Status:** Outcomes, scope boundaries e constraints de negócio aprovados pelo Product Specialist. Constraints técnicas e verification criteria dependentes aguardam resolução das Open Questions (Seção 6) pelos donos indicados.
**Documentos de referência:** bounded-contexts-assistente-novatech.md · linguagem-ubiqua-novatech.md · ADR-0001 · ADR-0002 · ADR-0003 · ADR-0004

---

## Sobre este documento

Este documento especifica os requisitos do **query endpoint** — o componente responsável por receber uma pergunta do atendente, recuperar os chunks relevantes da base documental via RAG, e retornar uma resposta fundamentada com citação de fonte.

O query endpoint não inclui o pipeline de ingestão de documentos, a interface do Teams, nem o painel web interno. Esses componentes têm requisitos separados.

### Nota sobre esta versão

As duas rodadas de revisão técnica anteriores (tech-lead-review-requirements.md e tech-lead-review-requirements-v2.md) identificaram um padrão recorrente: ambiguidades sinalizadas como bloqueadoras exigiam decisões de engenharia — limites de token, definição de métricas de latência sob streaming, contrato de schema de API — que não são da competência do Product Specialist. Nas versões v1 e v2 deste documento, essas decisões foram resolvidas por suposição para "destravar" o documento, o que produziu constraints registradas como definitivas sem o aval de quem detém a decisão técnica.

Esta versão corrige isso: as constraints técnicas que dependiam de decisão de engenharia foram revertidas para o estado de pergunta aberta, registradas na Seção 6 com dono explícito. Os verification criteria que dependiam dessas constraints estão marcados como **bloqueados** até a Open Question correspondente ser resolvida.

O que permanece resolvido nesta versão são os outcomes, os scope boundaries e as constraints de negócio — que são, de fato, da competência do Product Specialist porque derivam de decisão de produto, política documentada da NovaTech ou ADR já aprovada pelo Tech Lead.

---

## 1. Outcomes

*O que o atendente consegue fazer quando o query endpoint funciona corretamente. Orientados ao resultado do usuário — não a features técnicas.*

---

### OUT-01 — Responder perguntas operacionais sem sair do fluxo de atendimento

O atendente consegue obter a resposta para perguntas sobre frete, devolução, prazos e SLAs enquanto está em uma interação com o cliente, sem precisar abrir o SharePoint, consultar colegas ou interromper o atendimento. A resposta deve ser percebida como rápida o suficiente para não quebrar o ritmo da conversa com o cliente.

**Por que importa:** O discovery identificou que o tempo de resposta aceitável é menor que 30 segundos. Qualquer overhead que force o atendente a validar manualmente a informação em outro sistema anula o ganho do assistente.

**Nota:** O número exato de 30 segundos e a forma como a resposta é entregue (bloco único ou incremental) são parâmetros de produto — mas a métrica técnica que operacionaliza esse outcome (que ponto medir, se streaming é necessário, se os números são alcançáveis dada a infraestrutura) é decisão de engenharia. Ver OQ-02.

---

### OUT-02 — Tomar decisões de atendimento com confiança na fonte

O atendente sabe, para cada informação recebida, de qual documento ela veio — e pode julgar se precisa confirmar ou escalar. Não precisa confiar cegamente no assistente nem verificar tudo por padrão.

**Por que importa:** O assistente atua no fluxo de atendimento ao cliente. Uma resposta errada não citada é mais perigosa do que uma resposta incompleta citada, porque o atendente não tem sinal para questionar.

**Nota:** A capacidade do atendente de julgar quando escalar depende da qualidade das citações e da clareza da linguagem de roteamento. O julgamento humano em si não é testável por critério automatizado nesta versão — deve ser avaliado em sessão de usabilidade com atendentes reais antes do go-live.

---

### OUT-03 — Navegar conflitos documentais sem precisar resolver o conflito sozinho

Quando existem versões contraditórias de uma política (ex: PROC-042 v1 vs. v2), o atendente recebe as duas versões e a indicação de qual aplicar ao caso em questão, sem precisar descobrir isso por conta própria.

**Por que importa:** O time de atendimento já conhece o problema das duas versões do PROC-042 (FAQ item 8). O assistente precisa tornar o conflito navegável, não escondê-lo nem transferir o ônus da resolução para o atendente.

**Nota:** A regra de negócio (apresentar ambas as versões, indicar qual é a recomendada) é definida pela disposição transitória do PROC-042-v2 e está dentro da competência do Product Specialist registrar. O mecanismo técnico para identificar qual versão aplicar — se depende de um parâmetro de API, de que forma é validado, o que fazer em entradas inválidas — é decisão de contrato de API. Ver OQ-04.

---

### OUT-04 — Saber quando o assistente não tem a resposta

O atendente recebe uma sinalização clara quando a pergunta ultrapassa o escopo da base documental disponível — e é direcionado para o canal correto (ramal 4500, Comercial, sinistros@novatech.com.br) em vez de receber uma resposta fabricada.

**Por que importa:** Existem gaps documentais conhecidos (tabela base de frete padrão, carga danificada em trânsito, PROC-043) onde uma resposta genérica do modelo seria plausível mas incorreta. O custo de uma resposta errada confiante é maior que o custo de uma resposta honesta sobre o limite do sistema.

---

### OUT-05 — Responder perguntas que cruzam mais de uma área sem fragmentação

O atendente recebe uma resposta unificada para perguntas que envolvem mais de um domínio (ex: "quanto custa devolver uma carga especial para o Nordeste?" — cruza Frete e Devolução), sem precisar fazer duas perguntas separadas e compor manualmente.

**Por que importa:** 15% das perguntas do discovery cruzam duas categorias. Nesses casos, uma resposta que responde apenas metade da pergunta força o atendente a uma segunda interação — dobrando o tempo e o risco de inconsistência entre as respostas.

---

## 2. Scope Boundaries

*O que este componente cobre e o que está explicitamente fora. Derivado dos bounded contexts definidos em bounded-contexts-assistente-novatech.md.*

---

### Dentro do escopo

**BC-01 — Cálculo e Regras de Frete**
Regras e fórmula do frete especial (acima de 500 kg): fator de peso, multiplicadores regionais, prazo adicional, regras de desconto por volume, limite de autonomia do atendente. Inclui a apresentação do conflito entre PROC-042 v1 e v2 com indicação da versão aplicável.

**BC-02 — Devolução de Mercadorias**
Elegibilidade, prazo (7 dias úteis), procedimento passo a passo, devoluções parciais, responsabilidade pelo custo do frete reverso, categorias inelegíveis para o processo padrão e roteamento para Gestão de Riscos.

**BC-03 — Prazos de Entrega**
Prazo adicional do frete especial, referências práticas de prazo por região (Norte, Sul/Sudeste), identificação de situações que justificam abertura de chamado de rastreamento com prioridade alta. Não inclui consulta em tempo real ao sistema de tracking.

**BC-04 — SLA e Atendimento ao Cliente**
Classificação de tiers (Gold, Silver, Standard), tabela de SLAs por tier, definição de incidente crítico, penalidades por descumprimento, regras de pausa do relógio de SLA.

**Perguntas cross-context**
Perguntas que cruzam dois bounded contexts (ex: Frete + Devolução, Prazos + SLA). O endpoint deve compor a resposta a partir dos dois contextos relevantes em uma única resposta.

---

### Fora do escopo

| O que está fora | Por quê | O que fazer |
|----------------|---------|-------------|
| Frete padrão (abaixo de 500 kg) | Tabela base ausente da base documental | Informar o gap e encaminhar ao Comercial |
| Frete de cargas perigosas (PROC-043) | Documento não disponível na base | Informar e encaminhar ao ramal 4500 |
| Carga danificada em trânsito | Sem documento formal — apenas FAQ informal | Citar FAQ com ressalva e encaminhar a sinistros@novatech.com.br |
| Interceptação de carga em trânsito | PROC-088 não disponível | Informar o gap |
| Consulta em tempo real ao tracking | Dado operacional, não documental | Fora da competência do assistente — encaminhar ao sistema de tracking |
| Aprovações operacionais (ex: carga >5.000 kg) | Ação, não informação | Informar a necessidade de aprovação e o responsável |
| Negociação de SLA fora dos três tiers | Processo do Comercial | Encaminhar ao Comercial |
| Seguro de carga | Sem documento formal — apenas FAQ informal | Citar FAQ com ressalva e encaminhar ao Comercial |
| Pipeline de ingestão | Componente separado | Fora deste requirements.md |
| Interface Teams / painel web | Componentes separados | Fora deste requirements.md |
| Verificação de chunking de tabelas | Responsabilidade do pipeline de ingestão (ADR-0004), não do endpoint | Critério movido para requirements do pipeline de ingestão — ver nota em VC-12 |

---

## 3. Constraints de Negócio

*Restrições não negociáveis derivadas de política documentada da NovaTech ou de decisão de produto já validada. Esta seção é da competência do Product Specialist.*

---

**CON-N01 — O assistente nunca fabrica informações**
Toda afirmação factual na resposta deve ser suportada por um chunk recuperado da base documental. Se a informação não está na base, o assistente declara o limite explicitamente. Não é permitido que o modelo complete lacunas com conhecimento geral ou inferência — especialmente em valores monetários, prazos e regras de elegibilidade.

Uma **afirmação factual** é qualquer um dos seguintes elementos presentes na resposta: valor numérico (prazo, percentual, valor monetário, peso, contagem); regra de elegibilidade ou exceção; nome de processo, canal ou responsável; classificação ou categorização. Frases de estrutura e transição não são afirmações factuais e não precisam de citação.

**CON-N02 — Toda resposta cita a fonte**
Cada afirmação factual deve ser acompanhada da identificação do documento de origem (código do documento, versão, seção quando disponível). Respostas sem citação não são aceitas, mesmo que corretas.

**CON-N03 — Atualização da base em até 24 horas**
Uma alteração na documentação normativa deve estar disponível nas respostas do endpoint em até 24 horas após a ingestão. Este é um requisito de produto (vindo da spec de RAG da fase anterior); o mecanismo técnico de como o endpoint garante isso — se há cache e como ele respeita esse limite — é decisão de engenharia. Ver OQ-05.

**CON-N04 — Conflitos documentais são apresentados, não escondidos**
Quando a base contém versões contraditórias de um documento, o endpoint deve apresentar as duas versões ao atendente com a indicação de qual aplicar. É proibido apresentar apenas uma versão sem mencionar a existência da outra.

Para o caso específico do PROC-042 v1 vs. v2: a disposição transitória da v2 estabelece que chamados abertos antes de 01/12/2023 ainda em processamento seguem a v1; chamados a partir dessa data seguem a v2. O endpoint deve refletir essa regra na resposta. O mecanismo de como o endpoint identifica a qual período um chamado pertence é decisão de contrato de API — ver OQ-04.

**CON-N05 — Gaps documentais são declarados com roteamento**
Quando a pergunta cobre um tópico sem documento formal na base (frete padrão, carga danificada, seguro), o endpoint declara o gap e fornece o canal de roteamento correto. Não é aceitável silêncio, recusa genérica ("não sei responder") ou resposta baseada no FAQ informal sem ressalva explícita.

**CON-N06 — Linguagem ubíqua obrigatória**
O endpoint usa os termos definidos em linguagem-ubiqua-novatech.md de forma consistente. Em particular: nunca aceitar "tier Platinum" sem corrigir; usar "dias úteis" com a definição NovaTech (exclui feriados nacionais); distinguir "SLA de resposta" de "SLA de resolução"; usar "incidente crítico" apenas com os 4 critérios taxativos do SLA-2024.

---

## 4. Constraints Técnicas Já Validadas

*Restrições técnicas que já foram objeto de ADR formal, aprovadas pelo Tech Lead antes da fase atual. Diferem das Open Questions da Seção 6 porque já têm decisão e dono registrados — não são reabertas aqui.*

---

**CON-T01 — Modelo fixo: GPT-4o via Azure OpenAI (ADR-0001)**
O endpoint usa exclusivamente o GPT-4o disponível via Azure OpenAI. Não é permitido usar outro modelo ou provedor sem nova ADR. A janela de contexto de 128K tokens é o teto absoluto.

**CON-T03 — Pipeline de RAG: Azure AI Search + Azure OpenAI (ADR-0004)**
O endpoint usa Azure AI Search para recuperação dos chunks. O protótipo identificou problemas de chunking em tabelas — a estratégia de chunking (responsabilidade do pipeline de ingestão) deve tratar tabelas como unidades indivisíveis.

**CON-T04 — Documentos contraditórios: metadado de vigência + instrução no prompt (ADR-0003)**
Documentos com contradições são mantidos na base com metadado de vigência. O prompt instrui o modelo a priorizar a versão mais recente e a sinalizar o conflito na resposta. Documentos obsoletos são marcados, não excluídos.

**CON-T06 — Stack: TypeScript + Azure Functions**
O endpoint é implementado em TypeScript, rodando em Azure Functions. Sem exceções para este componente.

---

## 5. Prior Decisions

*Decisões já tomadas que o query endpoint não pode reverter.*

---

| ID | Decisão | Implicação para o endpoint | Onde revisar |
|----|---------|---------------------------|-------------|
| ADR-0001 | Modelo: GPT-4o via Azure OpenAI | O endpoint não pode chamar outro modelo ou provedor | ADR-0001 |
| ADR-0003 | Documentos contraditórios: manter ambos com metadado de vigência + instrução no prompt | O endpoint apresenta conflitos — não os resolve nem esconde | ADR-0003 |
| ADR-0004 | Pipeline RAG: Azure AI Search + Azure OpenAI; tabelas como unidades indivisíveis no chunking | O endpoint consome chunks já indexados segundo essa estratégia | ADR-0004 |
| BC-01 a BC-04 | Bounded contexts definidos: Frete, Devolução, Prazos, SLA e Atendimento | O escopo do endpoint é a cobertura desses 4 contextos | bounded-contexts-assistente-novatech.md |
| Linguagem ubíqua | Glossário de 20+ termos com definição NovaTech | O system prompt incorpora as definições críticas do Grupo 1 do glossário | linguagem-ubiqua-novatech.md |
| Base documental | 847 documentos válidos; 12 com contradições pendentes; FAQ marcado como informal | O endpoint trata FAQ como fonte secundária — nunca primária quando há documento normativo | Consolidação do discovery |

**Nota:** ADR-0002 (context budget) não aparece mais nesta tabela como decisão fechada. Os valores numéricos exatos de orçamento de tokens foram revertidos para Open Question (OQ-01) porque a ADR original usava valores aproximados que não sobrevivem à implementação sem refinamento técnico que ainda não foi feito pelo Tech Lead.

---

## 6. Open Questions

*Decisões que apareceram como ambiguidades bloqueadoras nas revisões técnicas anteriores, mas que não são da competência do Product Specialist para resolver. Cada item tem dono explícito. Nenhuma constraint técnica derivada de um destes itens deve ser tratada como definitiva até a resolução ser registrada aqui com data e responsável.*

---

### OQ-01 — Limites exatos de token do context budget

**Origem:** AMB-01 (revisão 1), NOVO-01 (revisão 2)

**O que precisa ser decidido:**
- Hard limit e soft target exatos para system prompt e chunks recuperados
- Hard limit de tokens para o histórico de conversa (não apenas limite de turnos) e política de truncamento quando excedido (FIFO, truncar turno mais longo, ou rejeitar)
- Comportamento de erro quando um hard limit é excedido (código de erro, se trunca ou rejeita)

**Por que não é competência do Product Specialist:** Esses valores dependem de engenharia de prompt (quanto o system prompt realmente precisa para cobrir a linguagem ubíqua e as instruções de citação) e de testes de qualidade de resposta em diferentes orçamentos — trabalho de prototipagem técnica, não de definição de produto.

**Dono:** Tech Lead, com validação do time de desenvolvimento via spike técnico.

**Status:** Pendente.

---

### OQ-02 — Arquitetura de entrega da resposta e métricas de latência

**Origem:** AMB-02 (revisão 1), NOVO-02 (revisão 2)

**O que precisa ser decidido:**
- Se o endpoint usa streaming (SSE) ou resposta síncrona
- Se streaming: definição precisa do ponto de medição de "primeiro token" (qual camada do pipeline Azure Functions)
- Valores de limite de latência que sejam viáveis dado o desempenho real do Azure AI Search em Brazil South — requer benchmark, não estimativa
- O que fazer quando o limite de latência é estruturalmente inalcançável em parte do pipeline (ex: se o AI Search sozinho já consome a maior parte do orçamento de tempo)

**Por que não é competência do Product Specialist:** A escolha entre streaming e síncrono é decisão de arquitetura com impacto em todos os consumidores do endpoint (bot do Teams, painel web). Os limites de latência exigem benchmark real de infraestrutura Azure, não uma estimativa de produto.

**Dono:** Tech Lead, em conjunto com quem detém a infraestrutura Azure (DevOps ou arquiteto de soluções) para validar os números com benchmark real antes de registrá-los como constraint.

**Status:** Pendente. Outcome de produto associado (OUT-01: resposta percebida como rápida, dentro de ~30s) permanece válido — apenas a métrica técnica que o operacionaliza está pendente.

---

### OQ-03 — Contrato de API: schema de retorno de chunks recuperados

**Origem:** QA-03 (revisão 1), NOVO-03 (revisão 2)

**O que precisa ser decidido:**
- Schema tipado completo do payload de resposta, incluindo quais campos são obrigatórios e quais são opcionais (em particular: `secao`, que não está disponível para todos os documentos)
- Onde e como o campo de declaração de gap documental aparece no schema
- Formato de versionamento do próprio contrato de API (para evolução futura sem quebrar consumidores)

**Por que não é competência do Product Specialist:** Definir um contrato de API com tipagem completa é decisão de design de API, normalmente feita pelo Tech Lead em conjunto com quem vai consumir o contrato (devs do bot do Teams e do painel web).

**Dono:** Tech Lead, com input dos desenvolvedores responsáveis pela interface Teams e pelo painel web (consumidores do contrato).

**Status:** Pendente. O requisito de produto — "o endpoint expõe a fonte de cada informação que retorna" — permanece como CON-N02; a forma técnica exata de expor isso é o que está pendente aqui.

---

### OQ-04 — Mecanismo de identificação de versão aplicável (PROC-042 v1/v2)

**Origem:** AMB-04 (revisão 1), NOVO-04 e NOVO-05 (revisão 2)

**O que precisa ser decidido:**
- Se a informação de qual versão aplicar depende de um parâmetro explícito no request (ex: data de abertura do chamado) ou se o endpoint sempre aplica a versão mais recente com ressalva
- Se houver parâmetro: formato exato, validação de entradas inválidas, código de erro retornado
- Como tratar a divergência entre a regra do PROC-042-v2 (que depende do status "em processamento", não disponível ao endpoint) e a regra que o endpoint de fato consegue implementar (apenas por data)

**Por que não é competência do Product Specialist:** Decidir se um parâmetro de data deve fazer parte do contrato de API, e como validá-lo, é decisão técnica. Decidir como tratar a divergência entre a regra normativa completa e o que é tecnicamente verificável também depende de saber o que o sistema de chamados (fora do escopo deste endpoint) consegue fornecer como input — informação que o Product Specialist não tem.

**Dono:** Tech Lead, com consulta à Compliance para validar se a simplificação (aplicar a regra apenas por data, sem verificar status do chamado) é aceitável como divergência registrada da norma original.

**Status:** Pendente. O requisito de negócio — apresentar ambas as versões e indicar qual é a recomendada (CON-N04) — permanece válido independentemente de como o parâmetro é implementado.

---

### OQ-05 — Estratégia de cache

**Origem:** MEN-01 (revisão 1), REM-01 (revisão 2) — aberto em ambas as rodadas sem resolução

**O que precisa ser decidido:**
- Se o endpoint implementa cache em algum nível (resposta completa, chunks, embeddings de query)
- Se sim: TTL exato, estratégia de invalidação, e como isso se relaciona com o requisito de atualização da base em 24h (CON-N03)
- Se não: registrar explicitamente "sem cache na v1" para remover a ambiguidade do texto atual de CON-N03

**Por que não é competência do Product Specialist:** Decisão de cache é trade-off de performance vs. atualidade de dados — depende de medição de carga esperada e de custo de chamadas repetidas ao Azure AI Search/OpenAI, informação que pertence à engenharia.

**Dono:** Tech Lead.

**Status:** Pendente — este item já foi sinalizado em duas rodadas de revisão sem decisão. Recomenda-se priorizá-lo para não chegar a uma terceira rodada com o mesmo apontamento.

---

## 7. Verification Criteria

*Critérios testáveis pelo QA. Critérios marcados como **bloqueados** dependem da resolução de uma Open Question da Seção 6 e não devem ser executados com valores definitivos até então — qualquer execução prévia deve ser tratada como exploratória, não como critério de aceite.*

---

### Critérios derivados dos Outcomes

**VC-01 — Latência (→ OUT-01) — 🔒 BLOQUEADO por OQ-02**

- **O que testa:** Se o atendente recebe a resposta dentro do tempo operacional aceitável
- **Pendência:** A métrica exata (síncrona vs. TTFT/TTC), o ponto de medição e os valores de limite dependem da resolução de OQ-02. Não executar como critério de aceite até a Open Question ser resolvida
- **Como executar (provisório, sujeito a revisão):** Conjunto fixado de perguntas de referência cobrindo os 4 bounded contexts e casos cross-context, executado sob condições de carga definidas
- **Aprovação:** A ser definida após OQ-02

---

**VC-02 — Citação de fonte em toda resposta factual (→ OUT-02)**

- **O que testa:** Se o atendente sempre sabe de onde veio a informação
- **Como executar:** Analisar 30 respostas geradas (10 por bounded context coberto, ponderando os mais críticos). Para cada afirmação factual identificada na resposta (conforme definição em CON-N01), verificar se há citação de documento (código + versão)
- **Aprovação:** 100% das afirmações factuais têm citação. Zero respostas com afirmação não citada. Citações parciais (só código sem versão em documentos com múltiplas versões) contam como falha
- **Método de avaliação:** Inspeção manual em par (QA + domain expert) para o conjunto inicial de release. Para regressão contínua em CI, usar subconjunto com ground truth fixado

---

**VC-03 — Apresentação de conflito documental (→ OUT-03) — 🔒 PARCIALMENTE BLOQUEADO por OQ-04**

- **O que testa:** Se perguntas sobre frete especial expõem o conflito entre PROC-042 v1 e v2
- **Como executar:** Enviar ao endpoint as seguintes perguntas: (a) "Qual o multiplicador regional para o Nordeste?"; (b) "Qual o fator de peso para uma carga de 2.000 kg?"; (c) "Qual o prazo adicional para frete especial?"
- **Aprovação (parte não bloqueada):** Todas as três respostas devem apresentar os valores das duas versões e identificar a v2 como a mais recente. Falha se apresentar apenas uma versão, mesmo que correta
- **Pendência:** As variantes de teste que dependem de informar a versão por data de chamado (ex: testar com um chamado anterior a 01/12/2023) dependem do mecanismo de OQ-04 estar definido. Não executar essas variantes até então

---

**VC-04 — Declaração de gap documental com roteamento (→ OUT-04) — 🔒 PARCIALMENTE BLOQUEADO por OQ-03**

- **O que testa:** Se o endpoint declara os limites corretos em vez de fabricar resposta
- **Como executar:** Enviar as seguintes perguntas: (a) "Qual o prazo de entrega para o Pará?"; (b) "Como funciona o frete para cargas perigosas acima de 500 kg?"; (c) "Minha carga chegou danificada, o que faço?"; (d) "Qual o valor do seguro de carga?"
- **Aprovação (parte não bloqueada):** Cada resposta deve: (1) declarar explicitamente que a informação não está disponível na base documental; (2) fornecer o canal de roteamento correto; (3) não conter nenhuma afirmação factual inventada sobre o tópico. Para (c) e (d), pode citar o FAQ com ressalva explícita de fonte informal
- **Pendência:** A verificação cruzada com os chunks efetivamente recuperados (para confirmar ausência de fabricação com evidência, não apenas inspeção da resposta) depende do schema de OQ-03 estar definido

---

**VC-05 — Resposta a perguntas cross-context (→ OUT-05)**

- **O que testa:** Se perguntas que cruzam dois bounded contexts recebem resposta unificada
- **Como executar:** Enviar: (a) "Quanto custa devolver uma carga de 2.000 kg para o Nordeste?" (Frete + Devolução); (b) "Uma carga Gold parada há 7 dias no Norte já é incidente crítico?" (Prazos + SLA)
- **Aprovação:** Cada resposta deve cobrir os elementos mínimos de cada contexto:

| Pergunta | Elementos mínimos contexto A | Elementos mínimos contexto B |
|----------|------------------------------|------------------------------|
| (a) | **Frete:** multiplicador regional do Nordeste + fator de peso para 2.000 kg | **Devolução:** custo do frete reverso é do cliente em caso de desistência, calculado com os mesmos multiplicadores |
| (b) | **Prazos:** rotas para o Norte podem levar até 10 dias úteis — não é necessariamente anomalia | **SLA:** incidente crítico requer valor declarado > R$100k com status desconhecido > 6h |

Falha se responder apenas um dos contextos, sem os elementos mínimos, ou exigir pergunta de follow-up para o segundo contexto.

---

### Critérios derivados das Constraints de Negócio

**VC-06 — Proibição de fabricação de informação (→ CON-N01) — 🔒 PARCIALMENTE BLOQUEADO por OQ-03**

- **O que testa:** Se o modelo inventa informações quando a base não cobre o tópico
- **Como executar:** Enviar 10 perguntas sobre tópicos deliberadamente ausentes da base
- **Aprovação:** Zero afirmações factuais sem suporte documental identificável. Toda resposta a pergunta fora do escopo deve ser declaração de limite, não estimativa ou inferência
- **Pendência:** A verificação automatizada cruzando afirmações da resposta com os chunks efetivamente recuperados depende do schema de OQ-03. Sem isso, a verificação é manual e mais sujeita a erro
- **Nota adicional:** Quando o schema estiver definido, este critério deve incluir também a verificação inversa: para cada chunk recuperado cujo conteúdo aparece na resposta, confirmar que há citação correspondente (caso contrário, é falha de VC-02, não de fabricação)

---

**VC-07 — Context budget dentro dos limites (→ OQ-01) — 🔒 BLOQUEADO por OQ-01**

- **O que testa:** Se o endpoint respeita os limites de tokens do context budget
- **Pendência:** Os valores exatos de hard limit e soft target, e a política de truncamento do histórico, dependem da resolução de OQ-01. Não executar como critério de aceite até então
- **Nota de design do teste (para quando OQ-01 for resolvido):** Separar em dois critérios — um de inspeção estática para componentes fixos (ex: tamanho do system prompt compilado, verificável em code review) e um de teste em runtime para componentes variáveis (histórico e chunks, que variam por conversa e por query)

---

**VC-08 — Recusa do tier Platinum (→ CON-N06)**

- **O que testa:** Se o endpoint usa a linguagem ubíqua e corrige termos inexistentes
- **Como executar:** Enviar: "Meu cliente é Platinum. Qual o SLA dele?" e "Qual o SLA de um cliente VIP?"
- **Aprovação:** Ambas as respostas devem: (1) informar que o tier mencionado não existe na NovaTech; (2) listar os três tiers válidos (Gold, Silver, Standard); (3) conter texto orientando o atendente a verificar o número do contrato para identificar o tier correto. Zero respostas que aceitem Platinum ou VIP como tier válido
- **Nota:** O item (3) testa apenas o conteúdo textual da resposta do endpoint — a ação de efetivamente solicitar e validar o número do contrato é responsabilidade da interface (Teams/painel web), fora do escopo deste VC

---

**VC-09 — Distinção entre SLA de resposta e SLA de resolução (→ CON-N06)**

- **O que testa:** Se o endpoint distingue os dois SLAs e não os confunde com o prazo de triagem de devolução
- **Como executar:** Enviar: (a) "Qual o SLA do cliente Gold?"; (b) "Em quanto tempo o time responde um chamado de devolução?"
- **Aprovação:** (a) deve apresentar os dois SLAs separadamente (resposta: 2h úteis; resolução: 24h úteis para chamados gerais) e os SLAs de incidente crítico (30 min / 4h). (b) deve distinguir o prazo de triagem interna (4h úteis, POL-001) do SLA contratual de primeira resposta do tier do cliente

---

**VC-10 — Não-pausa do relógio de SLA para Gold em incidente crítico (→ CON-N06)**

- **O que testa:** Se o endpoint apresenta corretamente a regra de pausa/não-pausa do relógio de SLA
- **Como executar:** Enviar: "Um incidente crítico de um cliente Gold foi aberto às 17h. O SLA de resolução conta até quando?"
- **Aprovação:** A resposta deve indicar que o relógio não pausa para incidentes críticos de Gold. Falha se indicar que o relógio pausa às 18h para esse caso

---

**VC-11 — Definição de incidente crítico com os 4 critérios (→ CON-N06)**

- **O que testa:** Se o endpoint usa a definição taxativa de incidente crítico
- **Como executar:** Enviar: (a) "Uma carga de R$ 80.000 está parada há 8 horas. É incidente crítico?"; (b) "Recebi 6 chamados do mesmo cliente hoje sobre o mesmo problema. É incidente crítico?"
- **Aprovação:** (a) deve responder que não — valor abaixo do critério de R$100k. (b) deve responder que sim — mais de 5 chamados nas últimas 24h é critério suficiente. Falha se classificar (a) como crítico ou (b) como não-crítico

---

### Critério movido para outro documento

**VC-12 — Tabelas indexadas como unidades indivisíveis**

Este critério testa o comportamento do pipeline de ingestão (estratégia de chunking definida na ADR-0004), não o query endpoint — o endpoint apenas consome o que o Azure AI Search já indexou. Removido deste documento. Deve constar no requirements.md do pipeline de ingestão.

---

## Resumo de rastreabilidade

| Outcome | Constraints relacionadas | Verification Criteria | Status |
|---------|--------------------------|----------------------|--------|
| OUT-01 Responder sem sair do fluxo | OQ-02 (pendente) | VC-01 | 🔒 Bloqueado |
| OUT-02 Confiar na fonte | CON-N01, CON-N02, OQ-03 (pendente) | VC-02, VC-06 | VC-02 ok · VC-06 parcial |
| OUT-03 Navegar conflitos | CON-N04, OQ-04 (pendente) | VC-03 | Parcial |
| OUT-04 Saber o limite do assistente | CON-N01, CON-N05, OQ-03 (pendente) | VC-04, VC-06 | Parcial |
| OUT-05 Perguntas cross-context | — | VC-05 | Ok |
| — | CON-N06 (linguagem ubíqua) | VC-08, VC-09, VC-10, VC-11 | Ok |
| — | OQ-01 (pendente) | VC-07 | 🔒 Bloqueado |
| — | CON-T03 (tabelas inteiras) | VC-12 movido para pipeline de ingestão | Movido |

**Itens com status "Ok" ou "Parcial" podem seguir para desenvolvimento e QA já agora.** Itens marcados "🔒 Bloqueado" e as partes pendentes dos itens "Parcial" aguardam resolução das Open Questions correspondentes (Seção 6) pelos donos indicados antes de virarem critério de aceite definitivo.
