# Harness de Produto — Assistente de IA NovaTech (versão final)

O harness parte de um princípio simples: **o assistente vai evoluir, mas os 18 guardrails do cenário 2 (G1-G6, N1-N6, F1-F6) não podem regredir nunca**. Esta versão fecha os pontos que dependiam só do Product Specialist e marca explicitamente, como **itens de validação**, o que depende de aceite de Tech Lead e Compliance — em vez de apresentar como resolvido algo que não é decisão só minha.

---

## 1. Processo de feedback: do atendente à melhoria efetiva

**Captura.** O bot do Teams registra, junto de cada resposta, os campos estruturados (`answer`, `source_document`, `source_version`, `confidence_score`, `source_type`) e um botão 👍/👎 com campo de motivo livre.

**Passo 1 — Diagnóstico estruturado.** Checklist fixo antes de classificar:
- Qual foi o `confidence_score` da resposta?
- Quais chunks foram efetivamente recuperados (log de retrieval)?
- O `source_type` estava correto (normativo vs. informal)?
- O erro é reprodutível?

**Passo 2 — Ação combinada, não rota única.** A causa raiz dispara uma ou mais ações em paralelo, com um ID de rastreamento único — porque a maioria dos guardrails da matriz é implementada em "Ambos" (código + prompt), então forçar uma rota só seria artificial:

| Causa raiz identificada | Ações disparadas | Guardrail relacionado |
|---|---|---|
| Documento ausente/desatualizado | Reindexação + verificação de metadado de vigência | G2, G6, F2, F5 |
| Chunk errado recuperado | Ajuste de threshold/chunking (código) + reforço no prompt | G3, N3, F3 |
| Guardrail de comportamento falhou | Ajuste de prompt + validação de pós-processamento | G4, G5, N2, N5, N6 |
| Padrão não coberto por guardrail existente | Proposta de guardrail novo para revisão | — |
| Reclamação de tom/estilo sem erro factual | Backlog contínuo, não bloqueia release | — |

**Passo 3 — Verificação de fechamento.** O item só é marcado como resolvido quando:
1. Virou caso de teste permanente no golden set, com paráfrases;
2. Esse teste passou no regression suite;
3. O fix foi deployado em produção;
4. O atendente que reportou recebeu confirmação ("seu report X foi corrigido na versão Y" ou explicação de por que não era um erro).

**SLA.** Feedback que reproduz um padrão já catalogado (I-1, I-2, I-3) é crítico — tratado antes da próxima release, não vai para backlog.

---

## 2. Regression testing de produto

### 2.1 Golden set — sempre suite completa, nunca fatia temática

Camada A (não-regressão de perguntas frequentes) + Camada B (casos por guardrail, casos de incidente, casos vindos do feedback). **Toda mudança executa a suite inteira**, sem exceção por "tema" — indexar um documento novo pode reordenar o ranking de chunks de temas não relacionados, e um ajuste de prompt para um guardrail é editado no mesmo system prompt que rege todos os outros.

### 2.2 Casos de tensão entre guardrails

Pares que empurram em direções opostas e precisam ser testados juntos:

| Par em tensão | O que testar junto |
|---|---|
| G5/N5 (priorizar normativo, sinalizar FAQ) × F4 (usar FAQ quando é única fonte) | Reforçar G5 não pode fazer o assistente recusar o FAQ quando é a única fonte válida |
| G1/N1 (verificar categoria) × N6 (encaminhar, não resolver) | Reforçar a verificação não pode virar tentativa de resolver tudo via clarificação em vez de encaminhar |
| G3/N3 (retrieval falhou ≠ ausência de informação) × N4 (não inventar valores) | Corrigir falso-negativo de retrieval não pode abrir margem para valor inventado |

### 2.3 Não-determinismo do modelo

Casos de guardrail de prompt (N6, F6, e a parte prompt dos demais) rodam no mínimo 5-10 vezes por execução do suite; o resultado é uma **taxa de sucesso**, não um pass/fail único. Queda na taxa entre releases é regressão, mesmo sem falha isolada visível.

### 2.4 Paráfrases para casos vindos de feedback

Todo caso promovido do feedback (Passo 3) ganha 2-3 paráfrases geradas e revisadas pelo Product Specialist. O caso só é considerado corrigido se passar na frase original **e** nas paráfrases — evita corrigir a string em vez do comportamento.

### 2.5 Critério de aprovação do release

- Zero regressões na Camada A;
- 100% dos guardrails de código passando (bloqueante, sem exceção);
- **F6 bloqueante** (cobertura transversal contra alucinação nos três incidentes);
- Demais guardrails de prompt/ambos: taxa de sucesso ≥ threshold definido (ex.: 95%) — abaixo disso, dispara revisão manual;
- Nenhuma degradação nos pares de tensão (2.2);
- Taxa de erro avaliada **por categoria** (alucinação, documento desatualizado, chunk incorreto, falha de encaminhamento) — piora em qualquer categoria barra o release mesmo com o agregado melhorando;
- Taxa de fallback desnecessário monitorada como critério de release (aumento indica fricção nova, mesmo sem violação de guardrail).

### 2.6 Quando roda

Antes de qualquer mudança de prompt, adição/atualização de documento, alteração de threshold de retrieval ou deploy de código do pipeline — inclusive código gerado por Copilot.

---

## 3. Human-in-the-loop — gatilhos e aprovação

### 3.1 Gatilhos objetivos (elimina autoavaliação)

- **"Prompt toca guardrail existente"**: mapa fixo guardrail → seção do system prompt, derivado da matriz do cenário 2. Qualquer diff que altere uma seção mapeada dispara o fluxo de aprovação automaticamente.
- **"Envolve contradição pendente"**: checagem automática contra a lista de documentos com contradição sinalizados por Compliance (base de G6/F5) — dado de pipeline, não inferência humana.

### 3.2 O que conta como aprovação

Toda aprovação gera um registro auditável com: ID da mudança, guardrail(s) afetado(s), aprovador, data/hora, decisão (**aprovado** / **aprovado com ressalva** / **reprovado**) e justificativa. Nenhum deploy sobe sem esse registro existir.

### 3.3 Critério objetivo para ressalva vs. bloqueio

Uma reprovação do regression suite só pode ser liberada como **ressalva** se, simultaneamente:
- (a) o guardrail afetado não está entre os bloqueantes (F6 e guardrails de código);
- (b) existe prazo compromissado e registrado para o fix definitivo;
- (c) o risco está documentado no registro de aprovação.

Qualquer mudança que não atenda às três condições é **bloqueio automático**.

### 3.4 Tabela de aprovação (pré-deploy)

| Mudança | Gatilho | Aprovador |
|---|---|---|
| Prompt tocando guardrail existente | Mapa guardrail → seção do prompt (3.1) | Product Specialist |
| Mudança envolvendo documento com contradição pendente | Lista automática G6/F5 (3.1) | + Compliance |
| Novo documento normativo | Manual, na ingestão | Product Specialist (metadado) + Compliance (fonte oficial) |
| Novo documento informal (FAQ) | Manual, na ingestão | Product Specialist |
| Threshold de relevância / retrieval | Manual, no PR | Tech Lead + Product Specialist |
| Proposta de guardrail novo | Feedback Passo 2 | Product Specialist formaliza; Compliance valida se normativo |
| Código Copilot (validação/logging/dados de atendente) | Manual, no PR | Tech Lead + Product Specialist |
| Reprovação do regression suite (F6 ou categoria específica) | Automático, no CI | Product Specialist, seguindo critério objetivo (3.3) |
| Reindexação sem mudança de metadado | — | Nenhuma, só log |

### 3.5 HITL em tempo real

`confidence_score` abaixo do threshold definido, em tópicos sensíveis, roteia a resposta para um atendente humano em vez de entregá-la automaticamente ao cliente final.

---

## 4. Itens pendentes de validação explícita

Os pontos abaixo têm proposta de rascunho, mas **não são decisão exclusiva do Product Specialist** — dependem de aceite formal de Tech Lead e/ou Compliance. Enquanto não validados, o harness opera com o comportamento default indicado, deliberadamente mais conservador.

### 4.1 Pendentes com Tech Lead e Compliance (governança do processo)

| Item | Proposta de rascunho | Validação necessária | Default enquanto pendente |
|---|---|---|---|
| Backup de aprovador | Head de Produto substitui Product Specialist; segundo dev sênior substitui Tech Lead | Aceite de Tech Lead e liderança de Produto | Sem substituto definido, deploy aguarda o titular — sem bypass |
| SLA de aprovação | 24-48h úteis por item | Aceite de Tech Lead e Compliance quanto à viabilidade | Sem SLA formal, item fica na fila sem prazo — sinalizado como risco de gargalo |
| Autoridade de desempate (Product Specialist × Tech Lead) | Escalonamento para liderança conjunta de Produto e Engenharia | Aceite de ambas as lideranças | Mudança fica bloqueada até desempate manual, mesmo sob prazo apertado |
| Via de emergência | Aprovação retroativa em até 24h, registrada com justificativa de urgência | Aceite formal de Compliance como via legítima (não bypass informal) | Sem via aceita, prazo apertado não justifica pular o registro de aprovação (3.2) |

### 4.2 Pendentes com Compliance e Operação (calibração dependente de dado)

| Item | Proposta de rascunho | Validação necessária | Default enquanto pendente |
|---|---|---|---|
| Threshold de `confidence_score` para HITL runtime | Valor inicial conservador (baseado no baseline de 12% de erro), revisado quinzenalmente com dado real | Compliance valida o valor inicial e a cadência de revisão | Threshold conservador vigora até calibração aprovada — prioriza encaminhamento a humano sobre resposta automática arriscada |
| Lista fechada de tópicos sensíveis | Financeiro, segurança, elegibilidade (com base nos três incidentes conhecidos) | Compliance valida exposição regulatória e completude da lista | Lista provisória vale até fechamento formal |
| Cobertura de HITL fora do horário / volume alto no go-live | Comportamento default: recusar responder automaticamente e indicar canal alternativo quando não há atendente disponível | Decisão de staffing/plantão pertence à Operação | Sem atendente disponível, o sistema nunca responde sozinho em tópico sensível |

**Encaminhamento.** Os itens da seção 4 seguem como pauta de validação formal com Tech Lead e Compliance antes do go-live. O piloto atual (5 atendentes) opera com os defaults indicados — o custo é mais fricção (mais encaminhamento a humano, mais bloqueio em caso de dúvida), não mais risco.
