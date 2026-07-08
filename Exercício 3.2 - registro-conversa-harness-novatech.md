# Registro da Conversa — Harness de Produto NovaTech

Este documento reproduz integralmente as mensagens trocadas nesta conversa (input do usuário e respostas de Claude), organizadas por turno.

---

## Turno 1

### Input (Usuário)

> **Anexo:** `Exercicio-2_2-guardrails-matriz-incidentes.md` (matriz de rastreabilidade guardrails × incidentes do cenário 2 — DEVE / NÃO DEVE / QUANDO EM DÚVIDA)

Você é um Product Specialist participando do projeto de assistente de IA para a NovaTech.

**Cenário:**
O assistente de IA da NovaTech está em desenvolvimento. O pipeline de RAG está funcional, os primeiros endpoints foram implementados, e o bot do Teams responde perguntas de teste. O assistente vai evoluir após o go-live. Você define, do ponto de vista de produto, como garantir que ele melhore sem degradar.

O que foi construído até agora:
- O pipeline de ingestão processa 847 documentos e os indexa no Azure AI Search.
- O query endpoint recebe perguntas via POST, busca chunks, e retorna respostas com citação de fonte.
- O bot do Teams funciona em ambiente de staging, acessível por 5 atendentes-piloto.
- O AGENTS.md (construído pelo time no cenário 2), as specs SDD e as skills estão no repositório e sendo usadas pelo Copilot.
- Os guardrails de produto foram formalizados pelo Product Specialist (cenário 2) em DEVE / NÃO DEVE / QUANDO EM DÚVIDA.
- Testes de integração cobrem ~75% do código.

O que foi descoberto durante o desenvolvimento:
- Em testes internos, 12% das respostas estavam incorretas: alucinação, documento desatualizado, e chunk incorreto recuperado.
- As respostas do assistente são retornadas como texto livre. Não há um formato estruturado garantindo que campos obrigatórios (fonte, confiança) sempre estejam presentes — quando o modelo "esquece" de incluir a fonte, nada impede a resposta de seguir.
- Um desenvolvedor gerou com o Copilot um módulo de feedback que ignorou regras do AGENTS.md (não usou Zod, logou dados sensíveis do atendente).
- A NovaTech pediu uma demonstração para a diretoria em 2 semanas.

**Conceitos-chave:**
- Structured Outputs: em vez de deixar o modelo responder em texto livre, define-se um formato (JSON) que a resposta DEVE seguir, com campos obrigatórios (ex: answer, source_document, confidence_score). Respostas que não seguem o formato são rejeitadas programaticamente.
- Human-in-the-Loop (HITL): pontos do fluxo onde a validação final é de um humano, não do modelo.
- Harness de produto: define quais métricas de qualidade são monitoradas, como o feedback de usuários é processado, e como mudanças no assistente são validadas antes de ir a produção (regression testing de produto).

**Sua tarefa como product specialist:**
A partir das informações fornecidas acima, projete um harness de produto que cubra:
1. Processo de feedback: como o feedback do atendente vira melhoria (novo documento? ajuste de prompt? reindexação?).
2. Regression testing de produto: antes de mudar o prompt ou adicionar documentos, como verificar que as respostas existentes não pioraram E que os guardrails do cenário 2 continuam sendo respeitados.
3. Ponto de human-in-the-loop: quais mudanças no assistente exigem aprovação humana antes de ir a produção, e quem aprova.

### Resposta (Claude)

# Harness de Produto — Assistente de IA NovaTech

O harness parte de um princípio simples: **o assistente vai evoluir, mas os 18 guardrails do cenário 2 (G1-G6, N1-N6, F1-F6) não podem regredir nunca**. Toda mudança — documento novo, ajuste de prompt, alteração de threshold, código gerado por Copilot — passa pelos três mecanismos abaixo antes de chegar em produção.

## 1. Processo de feedback: do atendente à melhoria

**Captura.** O bot do Teams registra, junto de cada resposta, os campos estruturados (`answer`, `source_document`, `source_version`, `confidence_score`, `source_type`) e um botão 👍/👎 com campo de motivo livre. Isso é o que torna o feedback classificável — sem *structured outputs*, "a resposta estava errada" não diz se foi alucinação, documento desatualizado ou chunk errado.

**Triagem (Product Specialist, diária durante o piloto).** Cada feedback negativo é classificado em uma de quatro rotas, usando a mesma lógica da matriz de guardrails:

| Causa raiz identificada | Ação | Guardrail relacionado |
|---|---|---|
| Documento ausente ou desatualizado no índice | Reindexação pontual + verificação de metadado de vigência antes de indexar | G2, G6, F2, F5 |
| Chunk errado recuperado (score baixo, mas modelo respondeu mesmo assim) | Ajuste de threshold de relevância ou estratégia de chunking (mudança de **código**) | G3, N3, F3 |
| Resposta correta na fonte, mas guardrail de comportamento falhou (ex.: esqueceu de sinalizar FAQ informal, misturou versões) | Vira caso de teste permanente no golden set + ajuste de prompt | G4, G5, N2, N5, N6 |
| Reclamação de tom/estilo, sem erro factual | Backlog de melhoria contínua, não bloqueia release | — |

Toda entrada gera um registro com: ID do guardrail afetado (se houver), padrão de incidente (I-1/I-2/I-3 ou novo padrão), tipo de correção proposta (documento/prompt/código/ambos) e prioridade. Isso mantém o harness rastreável do mesmo jeito que a matriz de incidentes já é.

**SLA.** Feedback que reproduz um padrão já catalogado como I-1, I-2 ou I-3 é tratado como **crítico** — corrigido antes da próxima release, não vai para backlog. Feedback de padrão novo é triado e priorizado normalmente.

## 2. Regression testing de produto

O ponto central: regression testing aqui não é "a resposta não mudou", é **"a resposta não mudou para pior E nenhum guardrail foi violado"**. Isso exige um golden set com duas camadas:

**Camada A — casos de não-regressão.** Amostra de perguntas frequentes com respostas já validadas como corretas. Compara-se `source_document`, `source_version` e equivalência semântica da resposta (não string exata, já que o modelo varia). Qualquer queda de qualidade nessa amostra bloqueia o release.

**Camada B — casos de guardrail.** No mínimo 1-2 casos de teste por guardrail (G1-G6, N1-N6, F1-F6), incluindo os cenários originais dos incidentes I-1/I-2/I-3 como casos permanentes que "nunca mais podem falhar assim". Aqui a verificação muda de acordo com a camada de implementação já definida na matriz:

- **Guardrails de código** (G2, N1, N2, F1, F3, e a parte código de G3/G4/G5/G6/N3/N4/N5/F2/F4/F5) → teste automatizado, determinístico, **bloqueante**. Ex.: `N2` — a resposta contém "PROC-042" sem sufixo de versão? Falha automática.
- **Guardrails de prompt puro** (N6, F6) → não são 100% verificáveis por assertion determinística. Avaliação amostral por revisor humano ou LLM-as-judge, com threshold (ex.: ≥95% de conformidade) — abaixo disso, exige revisão manual antes de liberar, mas não trava o pipeline sozinho.

**Quando roda.** Antes de qualquer mudança de prompt, adição/atualização de documento, alteração de threshold de retrieval ou deploy de código do pipeline — inclusive quando o código vem do Copilot.

**Critério de aprovação do release:**
- Zero regressões na Camada A;
- 100% dos guardrails de código passando (bloqueante, sem exceção);
- ≥ threshold definido nos guardrails de prompt/ambos (não bloqueante isolado, mas dispara revisão);
- Taxa geral de resposta incorreta não pode superar o baseline atual de 12% — a meta é reduzir, nunca regredir.

Quando um documento novo entra (ex.: PROC-042-v3), o golden set relacionado ao tema é re-executado automaticamente — é assim que se evita repetir o I-2 com uma versão futura.

## 3. Pontos de human-in-the-loop

HITL aqui tem dois momentos distintos: **antes do deploy** (mudança no assistente) e **em tempo real** (resposta individual arriscada).

**Pré-deploy**, por tipo de mudança:

| Mudança | Aprovação obrigatória? | Quem aprova |
|---|---|---|
| Prompt tocando guardrail existente (G/N/F) | Sim | Product Specialist; + Compliance se envolver documento com contradição pendente (G6/F5) |
| Novo documento normativo indexado (política/procedimento) | Sim | Product Specialist confirma metadado de vigência; Compliance confirma que é fonte oficial |
| Novo documento FAQ / informal | Sim, revisão leve | Product Specialist (`source_type: informal` correto) |
| Mudança de threshold de relevância / lógica de retrieval (código) | Sim | Tech Lead + Product Specialist (revisão conjunta, pois afeta F3/G3/N3) |
| Código gerado por Copilot que altera validação/logging/dados sensíveis | Sim, sempre — nunca merge automático | Tech Lead (revisão de código) + Product Specialist (revisão de conformidade com AGENTS.md/guardrails) |
| Resposta individual de baixa confiança em tema sensível (runtime, não deploy) | Sim, HITL em tempo real | Atendente humano assume antes de responder ao cliente final (roteamento automático quando confidence_score < threshold em tópicos financeiro/segurança) |
| Correção de conteúdo simples (reindexação de doc já aprovado, sem mudança de metadado) | Não, mas registrado em log | — |

O caso do módulo de feedback que ignorou o AGENTS.md (sem Zod, logando dado sensível do atendente) é exatamente o motivo da linha de "código gerado por Copilot": geração assistida não dispensa revisão — pelo contrário, é o tipo de mudança que mais precisa dela, porque o guardrail pode ser violado silenciosamente sem que ninguém tenha "decidido" violá-lo.

**Runtime (por resposta individual).** Aqui o `confidence_score` do structured output vira o gatilho: resposta abaixo do threshold definido em tópicos sensíveis (financeiro, segurança, elegibilidade) é roteada para um atendente humano em vez de ser entregue ao cliente final automaticamente.

## Sobre a demo em 2 semanas

Vale nomear isso explicitamente para a diretoria: dá para ter uma "pista rápida" para a demo — regression testing focado no golden set dos 3 incidentes documentados, ambiente de staging, 5 atendentes-piloto — sem abrir mão do HITL em código sensível (o caso do Copilot mostra por quê). O harness completo (regression cobrindo os 18 guardrails, HITL runtime por confidence_score) é o que precisa estar pronto para o go-live, não necessariamente para a demo.

---

## Turno 2

### Input (Usuário)

Com o Harness de produto criado, analise se o processo de feedback é completo (do atendente até a melhoria efetiva).

### Resposta (Claude)

Boa pergunta — vale revisitar com espírito crítico, porque o processo que desenhei tem lacunas reais entre "atendente clica 👎" e "melhoria efetivamente em produção". Vou mapear onde a cadeia quebra.

## Veredito: incompleto — quatro elos faltando

**1. O diagnóstico de causa raiz está tratado como dado, não como processo.**
Eu escrevi "o Product Specialist classifica em uma das quatro rotas" como se fosse trivial. Mas como ele decide se foi alucinação, documento desatualizado ou chunk errado? O processo não descreve a investigação: checar o `confidence_score` daquela resposta, olhar quais chunks foram de fato recuperados (logs de retrieval), verificar se o `source_type` estava correto, tentar reproduzir a pergunta. Sem esse passo explícito, a triagem vira palpite.

**2. O roteamento assume causa única — mas a própria matriz de guardrails não é assim.**
Minha tabela de triagem manda cada feedback para *uma* rota (documento OU código OU prompt). Só que olhando a matriz original, a maioria dos guardrails (G1, G3, G4, G5, G6, N3, N4, N5, F2, F4, F5) é **"Ambos"** — precisa de código e prompt juntos. Um feedback que reproduz o padrão do I-1, por exemplo, pode exigir reindexação de metadado **e** ajuste de prompt **e** uma verificação de código simultaneamente. O processo como está forçaria uma escolha artificial entre as três.

**3. Falta o elo entre "decisão de correção" e "isso realmente virou golden set case e foi testado".**
Eu disse "vira caso de teste no golden set" como se fosse automático. Mas quem garante que o caso reportado por aquele atendente específico foi, de fato, adicionado à Camada B do regression suite antes do próximo deploy? Sem um ID de rastreamento ligando feedback → caso de teste → guardrail testado, é fácil um item de feedback "morrer" na triagem sem nunca virar teste.

**4. Não existe fechamento de loop com quem reportou.**
O processo não diz o que o atendente vê depois. Se ele reporta um erro e nunca sabe se foi corrigido, a taxa de feedback despenca com o tempo — é um padrão bem documentado em produtos com canal de feedback. Precisa de uma notificação mínima ("seu report X foi corrigido na versão Y") para manter o canal vivo.

## Lacunas secundárias

| Lacuna | Por que importa |
|---|---|
| Feedback 👍 não é usado para nada | Perde sinal para consolidar o golden set e calibrar o threshold de `confidence_score` — só o negativo alimenta melhoria |
| "Antes da próxima release" sem cadência definida | Sem uma cadência de release explícita (semanal? sob demanda?), o SLA de "crítico" não tem prazo real |
| Sem distinção entre ruído isolado e padrão recorrente | Um 👎 isolado e dez 👎 no mesmo tópico não deveriam ter a mesma prioridade — falta um limiar de volume para escalar automaticamente |
| Sem caminho para guardrail novo | A triagem só mapeia para G1-G6/N1-N6/F1-F6 já existentes. Se o feedback revelar um padrão de incidente que a matriz do cenário 2 não cobre, não há processo para formalizar um guardrail novo |
| Desenhado para escala de piloto | Triagem diária manual por uma pessoa funciona com 5 atendentes-piloto. Não há nada definido para quando o volume crescer no go-live |

## O que eu acrescentaria

Para fechar o loop de fato, eu inseriria três passos explícitos que faltam entre a triagem e o "processo de feedback: como o feedback do atendente vira melhoria":

1. **Diagnóstico estruturado antes da classificação** — checklist fixo (confidence_score, chunks recuperados, source_type, reprodução da pergunta) que qualquer pessoa (não só quem já conhece o histórico) consegue seguir.
2. **Ação combinada, não rota única** — a triagem gera uma ou mais ações (documento + prompt + código, quando aplicável), cada uma rastreada com o mesmo ID do feedback original.
3. **Verificação de fechamento** — o item só é marcado como "resolvido" quando: (a) o caso virou teste no golden set, (b) o teste passou no regression suite, (c) o fix foi deployado, e (d) o atendente que reportou recebeu uma confirmação.

Sem esses três passos, o processo descreve bem "o que fazer com o feedback" mas não garante que o ciclo se fecha — e é exatamente esse tipo de lacuna silenciosa que gerou o I-3 original (uma falha que existia mas ninguém tinha visibilidade dela até virar incidente).

---

## Turno 3

### Input (Usuário)

Refaça o documento de harness de produto acrescentando os três passos sugeridos.

### Resposta (Claude)

# Harness de Produto — Assistente de IA NovaTech

O harness parte de um princípio simples: **o assistente vai evoluir, mas os 18 guardrails do cenário 2 (G1-G6, N1-N6, F1-F6) não podem regredir nunca**. Toda mudança — documento novo, ajuste de prompt, alteração de threshold, código gerado por Copilot — passa pelos três mecanismos abaixo antes de chegar em produção.

## 1. Processo de feedback: do atendente à melhoria efetiva

**Captura.** O bot do Teams registra, junto de cada resposta, os campos estruturados (`answer`, `source_document`, `source_version`, `confidence_score`, `source_type`) e um botão 👍/👎 com campo de motivo livre. Isso é o que torna o feedback diagnosticável.

**Passo 1 — Diagnóstico estruturado (antes de classificar).** Todo feedback negativo passa por um checklist fixo:
- Qual foi o `confidence_score` da resposta? Estava baixo e mesmo assim respondeu (sinal de F3/N3 não acionado)?
- Quais chunks foram efetivamente recuperados (log de retrieval)? O chunk relevante existia no índice e não foi selecionado, ou não existia?
- O `source_type` estava correto (normativo vs. informal)? Foi sinalizado quando deveria (G5/N5)?
- A pergunta é reprodutível — rodar de novo produz o mesmo erro, ou foi uma falha pontual?

Só depois desse checklist o item recebe uma classificação de causa.

**Passo 2 — Ação combinada, não rota única.** A tabela de causas gera **uma ou mais ações em paralelo**, com um único ID de rastreamento amarrando tudo — porque a própria matriz de guardrails já mostra que a maioria das regras (G1, G3, G4, G5, G6, N3, N4, N5, F2, F4, F5) é implementada em **"Ambos"** (código + prompt):

| Causa raiz identificada | Ações disparadas (podem ser combinadas) | Guardrail relacionado |
|---|---|---|
| Documento ausente ou desatualizado no índice | Reindexação pontual + verificação de metadado de vigência | G2, G6, F2, F5 |
| Chunk errado recuperado (score baixo, resposta saiu mesmo assim) | Ajuste de threshold/chunking (código) + reforço de instrução no prompt, quando o guardrail é "Ambos" | G3, N3, F3 |
| Guardrail de comportamento falhou (ex.: não sinalizou FAQ informal, misturou versões) | Ajuste de prompt + validação de pós-processamento no código, quando aplicável | G4, G5, N2, N5, N6 |
| Padrão não coberto por nenhum guardrail existente | Abre proposta de **guardrail novo** para revisão do Product Specialist | — |
| Reclamação de tom/estilo, sem erro factual | Backlog de melhoria contínua, não bloqueia release | — |

**Passo 3 — Verificação de fechamento.** O item de feedback só é marcado como **resolvido** — não "encaminhado", resolvido — quando todas as condições abaixo forem verdadeiras:

1. O caso virou um caso de teste permanente no golden set (Camada B da regression, seção 3);
2. Esse teste passou no regression suite antes do deploy;
3. O fix (documento/prompt/código) foi de fato deployado em produção;
4. O atendente que reportou recebeu uma confirmação mínima ("seu report X foi corrigido na versão Y" ou "identificamos que não era um erro, veja por quê").

Sem essa verificação explícita, um item pode "morrer" na triagem sem nunca virar teste — foi exatamente esse tipo de lacuna silenciosa que permitiu o I-3 original passar despercebido.

**SLA.** Feedback que reproduz um padrão já catalogado como I-1, I-2 ou I-3 é **crítico** — passa pelos três passos antes da próxima release, não vai para backlog. Feedback de padrão novo passa pelo mesmo checklist, mas com prioridade definida na triagem.

## 2. Regression testing de produto

O ponto central: regression testing aqui não é "a resposta não mudou", é **"a resposta não mudou para pior E nenhum guardrail foi violado"**. Isso exige um golden set com duas camadas:

**Camada A — casos de não-regressão.** Amostra de perguntas frequentes com respostas já validadas como corretas. Compara-se `source_document`, `source_version` e equivalência semântica da resposta (não string exata, já que o modelo varia). Qualquer queda de qualidade nessa amostra bloqueia o release.

**Camada B — casos de guardrail.** No mínimo 1-2 casos de teste por guardrail (G1-G6, N1-N6, F1-F6), incluindo os cenários originais dos incidentes I-1/I-2/I-3 como casos permanentes, **mais** os casos que chegam pelo Passo 3 do processo de feedback acima. Cada caso de teste carrega o ID do feedback que o originou, quando aplicável.

A verificação muda de acordo com a camada de implementação já definida na matriz de guardrails:

- **Guardrails de código** (G2, N1, N2, F1, F3, e a parte código de G3/G4/G5/G6/N3/N4/N5/F2/F4/F5) → teste automatizado, determinístico, **bloqueante**.
- **Guardrails de prompt puro** (N6, F6) → avaliação amostral por revisor humano ou LLM-as-judge, com threshold (ex.: ≥95% de conformidade) — abaixo disso, exige revisão manual, mas não trava o pipeline sozinho.

**Quando roda.** Antes de qualquer mudança de prompt, adição/atualização de documento, alteração de threshold de retrieval ou deploy de código do pipeline — inclusive quando o código vem do Copilot.

**Critério de aprovação do release:**
- Zero regressões na Camada A;
- 100% dos guardrails de código passando (bloqueante, sem exceção);
- ≥ threshold definido nos guardrails de prompt/ambos (dispara revisão se abaixo);
- Taxa geral de resposta incorreta não pode superar o baseline atual de 12% — meta é reduzir, nunca regredir.

Quando um documento novo entra (ex.: PROC-042-v3), o golden set relacionado ao tema é re-executado automaticamente — é assim que se evita repetir o I-2 com uma versão futura.

## 3. Pontos de human-in-the-loop

HITL tem dois momentos distintos: **antes do deploy** (mudança no assistente) e **em tempo real** (resposta individual arriscada).

**Pré-deploy**, por tipo de mudança:

| Mudança | Aprovação obrigatória? | Quem aprova |
|---|---|---|
| Prompt tocando guardrail existente (G/N/F) | Sim | Product Specialist; + Compliance se envolver documento com contradição pendente (G6/F5) |
| Novo documento normativo (política/procedimento) | Sim | Product Specialist confirma metadado de vigência; Compliance confirma que é fonte oficial |
| Novo documento informal (FAQ) | Sim, revisão leve | Product Specialist (garantir `source_type: informal` correto) |
| Threshold de relevância / lógica de retrieval | Sim | Tech Lead + Product Specialist (afeta diretamente G3/N3/F3) |
| Proposta de guardrail novo (originada do Passo 2 do processo de feedback) | Sim | Product Specialist formaliza; Compliance valida se envolve tema normativo |
| Código gerado por Copilot alterando validação, logging ou dados de atendente | Sim, sempre — **nunca merge automático** | Tech Lead (revisão técnica) + Product Specialist (conformidade com AGENTS.md e guardrails) |
| Reindexação de documento já aprovado, sem mudança de metadado | Não, mas fica logado | — |

O caso do módulo de feedback que ignorou o AGENTS.md (sem Zod, logando dado sensível do atendente) é exatamente o motivo da linha de "código gerado por Copilot": geração assistida não dispensa revisão — é o tipo de mudança que mais precisa dela, porque o guardrail pode ser violado silenciosamente sem que ninguém tenha "decidido" violá-lo.

**Runtime (por resposta individual).** O `confidence_score` do structured output é o gatilho: resposta abaixo do threshold definido em tópicos sensíveis (financeiro, segurança, elegibilidade) é roteada para um atendente humano em vez de ser entregue automaticamente ao cliente final.

## Sobre a demo em 2 semanas

Dá para ter uma "pista rápida" para a demo — regression testing focado no golden set dos 3 incidentes documentados, ambiente de staging, 5 atendentes-piloto — sem abrir mão do HITL em código sensível. O processo de feedback completo (com os três passos) já pode rodar no piloto em escala reduzida; o que precisa estar maduro para o go-live é o volume: triagem diária manual funciona com 5 atendentes, mas não escala sozinha quando o volume de feedback crescer.

---

## Turno 4

### Input (Usuário)

Com este novo documento de harness de produto e não esquecendo do seu papel como product spacialist, analise se o regression testing reconhece que mudanças em IA podem ter efeitos colaterais e verifica que os guardrails não regridem.

### Resposta (Claude)

Analisando com o mesmo rigor da vez passada: a resposta é **parcial**. O regression testing que desenhei cobre bem a ideia de "não regredir no que já testamos", mas trata efeitos colaterais de IA como se fossem localizados e determinísticos — e não são. Vou separar o que está coberto do que está faltando.

## O que já está coberto

- Guardrails de código são bloqueantes e determinísticos — isso é sólido, porque assertion em código não tem efeito colateral surpresa.
- Casos de incidente (I-1/I-2/I-3) e casos vindos do feedback (Passo 3) viram testes permanentes — isso evita que o mesmo erro reapareça de forma idêntica.

## Onde a análise de efeitos colaterais está incompleta

**1. O escopo "só o que é do mesmo tema" assume que side effects são locais — mas não são.**
Eu escrevi: *"quando um documento novo entra... o golden set relacionado ao tema é re-executado"*. Isso é o erro mais sério do documento. Em busca vetorial, indexar um documento novo pode mudar o *ranking* de chunks de temas completamente diferentes (reranking, redistribuição do espaço de embeddings). Uma mudança de prompt para corrigir G5 (hierarquia FAQ vs. normativo) é editada no mesmo system prompt que rege G1, N6, F6 — não existe "mudança isolada de tema" em um prompt compartilhado. **O regression suite completo (Camada A + Camada B inteira) precisa rodar em toda mudança**, não uma fatia temática — senão o harness está estruturalmente cego para regressão cruzada.

**2. Não existe teste para pares de guardrails que competem entre si.**
G5/N5 (priorizar normativo, sinalizar FAQ como informal) e F4 (usar FAQ quando é a única fonte disponível) empurram em direções opostas. Um ajuste de prompt que reforça G5 com mais força pode fazer o modelo passar a **recusar** usar o FAQ até quando ele é a única fonte válida — trocando o risco do I-1 (FAQ tratado como definitivo) pelo risco do I-3 (negar informação que existe). O mesmo vale para G1/N6: reforçar "verificar categoria antes de responder" pode fazer o assistente parar de encaminhar (N6) e tentar resolver tudo com perguntas de clarificação. O documento atual não define pares de tensão a serem testados juntos — só testa guardrails isoladamente.

**3. Não-determinismo do modelo não é tratado.**
Todo o desenho assume "rodou o teste, passou ou falhou" — um único run. Mas LLM não é determinístico: um guardrail de prompt pode falhar 15% das vezes e passar os outros 85%, e um único run do golden set nunca vai capturar isso. Sem rodar cada caso de guardrail em prompt (N6, F6, e a parte prompt de G3/G4/G5/G6/N3/N4/N5/F2/F4/F5) **múltiplas vezes e medir taxa de sucesso**, o regression suite pode dar "verde" em uma mudança que na verdade introduziu uma falha intermitente — que é o tipo de regressão mais perigoso porque passa despercebido até acumular volume em produção.

**4. A política de bloqueio para guardrails de prompt puro é inconsistente com o risco que eles cobrem.**
Eu defini que N6 e F6 "não travam o pipeline sozinhos, só disparam revisão". Mas F6 é descrito na própria matriz como cobertura transversal dos três incidentes — é a última linha de defesa contra alucinação quando a pergunta está fora do escopo documental. Tratar uma queda de conformidade em F6 como "não bloqueante" contradiz a própria lógica de risco do harness. Isso devia ser bloqueante, ou pelo menos ter um threshold muito mais rígido que os outros guardrails de prompt.

**5. Falta teste de paráfrase — o risco de "corrigir a string, não o comportamento".**
Um caso de teste vindo de feedback (Passo 3) tende a ser a pergunta exata que o atendente digitou. Se o fix for um ajuste de prompt genérico demais ("cite a versão do PROC-042"), ele pode passar no teste literal e ainda falhar em uma reformulação da mesma pergunta. O regression suite precisa gerar 2-3 variações (paráfrase) de cada caso de guardrail, não só guardar a frase original — senão o harness mede se o sintoma sumiu, não se a causa foi corrigida.

**6. A métrica agregada de 12% mascara regressão pontual.**
"A taxa geral de resposta incorreta não pode superar 12%" é uma métrica única, cega a onde o erro está. Uma mudança pode reduzir alucinação geral (baixando a taxa agregada, parecendo uma melhoria) e ao mesmo tempo piorar especificamente F3 (falha de retrieval tratada como ausência de informação) — que fica escondido dentro da média. O critério de aprovação precisa comparar a taxa **por guardrail/categoria de erro**, não só o agregado.

**7. Nenhum teste para fricção nova (falso positivo de fallback).**
"Guardrail não regrediu" no desenho atual só olha se o assistente parou de errar. Não olha se ele passou a acionar fallback (F1, pedir clarificação) em situações onde a categoria já estava clara na pergunta — o que não é um erro de guardrail, mas é uma degradação real de produto que o harness atual não captura.

## Resumo para ajustar o documento

| Lacuna | Ajuste necessário |
|---|---|
| Escopo temático estreito | Rodar suite completa (Camada A + B) em toda mudança, sem exceção por "tema" |
| Guardrails opostos não testados juntos | Definir pares de tensão (G5×F4, G1×N6, G3×N4) como casos combinados |
| Não-determinismo ignorado | Rodar casos de prompt N vezes (ex.: 5-10), medir taxa de sucesso, não pass/fail único |
| F6/N6 não bloqueantes | Reclassificar F6 como bloqueante dado seu papel transversal |
| Overfit à string literal | Gerar paráfrases automáticas de cada caso vindo de feedback |
| Métrica agregada esconde regressão local | Comparar taxa de erro por guardrail/categoria, não só overall |
| Fricção nova não medida | Adicionar métrica de "taxa de fallback desnecessário" ao critério de release |

Sem esses ajustes, o regression testing garante que "o que já sabemos testar continua passando" — mas não garante que a mudança não introduziu um efeito colateral em outro lugar do sistema, que é exatamente o comportamento que se espera de um regression suite maduro em produtos com IA.

---

## Turno 5

### Input (Usuário)

Realize os ajustes sugeridos acima.

### Resposta (Claude)

# Harness de Produto — Assistente de IA NovaTech

O harness parte de um princípio simples: **o assistente vai evoluir, mas os 18 guardrails do cenário 2 (G1-G6, N1-N6, F1-F6) não podem regredir nunca**. Toda mudança — documento novo, ajuste de prompt, alteração de threshold, código gerado por Copilot — passa pelos três mecanismos abaixo antes de chegar em produção.

## 1. Processo de feedback: do atendente à melhoria efetiva

**Captura.** O bot do Teams registra, junto de cada resposta, os campos estruturados (`answer`, `source_document`, `source_version`, `confidence_score`, `source_type`) e um botão 👍/👎 com campo de motivo livre.

**Passo 1 — Diagnóstico estruturado.** Checklist fixo antes de classificar: `confidence_score` da resposta, chunks efetivamente recuperados (log de retrieval), `source_type` correto, reprodutibilidade do erro.

**Passo 2 — Ação combinada, não rota única.** A causa raiz dispara uma ou mais ações em paralelo (documento + prompt + código, quando o guardrail é "Ambos"), com um ID de rastreamento único:

| Causa raiz identificada | Ações disparadas | Guardrail relacionado |
|---|---|---|
| Documento ausente/desatualizado | Reindexação + verificação de metadado de vigência | G2, G6, F2, F5 |
| Chunk errado recuperado | Ajuste de threshold/chunking (código) + reforço no prompt | G3, N3, F3 |
| Guardrail de comportamento falhou | Ajuste de prompt + validação de pós-processamento | G4, G5, N2, N5, N6 |
| Padrão não coberto por guardrail existente | Proposta de guardrail novo para revisão | — |
| Reclamação de tom/estilo sem erro factual | Backlog contínuo, não bloqueia release | — |

**Passo 3 — Verificação de fechamento.** Resolvido só quando: (1) virou caso de teste permanente no golden set — **com paráfrases**, ver seção 2.5; (2) o teste passou no regression suite; (3) o fix foi deployado; (4) o atendente recebeu confirmação.

**SLA.** Padrões já catalogados (I-1, I-2, I-3) são críticos — tratados antes da próxima release.

## 2. Regression testing de produto

A revisão anterior identificou o problema central: tratar mudanças de IA como se tivessem efeito local e determinístico. Os ajustes abaixo corrigem isso.

### 2.1 Golden set — sempre suite completa, nunca fatia temática

Camada A (não-regressão de perguntas frequentes) e Camada B (casos por guardrail + casos de incidente + casos vindos do feedback). **Toda mudança executa a suite inteira, sem exceção.** Indexar um documento novo pode reordenar o ranking de chunks de temas não relacionados (efeito de embedding/reranking); um ajuste de prompt para um guardrail é editado no mesmo system prompt que rege todos os outros. Rodar só o "golden set do tema" deixa o harness cego para regressão cruzada — foi esse o erro do desenho anterior.

### 2.2 Casos de tensão entre guardrails

Guardrails que empurram em direções opostas precisam de casos de teste combinados, não isolados:

| Par em tensão | O que testar junto |
|---|---|
| G5/N5 (priorizar normativo, sinalizar FAQ) × F4 (usar FAQ quando é única fonte) | Reforçar G5 não pode fazer o assistente recusar o FAQ quando ele é a única fonte válida (trocaria risco de I-1 por risco de I-3) |
| G1/N1 (verificar categoria antes de responder) × N6 (encaminhar, não resolver) | Reforçar a verificação não pode virar tentativa de resolver tudo via clarificação em vez de encaminhar |
| G3/N3 (retrieval falhou ≠ ausência de informação) × N4 (não inventar valores) | Corrigir falso-negativo de retrieval não pode abrir margem para o modelo preencher a lacuna com valor inventado |

Cada par em tensão vira um caso de teste específico, executado sempre que qualquer um dos dois lados do par for alterado.

### 2.3 Não-determinismo do modelo

Guardrails de prompt (N6, F6, e a parte prompt de G3/G4/G5/G6/N3/N4/N5/F2/F4/F5) não são pass/fail de um único run — um comportamento pode falhar 15% das vezes. Cada caso dessa categoria roda **N vezes** (mínimo 5, idealmente 10) por execução do suite, e o resultado é uma **taxa de sucesso**, não um booleano. Uma queda na taxa de sucesso entre releases é regressão, mesmo que nenhum run individual tenha "falhado" de forma isolada.

### 2.4 Geração de paráfrases para casos vindos de feedback

Um caso de teste originado de feedback (Passo 3) não pode ser só a frase literal que o atendente digitou — senão o fix corrige a string, não o comportamento. Toda vez que um caso é promovido ao golden set, o Product Specialist gera (ou usa um LLM para gerar, com revisão) 2-3 paráfrases da mesma pergunta. O caso só é considerado corrigido quando passa na frase original **e** nas paráfrases.

### 2.5 Critério de aprovação do release

- Zero regressões na Camada A;
- 100% dos guardrails de código passando (bloqueante);
- **F6 reclassificado como bloqueante** — por ser a cobertura transversal contra alucinação nos três incidentes, uma queda na taxa de sucesso de F6 barra o release, ao contrário dos demais guardrails de prompt;
- Demais guardrails de prompt/ambos: taxa de sucesso (seção 2.3) ≥ threshold definido (ex.: 95%) — abaixo disso, dispara revisão manual antes de liberar;
- Casos de tensão (seção 2.2): nenhuma degradação no lado oposto do par;
- **Taxa de erro por guardrail/categoria, não só agregada** — a meta de não superar 12% passa a ser medida por categoria (alucinação, documento desatualizado, chunk incorreto, falha de encaminhamento). Uma queda no agregado pode esconder piora localizada em uma categoria específica; o release é barrado se qualquer categoria piorar, mesmo com o agregado melhorando;
- **Taxa de fallback desnecessário** — mede quantas vezes o assistente acionou F1/clarificação em perguntas onde a categoria já estava clara. Um aumento aqui é degradação de produto real, mesmo sem violação de guardrail, e passa a compor o critério de release.

### 2.6 Quando roda

Antes de qualquer mudança de prompt, adição/atualização de documento, alteração de threshold/retrieval ou deploy de código do pipeline — inclusive código do Copilot. Sempre suite completa (2.1), sempre com as N execuções dos casos de prompt (2.3).

## 3. Pontos de human-in-the-loop

**Pré-deploy**, por tipo de mudança:

| Mudança | Aprovação obrigatória? | Quem aprova |
|---|---|---|
| Prompt tocando guardrail existente | Sim | Product Specialist; + Compliance se envolver contradição pendente (G6/F5) |
| Novo documento normativo | Sim | Product Specialist confirma metadado de vigência; Compliance confirma fonte oficial |
| Novo documento informal (FAQ) | Sim, revisão leve | Product Specialist (`source_type: informal` correto) |
| Threshold de relevância / retrieval | Sim | Tech Lead + Product Specialist |
| Proposta de guardrail novo | Sim | Product Specialist formaliza; Compliance valida se normativo |
| Código gerado por Copilot (validação/logging/dados de atendente) | Sim, sempre | Tech Lead + Product Specialist |
| Regression suite reprova por queda em F6 ou por categoria de erro específica (2.5) | Sim, adicional | Product Specialist decide se libera com ressalva ou bloqueia — nunca release automático nesse cenário |
| Reindexação sem mudança de metadado | Não, mas logado | — |

**Runtime.** `confidence_score` abaixo do threshold em tópicos sensíveis roteia para atendente humano em vez de resposta automática.

## Sobre a demo em 2 semanas

A suite completa (2.1) e a análise por categoria (2.5) já cabem no piloto — o custo é de tempo de execução, não de escopo reduzido. O que pode ficar para o go-live com escala maior é a automação da geração de paráfrases (2.4), que hoje depende de revisão manual do Product Specialist e não escala sozinha com volume alto de feedback.

---

## Turno 6

### Input (Usuário)

Com o último documento de harness de produto gerado, analise se o ponto de HITL é concreto (define o que precisa de aprovação humana e quem aprova).

### Resposta (Claude)

Analisando com o mesmo critério: **parcialmente concreto**. A tabela define bem *quais tipos* de mudança exigem aprovação e um nome de papel ao lado de cada linha — isso é melhor que a maioria dos desenhos de HITL que só dizem "precisa de revisão humana". Mas "definir quem aprova" não é suficiente para ser operacional; falta o "como" e o "e se não". Vou separar por tipo de lacuna.

## 1. Os gatilhos ainda dependem de julgamento não especificado

"Prompt tocando guardrail existente" — quem decide que um prompt "toca" um guardrail? Se não houver uma forma objetiva de verificar isso (ex.: um checklist de quais guardrails cada trecho do prompt cobre, ou um diff que sinaliza automaticamente quais seções do system prompt foram alteradas), a decisão de "isso precisa de aprovação ou não" fica na cabeça de quem está fazendo a mudança — exatamente o tipo de ambiguidade que already causou o incidente do módulo Copilot que ignorou o AGENTS.md sem ninguém perceber. O gatilho precisa ser verificável por outra pessoa além de quem propôs a mudança, não uma autoavaliação.

O mesmo vale para "envolver contradição pendente" (linha do prompt + Compliance) — isso depende da lista de documentos com contradição do G6/F5, que é um dado gerenciável. O gatilho devia ser automático (o pipeline já sabe se o documento tocado está nessa lista), não uma inferência humana de que "isso parece envolver contradição".

## 2. Não existe definição operacional do que é "aprovar"

A tabela diz *quem* aprova, mas não *como* essa aprovação se materializa: é um comentário aprovando um PR? Uma checkbox em um formulário? Um registro em log? Sem isso, "aprovação" não é auditável — não dá para provar depois que a aprovação de fato aconteceu antes do deploy, o que é precisamente o tipo de lacuna que interessaria a um Compliance real (e que o próprio caso do Copilot sugere que a NovaTech já tem motivo para se preocupar).

## 3. Ponto único de falha, sem substituto nem SLA

Toda a estrutura depende de "Product Specialist" e "Tech Lead" como pessoas físicas, sem:
- **Backup definido** — se o Product Specialist estiver de férias ou indisponível, quem aprova? Isso é especialmente crítico dado que há uma demo para a diretoria em 2 semanas — pressão de prazo é exatamente quando aprovações "informais" (pular a etapa "porque não deu tempo") acontecem.
- **SLA de resposta** — quanto tempo o aprovador tem para revisar antes que isso vire gargalo de release? Sem prazo, a aprovação pode travar indefinidamente ou, na prática, ser pressionada a sair rápido demais.
- **Autoridade de desempate** — se Tech Lead e Product Specialist discordarem sobre um ajuste de threshold, quem decide?

## 4. A linha de "reprovação do regression suite" delega julgamento sem critério

Escrevi: *"Product Specialist decide se libera com ressalva ou bloqueia"* para o caso de queda em F6 ou por categoria. Isso não é uma definição de aprovação — é dizer "alguém decide" sem dizer com base em quê. Que condições tornam uma "ressalva" aceitável versus um bloqueio obrigatório? Sem isso, essa linha da tabela é a menos concreta de todas, justamente na situação de maior risco (uma regressão de guardrail já detectada).

## 5. O HITL de runtime tem parâmetros não definidos

"`confidence_score` abaixo do threshold em tópicos sensíveis" tem dois vazios:
- O valor do threshold não é especificado, nem quem tem autoridade para calibrá-lo;
- "Tópicos sensíveis" não é uma lista fechada — em algum momento anterior usei "financeiro, segurança, elegibilidade" como exemplo, mas isso nunca foi formalizado como enumeração fixa e versionada. Sem lista fechada, o pipeline não tem como aplicar essa regra deterministicamente.

Além disso, falta o operacional: quando uma resposta é roteada para humano, isso acontece com os 5 atendentes-piloto atuais — o que acontece fora do horário deles, ou quando o volume crescer no go-live e não houver atendente disponível em tempo real? A tabela não cobre esse caso de borda, que é bem provável de acontecer logo na fase pós-piloto.

## 6. Não existe caminho de emergência definido

Dado o contexto (demo em 2 semanas, pressão de prazo), a ausência de um "modo emergencial" explícito é um risco prático: sem uma via formal e documentada para acelerar aprovação em caso de urgência real, a tendência natural é as pessoas pularem a etapa informalmente — o que anula o HITL exatamente quando ele mais importa.

## Resumo

| Lacuna | O que falta concretizar |
|---|---|
| Gatilho de "toca guardrail" / "envolve contradição" | Critério verificável (checklist ou detecção automática), não autoavaliação |
| Definição de "aprovação" | Artefato concreto e auditável (registro, checkbox, log) |
| Backup de aprovador | Substituto definido para cada papel |
| SLA de aprovação | Prazo máximo de resposta do aprovador |
| Autoridade de desempate | Quem decide em caso de discordância entre papéis |
| Critério de "ressalva vs. bloqueio" na reprovação do suite | Condições objetivas, não julgamento livre |
| Threshold de confidence_score e lista de tópicos sensíveis | Valores e lista fechada, versionados |
| Cobertura fora do horário / volume alto | O que acontece quando não há humano disponível para o HITL runtime |
| Via de emergência | Processo formal e documentado para acelerar aprovação sob prazo, em vez de bypass informal |

O documento acerta em identificar *categorias* de mudança e *nomes de papéis* — isso é a metade fácil. A parte que falta é o que transforma isso em processo executável: critério objetivo de gatilho, registro da decisão, prazo, substituto e o que fazer quando o caminho normal não é viável.

---

## Turno 7

### Input (Usuário)

Levando em consideração as lacunas identificadas, trabalhando como um product specialist existe a possibilidade de preencher estas lacunas?

### Resposta (Claude)

Sim, mas não todas da mesma forma — e é importante eu ser honesto sobre isso enquanto Product Specialist, porque tratar as nove lacunas como igualmente resolvíveis por mim seria repetir o mesmo erro de origem: prometer concretude que não tenho autoridade ou dado para entregar sozinho. Vou separar em três grupos.

## Grupo 1 — Posso preencher diretamente, agora, com o que já tenho

Essas dependem de artefatos que o Product Specialist já controla (a matriz de guardrails, os campos estruturados, o histórico de incidentes). Não exigem negociação com outro papel nem dado que ainda não existe:

- **Gatilho de "toca guardrail existente"** — dá para construir um mapa objetivo guardrail → seção do system prompt (a matriz já associa cada G/N/F a um comportamento específico). Qualquer diff de prompt que altere uma dessas seções mapeadas dispara o fluxo de aprovação automaticamente — deixa de ser autoavaliação de quem propôs a mudança.
- **Gatilho de "envolve contradição pendente"** — a lista de documentos com contradição já existe (os 12 sinalizados pelo Compliance, citados no G6/F5). Isso é dado de pipeline, não julgamento humano — é só formalizar a checagem automática.
- **Definição operacional de "aprovação"** — sou eu quem decide o formato (registro com campos fixos: ID da mudança, guardrail afetado, aprovador, data, decisão). Não depende de mais ninguém para ser especificado.
- **Critério de "ressalva vs. bloqueio"** — é julgamento de produto sobre risco, que é exatamente meu escopo: por exemplo, ressalva só é aceitável se o guardrail afetado não estiver na lista de bloqueantes (F6 e os guardrails de código) e o fix tiver prazo compromissado; qualquer outra situação é bloqueio automático.

## Grupo 2 — Posso rascunhar, mas a versão final não é só minha decisão

Aqui eu consigo propor um desenho concreto, mas ele só vira processo real com aval de quem carrega a responsabilidade do outro lado:

- **Backup de aprovador e SLA de resposta** — posso propor (ex.: 24-48h, substituto designado), mas isso compromete o tempo do Tech Lead e possivelmente de Compliance. Sem eles concordarem, viraria um SLA de papel que ninguém respeita na prática — o que é pior do que não ter SLA, porque cria falsa sensação de processo.
- **Autoridade de desempate entre Product Specialist e Tech Lead** — posso propor uma via de escalonamento (ex.: para liderança de produto/engenharia), mas atribuir autoridade final entre dois papéis de peso equivalente não é algo que eu decida sozinho sem criar um conflito de governança.
- **Via de emergência** — posso desenhar um processo leve (aprovação retroativa em 24h, registrada, para casos sob pressão real de prazo), mas isso precisa ser aceito por Compliance como legítimo — senão, na prática, vira exatamente o "bypass informal" que a via de emergência deveria evitar formalizar.

## Grupo 3 — Não dá para preencher com valor final ainda, só com processo de calibração

Essas dependem de dado que a NovaTech simplesmente ainda não tem — 5 atendentes-piloto e semanas de operação não geram volume suficiente para calibrar com confiança:

- **Threshold de `confidence_score`** — posso propor um valor inicial conservador (ex.: baseado no baseline de 12% de erro, começar mais restritivo e afrouxar com dado real), mas fixar o número definitivo agora seria arbitrário. O que dá para entregar concretamente é o *processo* de calibração (revisão quinzenal do threshold contra taxa de acerto observada), não o número em si.
- **Lista fechada de "tópicos sensíveis"** — a primeira versão (financeiro, segurança, elegibilidade) é um ponto de partida razoável a partir dos três incidentes conhecidos, mas fechar essa lista com segurança jurídica é uma decisão que depende de Compliance validar o que tem exposição regulatória — não é só uma escolha de produto.
- **Cobertura fora do horário / volume alto no go-live** — isso depende de decisão de staffing/operação (escala de atendentes, plantão) que está fora da minha autoridade. O que posso concretizar é o comportamento default do sistema quando não há humano disponível (ex.: recusar responder e informar canal alternativo, em vez de responder automaticamente sem HITL) — que é uma decisão de produto —, mas a solução de staffing em si não é minha para decidir.

## Resumo prático

| Grupo | Consigo preencher sozinho? | O que entrego agora |
|---|---|---|
| 1 | Sim | Critério objetivo pronto para uso imediato |
| 2 | Rascunho, não a versão final | Proposta formal para validação de Tech Lead/Compliance |
| 3 | Não com valor definitivo | Processo de calibração + comportamento default provisório |

Dado isso, o próximo passo que faz sentido é eu já fechar o Grupo 1 no documento (é trabalho que não depende de ninguém) e formalizar as propostas do Grupo 2 e 3 como itens para validação explícita com Tech Lead e Compliance — em vez de tentar apresentar o harness como 100% resolvido quando uma parte real dele depende de decisão que não é só minha.

---

## Turno 8

### Input (Usuário)

Vamos finalizar o documento de harness de produto com o que foi sugerido: fechar o grupo 1 e formalizar propstas dos grupos 2 e 3 como itens de validação explícita com Tech Lead e Compliance.

### Resposta (Claude)

*(Conteúdo idêntico ao "Turno 9 — Resposta", que é a versão final consolidada em markdown apresentada como arquivo. Ver seção completa no Turno 9 abaixo, item "Resposta (Claude)", que reproduz este mesmo documento na íntegra.)*

---

## Turno 9

### Input (Usuário)

Este último documento criado cobre orocesso de feedback, regression testing de produto e ponto de human-in-the-loop? Se sim, gere um documento markdown com o último documento harness de produto gerado.

### Resposta (Claude)

Sim — o documento cobre os três pontos pedidos:

1. **Processo de feedback** (seção 1) — captura, diagnóstico estruturado, ação combinada e verificação de fechamento até a confirmação ao atendente.
2. **Regression testing de produto** (seção 2) — suite completa, casos de tensão, não-determinismo, paráfrases e critério de aprovação por categoria.
3. **Ponto de human-in-the-loop** (seções 3 e 4) — gatilhos objetivos, definição de aprovação, tabela de aprovadores, e os itens que ainda dependem de validação formal com Tech Lead/Compliance, com default explícito enquanto isso.

[Arquivo gerado: `harness-produto-novatech.md`, contendo o documento final completo — reproduzido a seguir na íntegra, correspondente também ao conteúdo do Turno 8.]

# Harness de Produto — Assistente de IA NovaTech (versão final)

O harness parte de um princípio simples: **o assistente vai evoluir, mas os 18 guardrails do cenário 2 (G1-G6, N1-N6, F1-F6) não podem regredir nunca**. Esta versão fecha os pontos que dependiam só do Product Specialist e marca explicitamente, como **itens de validação**, o que depende de aceite de Tech Lead e Compliance — em vez de apresentar como resolvido algo que não é decisão só minha.

## 1. Processo de feedback: do atendente à melhoria efetiva

**Captura.** O bot do Teams registra, junto de cada resposta, os campos estruturados (`answer`, `source_document`, `source_version`, `confidence_score`, `source_type`) e um botão 👍/👎 com campo de motivo livre.

**Passo 1 — Diagnóstico estruturado.** Checklist fixo antes de classificar:
- Qual foi o `confidence_score` da resposta?
- Quais chunks foram efetivamente recuperados (log de retrieval)?
- O `source_type` estava correto (normativo vs. informal)?
- O erro é reprodutível?

**Passo 2 — Ação combinada, não rota única.** A causa raiz dispara uma ou mais ações em paralelo, com um ID de rastreamento único:

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

*(Em seguida, o arquivo `harness-produto-novatech.md` foi criado e compartilhado com o usuário via `present_files`.)*

---

## Turno 10

### Input (Usuário)

Agora gere um documento com todo o conteúdo desse chat: input + respostas. Caso fique muito extenso, divida em turnos.

### Resposta (Claude)

*(Este próprio documento — `registro-conversa-harness-novatech.md` — é a resposta a este turno.)*
