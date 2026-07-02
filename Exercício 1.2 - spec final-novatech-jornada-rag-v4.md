# NovaTech — Assistente de IA para Atendimento
## Jornada do atendente + Pipeline de manutenção da RAG
**Versão:** 4.0  
**Data:** 05/06/2026  
**Escopo:** Jornada do atendente com fallback expandido e triagem de volume no feedback. Pipeline técnico de manutenção da RAG com três conectores de ingestão heterogêneos, classificação semi-automática de status, estratégia tabular para planilhas, recuperação multi-documento e suite de validação expandida.

---

## 1. Contexto operacional

| Dado | Valor |
|---|---|
| Equipe de atendimento | 45 atendentes |
| Volume médio | 320 chamados/dia |
| Chamados com consulta documental | ~60% (~192/dia) |
| Taxa de escalada ao supervisor | 15% dos chamados |
| Distribuição de dúvidas | Prazos 35% · Frete 25% · Devolução 20% · Outros 20% |
| Fontes documentais | SharePoint (~800 docs) · Confluence (~400 páginas) · Pasta de rede (planilhas mensais) |
| Tipos de conteúdo | Manuais operacionais · Políticas de compliance · Tabelas de SLA · Regras de frete · Normas de segurança de carga |
| Ciclo de atualização documental | Mensal — 3 áreas sem processo unificado (Operações, Compliance, Comercial) |
| Infraestrutura disponível | Microsoft 365 E3 + Azure AI Services |

---

## 2. Jornada do atendente

### 2.1 Fluxo principal

**Etapa 1 — Recebimento da dúvida com contexto**

O atendente identifica o tier do cliente (Gold, Silver ou Standard) antes de qualquer consulta. Esse dado é enviado junto com a pergunta ao assistente e determina qual SLA se aplica, quais obrigações contratuais estão em jogo e qual nível de detalhe é necessário na resposta. Uma pergunta sobre prazo de incidente crítico tem resposta diferente para um cliente Gold (30 min) e um Standard (2h).

**Etapa 2 — Consulta ao assistente**

O atendente digita a dúvida em linguagem natural no painel integrado ao ambiente de atendimento. O assistente recebe tier + pergunta como contexto unificado.

**Etapa 3 — Busca na base vetorial com metadados**

O assistente recupera chunks relevantes junto com seus metadados: status do documento (vigente, em revisão, informal), versão, data de vigência e fonte. A busca não retorna apenas conteúdo — retorna conteúdo com proveniência rastreável até a fonte original.

**Etapa 4 — Decisão de versão para PROC-042**

Se a pergunta envolver frete especial, o assistente resolve automaticamente qual versão aplicar consultando a data de abertura do chamado:

- Chamados anteriores a 01/12/2023 → PROC-042 v1
- Chamados a partir de 01/12/2023 → PROC-042 v2

Essa decisão não é subjetiva nem delegada ao atendente — é uma regra objetiva executada pelo sistema com base no metadado `regra_aplicacao` do chunk recuperado.

**Etapa 5 — Classificação da fonte**

O assistente verifica se os chunks recuperados vêm de normativo oficial (POL, PROC, SLA) ou do FAQ operacional. Se a única fonte disponível for o FAQ, o assistente sinaliza explicitamente: *"essa informação vem do FAQ operacional, que não foi validado por Compliance."* O atendente decide conscientemente se usa a informação com essa ressalva ou escala para o supervisor.

**Etapa 6 — Entrega da resposta**

A resposta inclui obrigatoriamente:

- Conteúdo direto e objetivo
- Documento de origem com versão e data
- Tier do cliente aplicado à resposta
- Alerta quando o documento estiver marcado como "em revisão" (ex: PROC-043)

**Etapa 7 — Verificação e uso**

O atendente valida a resposta antes de repassar ao cliente. Se discordar, escala para o supervisor pelo mesmo fluxo dos demais fallbacks e registra o desacordo no fluxo de feedback.

---

### 2.2 Guardrails de comportamento do assistente

1. Nunca inventar prazos, valores ou multiplicadores que não estejam na documentação indexada
2. Sempre citar a fonte: documento, versão e data
3. Quando não encontrar resposta com confiança suficiente, dizer explicitamente e sugerir escalar para o supervisor
4. Sinalizar a origem quando a fonte for o FAQ operacional (não validado por Compliance)
5. Sinalizar quando o documento consultado estiver marcado como "em revisão"
6. Aplicar automaticamente a versão correta da PROC-042 com base na data de abertura do chamado — não delegar essa decisão ao atendente
7. Responder em português formal e acessível

---

### 2.3 Fluxo de fallback — três caminhos

**Caminho 1 — Assistente sem confiança**

A RAG não encontrou chunks suficientes ou o score de relevância ficou abaixo do limiar configurado. O assistente sinaliza a incerteza e o atendente escala para o supervisor. O cliente aguarda — nenhuma informação incerta chega ao atendimento.

**Caminho 2 — Fonte informal sem normativo correspondente**

O assistente encontrou resposta, mas a única fonte é o FAQ operacional. Sinaliza ao atendente o status da fonte. O atendente decide conscientemente se usa a informação com a ressalva ou escala — não é bloqueado automaticamente, mas tem a informação para avaliar o risco.

Casos típicos deste caminho (identificados na análise documental):
- Item 38: processo para carga danificada em trânsito (prazo 48h, `sinistros@novatech.com.br`)
- Item 3: fluxo prático para devolução de carga perigosa (ramal 4500 — Gestão de Riscos)
- Item 27: parâmetros de alerta de rastreamento por rota (Norte: até 10 dias úteis; Sul/Sudeste: alerta após 3 dias)

**Caminho 3 — Atendente discorda da resposta**

O atendente tem autonomia para rejeitar qualquer resposta, mesmo quando o assistente entrega com alta confiança e fonte oficial. Escala para o supervisor e registra o desacordo no fluxo de feedback.

---

### 2.4 Fluxo de feedback — quatro tipos de sinal com triagem de volume

Com 45 atendentes gerando feedback simultaneamente, a fila de revisão recebe triagem de volume antes de chegar às áreas responsáveis.

**Tipos de sinal:**

| Tipo | Descrição | Exemplo concreto |
|---|---|---|
| Resposta incorreta | Valor errado ou interpretação falha do documento | Assistente informa desconto a partir de 10 fretes (FAQ desatualizado) em vez de 8 (PROC-042 v2) |
| Informação incompleta | Exceção não citada ou obrigação omitida | SLA Gold citado sem mencionar gerente dedicado ou relatório mensal |
| Documento desatualizado | Versão errada aplicada na recuperação | PROC-042 v1 usada em chamado aberto após 01/12/2023 |
| Gap documentado | Informação útil existe apenas no FAQ, sem normativo correspondente | Processo de sinistros, ramal 4500 para carga perigosa, parâmetros de rastreamento |

**Triagem de volume:**

| Condição | Tratamento |
|---|---|
| Sinal isolado (1 atendente) | Fila de revisão normal — ciclo mensal |
| 15+ atendentes sinalizam o mesmo item em até 3 dias | Prioridade alta — notificação imediata à área responsável fora do ciclo mensal |

A triagem evita que ruído individual consuma capacidade das áreas revisoras, e garante que problemas sistêmicos sejam tratados antes do próximo ciclo. O quarto tipo — gap documentado — é o insumo principal para decisão de formalizar conhecimento operacional em normativo oficial.

---

## 3. Pipeline de manutenção da RAG

### 3.1 Fontes monitoradas e estratégias de ingestão

| Fonte | Volume | Formato | Estratégia de ingestão | Observação |
|---|---|---|---|---|
| SharePoint corporativo | ~800 documentos | PDF e Word | Extração de texto estruturado · seções e tabelas preservadas como bloco | Webhook para detecção de mudança |
| Confluence (wiki interna) | ~400 páginas | HTML/wiki | Extração via API · chunking por página · histórico de versões nativo aproveitado para metadado de vigência | Versionamento nativo reduz trabalho de classificação manual |
| Pasta de rede | Planilhas mensais | Excel (.xlsx) | Extração tabular linha a linha · metadado de coluna e contexto da tabela preservados · chunking não destrói estrutura | Monitoramento por hash — sem webhook nativo · risco crítico de desatualização silenciosa |
| FAQ operacional | ~50 itens | Texto | Ingestão com `fonte: faq` · peso reduzido · nunca priorizado sobre normativo conflitante | Não validado por Compliance — fonte secundária |

---

### 3.2 Pesos de recuperação por status

| Status | Comportamento na busca | Sinalização ao atendente |
|---|---|---|
| Vigente | Peso total | Nenhuma |
| Em revisão | Peso reduzido | "Este documento está em revisão pelo Compliance e pode sofrer alterações" |
| FAQ informal | Fonte secundária | "Informação proveniente do FAQ operacional — não validada por Compliance" |
| Revogado | Removido da base vetorial | Não aparece em respostas |

---

### 3.3 Etapas do ciclo técnico

**Etapa 1 — Detectar mudança**

Monitoramento automático por fonte:
- SharePoint e Confluence: webhook nativo — evento disparado a cada alteração de arquivo ou página
- Pasta de rede (planilhas): monitoramento por hash — comparação do hash do arquivo a cada ciclo programado (diário recomendado), dado que não há webhook disponível. Planilhas atualizadas mensalmente sem processo unificado são o principal vetor de desatualização silenciosa da base

**Etapa 2 — Classificar status — semi-automático**

Com ~1.200 artefatos documentais, a classificação não pode ser totalmente manual. O processo opera em dois modos:

- **Automático** (sem revisão humana): documentos com metadado de status explícito no SharePoint ou histórico de versões no Confluence são classificados diretamente como Vigente, Revogado ou Em revisão
- **Revisão humana** apenas para casos ambíguos: ausência de metadado, coexistência de versões sem regra de transição documentada (como PROC-042 v1 e v2), ou documentos movidos de pasta sem indicação de status

Regras de classificação para os casos conhecidos:
- PROC-042 v1: status `coexistente` · `regra_aplicacao`: chamados anteriores a 01/12/2023
- PROC-042 v2: status `vigente` · `regra_aplicacao`: chamados a partir de 01/12/2023
- PROC-043: status `em_revisao` até publicação da versão revisada pelo Compliance
- FAQ-atendimento: status `faq` permanente — só promovido a `oficial` após validação formal

> **Nota operacional:** Documentos revogados são removidos da base vetorial — não sobrescritos. Sobrescrever mantém o chunk no índice e permite recuperação residual. A PROC-042 v1 é exceção: mantida com status `coexistente` enquanto existirem chamados históricos que a referenciam.

**Etapa 3 — Re-indexar com metadados**

Cada chunk carrega os seguintes metadados obrigatórios:

- `status`: vigente | revogado | em_revisao | coexistente | faq
- `fonte_tipo`: oficial | faq
- `fonte_sistema`: sharepoint | confluence | planilha_rede
- `documento`: identificador (ex: POL-001, PROC-042-v2, SLA-2024)
- `versao`: número da versão
- `vigencia_inicio`: data de início de vigência
- `vigencia_fim`: data de fim (quando aplicável)
- `regra_aplicacao`: regra contextual (ex: "aplicar para chamados abertos a partir de 01/12/2023")
- `tipo_conteudo`: texto | tabela | lista — relevante para estratégia de recuperação

Estratégias específicas por formato:
- **PDFs e Word (SharePoint):** chunking por seção com preservação de cabeçalhos; tabelas extraídas como bloco único com contexto da seção pai
- **Confluence:** chunking por página; vigência extraída automaticamente do histórico de versões nativo — reduz dependência de classificação manual
- **Planilhas Excel:** extração tabular linha a linha; cada linha carrega metadado de cabeçalho de coluna e nome da aba; a estrutura relacional da tabela é preservada no chunk, não destruída pelo chunking de texto

**Etapa 3b — Configurar recuperação multi-documento**

Perguntas que cruzam domínios exigem recuperação de múltiplos documentos sem referência cruzada nativa entre eles.

Exemplo:
> *"Quanto tempo um cliente Gold tem para devolver uma carga de frete especial com destino ao Norte?"*

Essa pergunta exige chunks de:
- POL-001 (prazo base de devolução: 7 dias úteis)
- PROC-042 v2 (prazo adicional para frete especial ao Norte: +3 dias úteis; multiplicador regional: 1,8)
- SLA-2024 (obrigações específicas do tier Gold)

Os três documentos não se referenciam entre si na base original. A lógica de recuperação precisa combinar múltiplas fontes para construir uma resposta coerente. Isso é uma decisão de arquitetura — não de indexação.

**Etapa 4 — Validar com suite expandida de casos reais**

A suite mínima de testes é derivada da análise documental (tabela 3.5) e expandida para cobrir as categorias de conteúdo identificadas nos novos dados de discovery:

| Categoria | Caso de teste | Resposta esperada | Risco |
|---|---|---|---|
| Frete | Desconto de volume: a partir de quantos fretes? | 8 fretes/mês — 5% automático (PROC-042 v2) | Alto — FAQ diz 10 |
| Frete | Versão da PROC-042 para chamado aberto em 15/01/2024? | v2 — posterior a 01/12/2023 | Alto — v1 coexiste |
| Frete | Multiplicador para carga >3.000kg destino Norte (chamado atual)? | Fator 1,4 × multiplicador 1,8 = 2,52 (PROC-042 v2) | Alto — v1 dá 2,4 |
| SLA | Tempo de resposta para incidente crítico Gold? | 30 minutos (SLA-2024) | Médio — FAQ omite |
| SLA | SLA Gold inclui gerente dedicado? | Sim (SLA-2024, seção 2) | Médio — FAQ não menciona |
| SLA | Existe tier Platinum? | Não — descontinuado em 2022 | Baixo — FAQ cobre |
| Devolução | Prazo base de devolução? | 7 dias úteis após recebimento confirmado (POL-001) | Baixo |
| Segurança | Fluxo para carga danificada em trânsito? | 48h, fotos, laudo, sinistros@novatech.com.br (FAQ — sem conflito) | Baixo |
| Segurança | Fluxo para carga perigosa? | Ramal 4500 — Gestão de Riscos (FAQ) · PROC-043 em revisão | Médio — PROC-043 instável |

Regressão detectada em qualquer caso bloqueia a publicação e aciona notificação à área responsável. O documento volta para reclassificação na etapa 2.

**Etapa 5 — Publicar com registro**

A nova versão da base entra em produção com:

- Data e hora da publicação
- Lista de documentos alterados e seus novos status
- Responsável pela aprovação
- Referência à versão anterior (arquivada, não deletada, acessível para auditoria de chamados históricos)

---

## 4. Diagnóstico de origem — lacunas incorporadas

| Lacuna identificada | Versão incorporada | Onde no design |
|---|---|---|
| PROC-042 v1 e v2 coexistem sem hierarquia | v3.0 | Jornada: nó de decisão por data (etapa 4) · Pipeline: metadado `regra_aplicacao` (etapa 3) |
| FAQ como fonte não validada | v3.0 | Jornada: fallback de fonte informal (etapa 5) · Pipeline: metadado `fonte` + peso reduzido |
| PROC-043 em revisão pelo Compliance | v3.0 | Jornada: alerta na entrega (etapa 6) · Pipeline: status "em revisão" (etapa 2) |
| Tier do cliente ausente como entrada | v3.0 | Jornada: tier identificado na etapa 1 |
| Ilhas documentais sem referência cruzada | v3.0 | Pipeline: recuperação multi-documento (etapa 3b) |
| Suite de validação genérica | v3.0 | Pipeline: casos de teste derivados da tabela 3.5 (etapa 4) |
| Documentos revogados não removidos | v3.0 | Pipeline: distinção remover vs sobrescrever (etapa 2) |
| Três fontes heterogêneas (SharePoint, Confluence, planilhas) | **v4.0** | Pipeline: três conectores com estratégias distintas (etapas 1 e 3) |
| Planilhas mensais sem processo unificado | **v4.0** | Pipeline: monitoramento por hash + alerta de desatualização silenciosa (etapa 1) |
| ~1.200 artefatos — classificação manual inviável | **v4.0** | Pipeline: classificação semi-automática com revisão humana apenas para ambíguos (etapa 2) |
| 45 atendentes gerando feedback simultâneo | **v4.0** | Jornada: triagem de volume no feedback (seção 2.4) |
| Normas de segurança e manuais operacionais não cobertos | **v4.0** | Pipeline: suite expandida com categoria segurança (etapa 4) |
| Confluence com versionamento nativo subutilizado | **v4.0** | Pipeline: extração automática de vigência do histórico (etapa 3) |

---

## 5. Perguntas de discovery em aberto

**P1 — Governança do FAQ** *(prioridade máxima)*  
Quem é o dono formal do FAQ-atendimento? Existe processo de revisão periódica ou ele cresce organicamente sem controle?  
*Antes de qualquer correção pontual, é preciso saber se o problema é de conteúdo ou de governança. Se não houver dono, qualquer atualização feita agora vai se deteriorar novamente.*

**P2 — Ramal 4500**  
O redirecionamento para Gestão de Riscos para carga perigosa é processo formal ou workaround criado pelo time?  
*Define se o item 3 do FAQ deve ser formalizado em POL-001 ou tratado como exceção gerenciada.*

**P3 — Fluxo de sinistros**  
O processo do item 38 (48h, `sinistros@novatech.com.br`) é reconhecido formalmente pelo Jurídico?  
*É o item mais útil do FAQ e o principal candidato a virar normativo oficial.*

**P4 — PROC-043**  
Qual o prazo para conclusão da revisão pelo Compliance? Existe versão provisória em uso?  
*Enquanto não houver versão vigente, o pipeline mantém status "em revisão" e o assistente sinaliza ao atendente.*

**P5 — Contratos na v1**  
Existem contratos de clientes que ainda referenciam explicitamente a PROC-042 v1?  
*Se sim, a regra de aplicação por data de chamado pode não ser suficiente — pode haver clientes com direito contratual à v1 independente da data.*

**P6 — Planilhas de rede: responsável e frequência real**  
Quem atualiza as planilhas mensais da pasta de rede? A atualização é realmente mensal ou acontece de forma ad hoc?  
*Determina a frequência mínima do monitoramento por hash e o risco de janela de desatualização silenciosa.*

**P7 — Confluence: páginas ativas vs abandonadas**  
Qual o percentual das ~400 páginas do Confluence que está ativamente mantido? Há páginas sem revisão há mais de 6 meses?  
*Páginas obsoletas indexadas com peso total são tão problemáticas quanto documentos revogados não removidos.*

---

*Documento gerado a partir da análise documental NovaTech (05/06/2026) e dos dados de discovery de 05/06/2026. Todas as decisões de design derivam dos documentos POL-001, PROC-042 v1 e v2, SLA-2024, FAQ-atendimento e dos dados operacionais fornecidos. Validação com stakeholders necessária antes da implementação.*
