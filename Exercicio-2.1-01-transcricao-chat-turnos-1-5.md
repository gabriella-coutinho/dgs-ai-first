# Transcrição do Chat — Turnos 1 a 5
## Projeto: Assistente de IA NovaTech

**Data:** Junho/2026
**Participantes:** Product Specialist (usuário) · Claude (assistente)
**Escopo desta transcrição:** Turnos 1–5 — recorte de domínio, linguagem ubíqua e requirements.md (versão inicial)

---

## Contexto do projeto (fixo em todos os turnos)

**Cenário:** O projeto foi aprovado, o discovery está concluído. O time está na fase de estruturação do ambiente, padrões e artefatos de governança do desenvolvimento.

**Decisões da fase anterior:**
- Modelo LLM: Azure OpenAI (GPT-4o) — ADR-0001
- Pipeline de RAG: Azure AI Search + Azure OpenAI — ADR-0004
- Context budget: ~4K system prompt + ~8K chunks + 3 turnos de histórico — ADR-0002
- Documentos contraditórios: metadado de vigência no pipeline; prompt prioriza versão mais recente — ADR-0003
- Integração: Microsoft Teams + painel web interno
- Base documental: 847 documentos válidos (12 com contradições pendentes), 63 descartados, ~340 eliminados como duplicatas
- Stack: TypeScript (backend e bot), React (painel web), Bicep para infraestrutura
- Time: 1 Tech Lead, 2 Devs, 1 QA, 1 Product Specialist, 1 Delivery Manager

**Inputs de produto:**
- Spec de RAG: "O assistente responde perguntas sobre SLAs, frete e devoluções. Fontes contraditórias devem mostrar ambas as versões. O assistente nunca inventa informações. Toda resposta cita fonte. Atualização em até 24h."
- Discovery: perguntas mais frequentes em 4 categorias — prazos de entrega, regras de frete, política de devolução e SLAs. Em 15% dos casos, a pergunta cruza duas categorias. Os atendentes precisam da resposta em menos de 30 segundos.

**Documentos de referência disponíveis (Anexo A):**
- POL-001 v3.1 — Política de Devolução de Mercadorias
- PROC-042 v1.0 — Procedimento de Cálculo de Frete Especial
- PROC-042 v2.0 — Procedimento de Cálculo de Frete Especial (Revisado)
- SLA-2024 v2024.1 — Tabela de SLA por Tipo de Cliente
- FAQ-Atendimento — Perguntas Frequentes do Time de Suporte (informal, não validado por Compliance)

---

## Turno 1

### Prompt

> Você é um Product Specialist analisando a documentação da NovaTech para um projeto de assistente de IA para a NovaTech.
>
> [TAREFA]
> Com base no cenário, nos inputs e na documentação da NovaTech em anexo, faça o recorte de domínio do projeto, começando por:
>
> 1. Identifique os bounded contexts do assistente NovaTech. Para cada contexto, defina: o que está dentro, o que está fora, e como ele se relaciona com os outros. Os contextos devem refletir o domínio de negócio de logística — não divisões técnicas como frontend ou backend.

### Resposta

**Premissa metodológica:** Os bounded contexts foram derivados cruzando três fontes — as categorias de perguntas frequentes do discovery, os documentos normativos do Anexo A, e as fronteiras naturais de responsabilidade de negócio que emergem dos próprios documentos (cada vez que um documento diz "consulte o setor X" ou "encaminhe para Y", está revelando onde um contexto termina e outro começa).

---

**BC-01 — Cálculo e Regras de Frete**

*Dentro:* Fórmula do frete especial (`Valor base × Multiplicador regional × Fator de peso`), faixas de peso, multiplicadores regionais por destino, prazo adicional (+2 dias v1 / +3 dias v2), regras de desconto por volume, limite de autonomia do atendente (zero), conflito PROC-042 v1 vs. v2 — o assistente deve surfá-lo explicitamente.

*Fora:* Tabela de fretes base (ausente da base documental), PROC-043 (cargas perigosas acima de 500 kg), aprovação para cargas acima de 5.000 kg, negociação de aditivo contratual.

*Relações:* Fornece prazo adicional para BC-03; compartilha regra de multiplicadores com BC-02 (frete reverso); roteia cargas perigosas para Gestão de Riscos (fora do escopo).

---

**BC-02 — Devolução de Mercadorias**

*Dentro:* Prazo de 7 dias úteis, categorias inelegíveis (cargas perigosas classes 1–6 ANTT, refrigeradas com ruptura >30 min por sensor IoT, lacre violado sem documentação), procedimento completo (Portal do Cliente → CT-e + fotos → triagem 4h úteis → coleta reversa 2 dias úteis → reembolso 5 dias úteis), devoluções parciais, responsabilidade pelo custo, roteamento para ramal 4500.

*Fora:* Carga danificada em trânsito (sem documento formal — apenas FAQ item 38), interceptação de carga em trânsito (PROC-088 indisponível), o que acontece dentro da Gestão de Riscos.

*Relações:* Consome multiplicadores de BC-01 para frete reverso; coexiste com SLAs de BC-04 (triagem interna ≠ SLA contratual).

---

**BC-03 — Prazos de Entrega**

*Dentro:* Prazo adicional do frete especial (+2 ou +3 dias conforme versão do PROC aplicável), referências práticas do FAQ (Norte até 10 dias úteis; Sul/Sudeste mais de 3 dias parado é anomalia), definição do que justifica chamado de rastreamento com prioridade alta.

*Fora:* Rastreamento operacional em tempo real, motivos de atraso, roteirização. A tabela base de prazos por rota não está disponível — este é o contexto com maior incompletude documental.

*Relações:* Consome prazo adicional de BC-01; fornece gatilho de incidente crítico para BC-04.

---

**BC-04 — SLA e Atendimento ao Cliente**

*Dentro:* Tiers Gold/Silver/Standard (critérios e confirmação de que Platinum não existe), tabela completa de SLAs, definição taxativa de incidente crítico (4 critérios), penalidades por descumprimento, regra de pausa/não-pausa do relógio de SLA.

*Fora:* Negociação de SLA fora dos três tiers, histórico de violações de um cliente específico, processo interno no Azure DevOps.

*Relações:* Distingue SLA de triagem (POL-001, interno) de SLA contratual; consome prazo de entrega de BC-03 como gatilho de incidente crítico.

---

**Mapa de relações:**

```
┌─────────────────────┐         ┌─────────────────────┐
│   BC-01 FRETE       │────────▶│   BC-03 PRAZOS      │
└─────────────────────┘         └──────────┬──────────┘
         │ custo frete reverso             │ gatilho incidente crítico
         ▼                                 ▼
┌─────────────────────┐         ┌─────────────────────┐
│   BC-02 DEVOLUÇÃO   │────────▶│   BC-04 SLA         │
└─────────────────────┘         └─────────────────────┘
         └────────────┬────────────────────┘
                      ▼
         ┌────────────────────────┐
         │  GESTÃO DE RISCOS      │
         │  (fora do escopo)      │
         └────────────────────────┘
```

**Decisões de design que emergem do recorte:**

1. Perguntas cross-context (15% do discovery) exigem composição de chunks de dois BCs em uma única resposta — não podem ser truncadas no primeiro contexto encontrado.
2. BC-01 carrega o conflito documental mais crítico (PROC-042 v1/v2 com impacto monetário direto) — é obrigação de design do prompt, não escolha do modelo.
3. BC-03 é o contexto com maior incompletude — respostas parciais explícitas são preferíveis a respostas fabricadas.
4. FAQ não é fonte primária em nenhum contexto — precisa de metadado de tipo de fonte no pipeline de ingestão.
5. Gestão de Riscos é contexto de saída, não de conteúdo — o assistente roteia, não decide.

---

## Turno 2

### Prompt

> Gere um arquivo em markdown para download da definição de bounded contexts para assistente de IA antes de partir para a próxima tarefa.

### Resposta

Arquivo gerado: **`bounded-contexts-assistente-novatech.md`**

O documento contém:
- Premissa metodológica (3 fontes de derivação dos contextos)
- Definição completa dos 4 bounded contexts (BC-01 a BC-04), cada um com: o que está dentro, o que está fora, tabela de relações com outros contextos, e alertas de conflito documental ou gap documental quando aplicável
- Tabela de SLAs completa dentro de BC-04
- Mapa de relações ASCII
- 5 decisões de design que emergem do recorte
- Tabela de gaps e riscos identificados (G-01 a G-05) com impacto e recomendação

---

## Turno 3

### Prompt

> Tendo conhecimento do cenário do projeto, o que foi definido na fase anterior, os inputs, as definições de referência e o recorte de domínio e bounded context, realize a tarefa 2:
>
> - Extraia da documentação da NovaTech a linguagem ubíqua do domínio: termos que precisam ser usados de forma consistente por humanos e agentes. Priorize termos que um LLM poderia interpretar de forma errada ou genérica sem uma definição explícita — por exemplo, termos com significado contratual, técnico ou específico da NovaTech que diferem do uso comum.

### Resposta

O critério de seleção dos termos não foi "jargão de logística" — foi especificamente *onde um LLM erraria sem a definição*. Os termos foram organizados em quatro grupos por tipo de risco.

---

**Grupo 1 — Risco crítico de interpretação** (impacto contratual ou financeiro direto se interpretados genericamente):

- **Dias úteis** — exclui feriados nacionais (não apenas fins de semana). Erros afetam todos os prazos contratuais.
- **Incidente crítico** — 4 critérios taxativos do SLA-2024 §3. O LLM tenderia a usar o conceito ITIL ou o sentido coloquial.
- **Devolução (processo padrão)** — pós-entrega apenas, com CT-e obrigatório. O LLM misturaria com devolução de e-commerce, carga em trânsito ou carga danificada.
- **Frete especial** — faixa de peso acima de 500 kg, não urgente/premium. O LLM usaria uma das duas versões do PROC-042 sem sinalizar o conflito.
- **Carga perigosa** — classes 1–6 da ANTT conforme Resolução 5.947/2021. O LLM usaria definição popular.
- **Tier de cliente** — lista fechada: Gold, Silver, Standard. Platinum não existe.
- **SLA de resposta vs. SLA de resolução** — dois SLAs independentes; o cumprimento de um não implica o outro.
- **Relógio de SLA** — pausa fora do horário comercial para chamados gerais; não pausa para incidentes críticos de Gold.

**Grupo 2 — Termos técnicos/regulatórios** (parcialmente conhecidos pelo LLM, sem precisão suficiente):

- **CT-e** — identificador primário de operação de transporte; obrigatório para abrir devolução.
- **Cadeia de frio (ruptura)** — >30 min contínuos fora da faixa, comprovado por sensor IoT.
- **Lacre de segurança violado** — exceção: se documentado com assinatura no ato da entrega, pode ser elegível.
- **Coleta reversa** — etapa dentro da devolução, não sinônimo.
- **Multiplicador regional** — dois conjuntos vigentes sem hierarquia formal (v1 e v2).
- **Fator de peso** — dois conjuntos de valores (v1: 1.0/1.2/1.5; v2: 1.0/1.15/1.4).
- **Valor base** — não está na base documental do assistente; não pode ser inventado.
- **Operação (contagem para tier)** — entrega/transporte registrado, não chamado de suporte.
- **Triagem** — prazo interno de 4h úteis (POL-001), não SLA contratual. O LLM confundiria com SLA de primeira resposta.

**Grupo 3 — Risco de escopo incorreto** (corretos em sentido geral, com fronteira diferente na NovaTech):

- **Devolução parcial** — volumes físicos de entrega multi-volume, não parte de um item.
- **Frete reverso** — calculado com os mesmos multiplicadores do frete original, não tarifa nova.
- **Chamado** — abertura formal no portal (timestamp de SLA); não inclui contato por telefone ou e-mail sem abertura.
- **Reembolso vs. crédito** — a política não especifica quando aplica cada um; o LLM afirmaria reembolso em dinheiro.
- **Prazo expirado** — não é negativa automática; vai ao Comercial para negociação caso a caso.

**Grupo 4 — Termos que não existem no domínio NovaTech** (o assistente deve reconhecer e corrigir):

- **Tier Platinum** — descontinuado em 2022.
- **Frete expresso para cargas perigosas** — mencionado no FAQ item 32, sem documento formal.
- **Frete padrão (abaixo de 500 kg)** — existe operacionalmente, mas sem regras documentadas na base.

**Termos que devem ser explicitados no system prompt** (não apenas no glossário):
1. Incidente crítico (4 critérios taxativos)
2. Dias úteis / horas úteis (regra de feriados nacionais)
3. Relógio de SLA (distinção pausa/não-pausa)
4. Frete especial (instrução de sinalizar conflito v1/v2)
5. Devolução processo padrão (limites de escopo)
6. Tier (lista fechada, nunca aceitar Platinum)
7. Triagem vs. SLA de resposta (não confundir os dois prazos)

---

## Turno 4

### Prompt

> A próxima tarefa é escrever o documento requirements.md do query endpoint seguindo o fluxo SDD.
>
> [INPUTS]
> - Spec de requisitos de RAG (simulada): "O assistente responde perguntas sobre SLAs, frete e devoluções. Fontes contraditórias devem mostrar ambas as versões. O assistente nunca inventa informações. Toda resposta cita fonte. Atualização em até 24h."
> - Dados do discovery: perguntas em 4 categorias; 15% cruzam duas categorias; atendentes precisam da resposta em menos de 30 segundos.
>
> [CRITÉRIOS]
> - Fluxo SDD: requirements.md contém: outcomes, scope boundaries, constraints, prior decisions, verification criteria.
> - Outcomes orientados a resultado do usuário, não a features técnicas.
> - Scope boundaries derivam dos bounded contexts.
> - Verification criteria são testáveis pelo QA.

### Resposta

Arquivo gerado: **`requirements.md`** (versão inicial)

O documento contém:

**1. Outcomes** (orientados ao atendente como usuário):
- OUT-01: Responder perguntas operacionais sem sair do fluxo de atendimento
- OUT-02: Tomar decisões com confiança na fonte
- OUT-03: Navegar conflitos documentais sem precisar resolver o conflito sozinho
- OUT-04: Saber quando o assistente não tem a resposta
- OUT-05: Responder perguntas cross-context sem fragmentação

**2. Scope Boundaries** — derivados dos 4 BCs, com tabela "fora do escopo" incluindo por quê e o que fazer (roteamento).

**3. Constraints técnicas:**
- CON-T01: Modelo fixo GPT-4o via Azure OpenAI (ADR-0001)
- CON-T02: Context budget — ~4K system prompt + ~8K chunks + 3 turnos
- CON-T03: Azure AI Search + tabelas como unidades indivisíveis (ADR-0004)
- CON-T04: Metadado de vigência + instrução no prompt (ADR-0003)
- CON-T05: Latência máxima 30s end-to-end
- CON-T06: Stack TypeScript + Azure Functions

**Constraints de negócio:**
- CON-N01: Nunca fabrica informações (com definição operacional de "afirmação factual")
- CON-N02: Toda resposta cita fonte
- CON-N03: Atualização da base em até 24h
- CON-N04: Conflitos documentais apresentados, não escondidos
- CON-N05: Gaps declarados com roteamento
- CON-N06: Linguagem ubíqua obrigatória

**4. Prior Decisions** — tabela com as ADRs e decisões já travadas.

**5. Verification Criteria** (VC-01 a VC-12):
- VC-01: Latência ≤ 30s
- VC-02: Citação em toda afirmação factual
- VC-03: Apresentação de conflito PROC-042 v1/v2
- VC-04: Gap documental com roteamento
- VC-05: Respostas cross-context com elementos mínimos por contexto
- VC-06: Proibição de fabricação
- VC-07: Context budget dentro dos limites
- VC-08 a VC-11: Linguagem ubíqua (Platinum, SLA resposta/resolução, relógio de SLA, incidente crítico)
- VC-12: Tabelas como unidades indivisíveis

**Nota metodológica sobre os outcomes:** A tentação era escrever outcomes técnicos. Todos os 5 estão escritos do ponto de vista do atendente — o que ele *consegue fazer*. A tabela "fora do escopo" inclui três colunas (o quê, por quê, o que fazer) porque "fora do escopo" sem roteamento vira um buraco negro operacional. O VC-11 usa deliberadamente a carga de R$80k (abaixo do threshold de R$100k) para testar se o modelo usa o valor exato ou "arredonda por segurança".

---

## Turno 5

### Prompt

> Agora aja como Tech Lead e aponte ambiguidades ou critérios que o QA não conseguiria testar.

### Resposta

Arquivo gerado: **`tech-lead-review-requirements.md`**

A revisão identificou 12 problemas em três categorias:

---

**Categoria 1 — Ambiguidades que bloqueiam implementação (4 itens):**

- **AMB-01 — CON-T02 usa "~" em valores que deveriam ser exatos.** "~4.000 tokens" não é implementável: o que acontece quando o system prompt chega em 4.200 tokens? Trunca? Falha? A sugestão é definir hard limit e soft target separados por componente.

- **AMB-02 — CON-T05 não define o ponto de início/fim da medição com streaming.** "Resposta completa" pressupõe bloco síncrono. Com streaming (padrão esperado para GPT-4o), o critério se aplica de forma diferente — e a decisão de usar streaming tem impacto na arquitetura do bot do Teams e do painel web. Sugestão: especificar se usa streaming e definir TTFT e TTC separadamente.

- **AMB-03 — "Afirmação factual" sem definição operacional.** O desenvolvedor não sabe o que instruir o modelo a citar; o QA não sabe o que inspecionar. Sugestão: definir por categorias de conteúdo (valores numéricos, regras de elegibilidade, nomes de processo/canal/responsável, classificações).

- **AMB-04 — CON-N04 não resolve o chamado pré-01/12/2023.** A disposição transitória do PROC-042-v2 usa a v1 para chamados em processamento antes dessa data — mas o endpoint não recebe a data do chamado como parâmetro. São instruções que colidem sem resolução registrada.

---

**Categoria 2 — Critérios que o QA não consegue executar (5 itens):**

- **QA-01 — VC-01: 20 perguntas não fixadas, "condições normais" indefinidas, critério 100% frágil para LLM.** Sistema estocástico com critério binário não é reproduzível. Sugestão: fixar as perguntas em arquivo separado, definir carga numericamente, usar percentis p95/p99 sobre N=100.

- **QA-02 — VC-02: sem método de avaliação definido.** Quem avalia as "afirmações factuais"? Como? Para sistemas LLM isso tipicamente requer LLM-as-judge — que tem custo e infraestrutura própria, não mencionados.

- **QA-03 — VC-04: sem retorno de chunks, impossível distinguir FAQ usado de informação inventada.** Para verificar que o modelo não inventou nada, o QA precisa inspecionar os chunks recuperados. Isso requer que o endpoint retorne os chunks junto com a resposta — requisito de implementação implícito que precisa ser explícito.

- **QA-04 — VC-05: "cobrir os dois contextos" é subjetivo.** Uma resposta pode citar dois documentos e responder apenas superficialmente um dos contextos. Sugestão: definir elementos mínimos obrigatórios por contexto para cada pergunta cross-context.

- **QA-05 — VC-12 está no documento errado.** Testa chunking (responsabilidade do pipeline de ingestão), não o endpoint. O dev do endpoint vai (corretamente) dizer que não é problema dele quando falhar.

---

**Categoria 3 — Problemas menores (3 itens):**

- **MEN-01 — CON-N03 pressupõe cache sem defini-lo.** "Nunca cachear além do TTL" pressupõe que cache existe, sem especificar nível, TTL exato ou estratégia de invalidação.

- **MEN-02 — VC-08 testa comportamento de interface.** "Solicitar o número do contrato" é ação da interface Teams/painel — não do endpoint. O critério mistura responsabilidades.

- **MEN-03 — OUT-02 sem cobertura de VC.** A parte "pode julgar quando escalar" não tem nenhum verification criteria associado. Deve ser registrado explicitamente como "não coberto nesta versão".

---

**Resumo de prioridade:**

| Bloqueia início do dev? | Itens |
|------------------------|-------|
| Sim | AMB-01, AMB-02, AMB-04, QA-03 |
| Não (tratar antes do QA) | QA-01, QA-02, QA-04, QA-05, MEN-01, MEN-02, MEN-03 |

---

*Fim dos turnos 1–5. Os turnos 6–12 cobrem: correções dos 4 bloqueadores no requirements.md (turno 6), segunda rodada de revisão do Tech Lead (turno 7), geração do documento da segunda revisão (turno 8), nova rodada de correções (turno 9), reflexão sobre decisões fora da competência do Product Specialist (turno 10), reestruturação do requirements.md com seção de Open Questions (turno 11), e solicitação desta transcrição (turno 12).*
