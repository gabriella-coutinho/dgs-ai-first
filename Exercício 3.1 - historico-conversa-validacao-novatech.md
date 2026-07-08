# Histórico da Conversa — Validação do Assistente NovaTech

> Documento gerado a partir da conversa entre o usuário (Product Specialist) e o Claude, sobre a validação de respostas do assistente de IA da NovaTech em staging, com base no Anexo A (documentação simulada).

---

## Turno 1 — Usuário

**Arquivo anexado:** `anexo-a-documentacao-simulada-novatech.md`

Você é um Product Specialist participando do projeto de assistente de IA para a NovaTech.

**Cenário:**
O assistente de IA da NovaTech está em desenvolvimento. O pipeline de RAG está funcional, os primeiros endpoints foram implementados, e o bot do Teams responde perguntas de teste.

**Input:**
- Anexo: anexo-a-documentacao-simulada-novatech
- Pergunta/resposta do assistente em staging (simulados):

| # | Pergunta | Resposta | Fonte Citada | Confiança |
|---|----------|---------|--------------|-----------|
| 1 | "Qual o prazo de devolução para produtos standard?" | "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos." | POL-001, seção 3.2 | Alta |
| 2 | "Meu cliente é Silver. Qual o prazo de resolução?" | "O cliente Silver tem resolução em até 48h." | SLA-2024 | Alta |
| 3 | "Posso devolver carga perigosa classe 3?" | "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor." | POL-001, seção 3.2 | Alta |
| 4 | "Qual a política para carga danificada durante transporte?" | "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos." | Nenhuma | Alta |
| 5 | "Qual o SLA do cliente Enterprise?" | "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor." | — | Baixa |
| 6 | "Posso enviar carga perigosa com frete expresso?" | "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada." | FAQ-Atendimento, item 32 | Alta |

**Tarefa:**
- Analisar o documento em anexo;
- Analisar as perguntas/respostas do assistente em staging (simulado);
- Avaliar se a resposta dada pelo assistente foi correta, parcialmente correta, ou incorreta;
- Justificar a avaliação dada para cada pergunta/resposta.

---

## Turno 1 — Claude

Analisei o Anexo A e cruzei cada resposta simulada com as seções específicas dos documentos-fonte. Aqui está a validação:

### Avaliação por item

**1. Prazo de devolução — Standard**
**Classificação: Parcialmente correta**

O conteúdo está certo (7 dias úteis após recebimento confirmado no tracking, abertura de chamado, fotos), mas a fonte citada está errada. A seção 3.2 do POL-001 trata das exceções (cargas perigosas, refrigeradas, lacre violado) — não do prazo nem do procedimento. O prazo está na seção 3.1 e o procedimento (portal, CT-e, mínimo 3 fotos) está na 3.3. Uma citação de seção incorreta em produção é um problema sério: mina a auditabilidade da resposta e, se o time de atendimento for conferir a fonte, vai cair na seção errada.

**2. Prazo de resolução — cliente Silver**
**Classificação: Parcialmente correta**

O valor "48h" está certo, mas a resposta omite duas informações que mudam o sentido prático: (a) são 48h úteis, não 48h corridas — o relógio pausa fora do horário comercial (seção 5 do SLA-2024); (b) esse número vale para chamados gerais. Para incidentes críticos, o prazo de resolução do Silver é de 8h (não 48h). Sem essa qualificação, um cliente Silver com um incidente crítico pode achar que tem 48h quando na verdade o SLA aplicável é bem mais curto.

**3. Devolução de carga perigosa classe 3**
**Classificação: Parcialmente correta**

A regra central está correta: classes 1–6 da ANTT não são elegíveis para devolução padrão (POL-001, 3.2), e classe 3 (líquidos inflamáveis) está nessa lista. Porém a recomendação de encaminhamento está errada: a política não fala em "escalar para o supervisor", e sim em contatar Gestão de Riscos, ramal 4500 — um canal específico, não um encaminhamento genérico. Vale notar também (não como erro do assistente, mas como risco de produto) que o FAQ item 3 relata que na prática já houve exceções autorizadas por Riscos — o que reforça que a orientação certa é encaminhar ao canal certo, não "escalar" de forma vaga.

**4. Política para carga danificada em transporte**
**Classificação: Incorreta**

Este é o item mais grave da amostra. Não existe nenhum documento normativo (POL ou PROC) sobre carga danificada em trânsito — isso está explicitamente registrado nas notas de gaps do Anexo A. A única informação disponível está no FAQ-Atendimento, item 38, um documento explicitamente marcado como informal e "NÃO validado por Compliance ou Operações". O assistente:
- apresentou a informação como se fosse uma "política" oficial, sem hedge;
- deu confiança Alta citando "Fonte: Nenhuma" — uma combinação inaceitável (alta confiança sem fonte é um sinal clássico de alucinação ou de fonte não recuperada corretamente pelo RAG);
- omitiu a parte central do processo real descrito no FAQ: o caso deve ser registrado em até 48h e encaminhado ao Jurídico (sinistros@novatech.com.br), não tratado pelo atendimento comum.

Isso é exatamente o tipo de resposta que não pode ir para produção: confiante, sem citação, e divergente do processo real descrito na única fonte existente.

**5. SLA do cliente "Enterprise"**
**Classificação: Correta**

Excelente exemplo de comportamento desejado. O tier Enterprise não existe (SLA-2024 confirma só Gold, Silver e Standard), o assistente não inventou um SLA, sinalizou confiança Baixa de forma coerente com a ausência de fonte, e recomendou confirmar/escalar. É o padrão de "admitir não saber" que os outros itens deveriam seguir.

**6. Carga perigosa com frete expresso**
**Classificação: Parcialmente correta**

Aqui o assistente citou a fonte corretamente (FAQ-Atendimento, item 32) — ao contrário do item 4, isso é positivo e rastreável. O problema é a confiança Alta: essa informação vem de um documento explicitamente não validado, e nenhum documento formal (POL/PROC) rege esse processo (ver nota de contradição #4 do Anexo). Confiança Alta não é compatível com uma fonte que o próprio documento-fonte classifica como prática informal e potencialmente imprecisa. Além disso, a resposta omite o caveat prático do próprio FAQ — que a autorização leva cerca de 2 dias, ou seja, o "expresso" na prática não é expresso — informação relevante para não gerar expectativa errada no cliente.

### Resumo

| # | Pergunta (resumo) | Avaliação | Problema principal |
|---|---|---|---|
| 1 | Prazo devolução Standard | Parcialmente correta | Seção citada errada (3.2 em vez de 3.1/3.3) |
| 2 | Resolução Silver | Parcialmente correta | Falta "úteis" e falta especificar "chamados gerais" |
| 3 | Devolução classe 3 | Parcialmente correta | Encaminhamento errado ("supervisor" vs. Gestão de Riscos/ramal 4500) |
| 4 | Carga danificada | Incorreta | Alta confiança sem fonte; trata FAQ informal como política; omite rota ao Jurídico |
| 5 | SLA Enterprise | Correta | — |
| 6 | Perigosa + expresso | Parcialmente correta | Confiança Alta incompatível com fonte informal; omite caveat de prazo real |

**Padrão que se repete:** o pipeline está confundindo *ter uma fonte com algum conteúdo relacionado* com *ter uma fonte confiável e oficial*. Os itens 4 e 6 mostram que o sistema de scoring de confiança não está diferenciando documentos normativos (POL, PROC, SLA) de documentos informais (FAQ). Isso é um risco de go-live: antes de liberar, eu recomendaria que a lógica de confiança rebaixe automaticamente respostas cuja única fonte seja o FAQ-Atendimento, e que force citação de seção específica (não só o número do documento) para reduzir erros como o do item 1.

---

## Turno 2 — Usuário

Antes de pedir para que você analisasse e validasse as perguntas/respostas do assistente em relação ao anexo-a, eu fiz minha própria avaliação e incluí justificativas.

**Avaliação própria:**

Justificativa para a resposta 1: A resposta está correta. O documento POL-001 no tópico "regras de devolução" afirma que o prazo de devolução geral é de até 7 dias úteis. A minha mudança seria a de especificar de que deve ser em até 7 dias úteis, não deixar fechado de que o prazo é de exatos 7 dias úteis.

Justificativa para a resposta 2: A resposta está correta. De acordo com o documento SLA-2024, o tempo de resolução para chamados gerais de clientes silver é de até 48 horas. Além disso, o documento de FAQ corrobora essa informação.

Justificativa para a resposta 3: A resposta está parcialmente correta. O documento POL-001 afirma que cargas perigosas de classe 3 não podem ser devolvidas pelo processo padrão. Para isso, é necessário entrar em contato com a Gestão de Riscos, no ramal 4500, não escalar para um supervisor.

Justificativa para a resposta 4: A resposta está incorreta. O cliente precisa registrar o dano em até 48 horas e o reembolso integral só ocorre se houver comprovação da Novatech da negligência da transportadora. O problema aqui é que a informação é encontrada somente nas FAQ e o assistente afirma com alta confiança sem fonte oficial.

Justificativa para a resposta 5: A resposta está correta, porém com ressalvas, pois esta foi uma resposta gerada por suposição, não por ter encontrado os dados exatos no documento. A resposta do assistente foi gerada a partir da classificação de tiers no anexo e da afirmação de que não existem outros tiers, além dos informados.

Justificativa para a resposta 6: A resposta está incorreta. No documento FAQ, que não é um documento oficial, há a informação de o atendimento orienta o cliente a buscar a Gestão de Risco, no ramal 4500. Há menções sobre envio de carga perigosa no anexo A, porém não estão relacionadas com frete expresso.

Agora, realize um comparativo da sua avaliação e da minha avaliação.

---

## Turno 2 — Claude

Boa iniciativa fazer a avaliação própria antes — isso é exatamente o tipo de checagem cruzada que evita viés de "concordar com quem responde primeiro". Vamos comparar item a item.

### Visão geral

| # | Minha avaliação | Sua avaliação | Status |
|---|---|---|---|
| 1 | Parcialmente correta | Correta | Divergência |
| 2 | Parcialmente correta | Correta | Divergência |
| 3 | Parcialmente correta | Parcialmente correta | Convergência |
| 4 | Incorreta | Incorreta | Convergência |
| 5 | Correta | Correta (com ressalva) | Convergência, com nuance sua que eu não capturei |
| 6 | Parcialmente correta | Incorreta | Divergência — aqui acho que há um engano de leitura da fonte |

### Onde convergimos

**Item 3** — Concordância total. Você e eu identificamos o mesmo problema: o conteúdo normativo está certo (classe 3 não é elegível para devolução padrão), mas o encaminhamento sugerido pelo assistente ("escalar para supervisor") não corresponde ao processo real (contatar Gestão de Riscos, ramal 4500).

**Item 4** — Concordância total na classificação, e sua justificativa acrescenta um detalhe que eu tratei de forma mais genérica: você aponta explicitamente que o cliente precisa registrar o dano em até 48h, algo que eu mencionei ao falar do encaminhamento ao Jurídico, mas você foi mais direto no ponto central — a informação existe só no FAQ informal e o assistente apresentou como política oficial com alta confiança sem fonte. Mesma conclusão, ângulos complementares.

### Onde divirjo (e por quê)

**Item 1 — Prazo de devolução Standard**
Seu ponto sobre "até 7 dias úteis" vs. "7 dias úteis" (fechado) é válido e é um refinamento que eu não tinha destacado. Mas note que sua avaliação validou o conteúdo contra o tópico "regras de devolução" de forma geral — sem checar se a seção especificamente citada pelo assistente (3.2) sustenta a resposta. A 3.2 é a seção de exceções (cargas perigosas, refrigeradas, lacre violado), não a do prazo (3.1) nem a do procedimento com fotos (3.3). Então a resposta é correta em conteúdo, mas a citação de fonte está errada — e numa avaliação pré-go-live isso importa, porque é exatamente a citação que o atendente humano vai abrir para conferir.

**Item 2 — Resolução Silver**
Aqui concordo que o SLA-2024 confirma 48h para chamados gerais do Silver, e que o FAQ (item 41) corrobora. Mas dois pontos que sua avaliação não capturou:
- A tabela diz "até 48h úteis" — o relógio pausa fora do horário comercial (seção 5). O assistente falou só "48h", que pode ser lido como corridas.
- A pergunta do cliente foi genérica ("qual o prazo de resolução"), sem especificar se é chamado geral ou incidente crítico. Para incidente crítico, o Silver tem 8h, não 48h. O assistente respondeu como se só existisse um tipo de prazo, sem qualificar.

Isso não muda o fato de que o número "48h" está tecnicamente certo para o caso mais comum — mas achei que faltou precisão suficiente para ser "correta" sem ressalva.

**Item 5 — SLA Enterprise**
Sua observação é um ponto que eu não tinha explicitado: a resposta foi construída por inferência (ausência do tier na lista + a nota explícita "não existem outros tiers") e não por recuperação direta de um fato documentado. Isso é uma nuance importante que reforça por que a confiança Baixa foi a escolha certa — o assistente não tinha uma frase pronta dizendo "Enterprise não existe", ele deduziu isso corretamente. Concordo com sua leitura mais refinada aqui; eu tinha classificado como "Correta" sem essa camada extra de análise sobre o mecanismo da resposta.

**Item 6 — Carga perigosa + frete expresso**
Este é o ponto onde preciso te chamar atenção para algo específico no documento, porque acho que houve uma mistura de duas entradas diferentes do FAQ.

Você escreveu que o FAQ menciona a orientação de buscar a Gestão de Riscos (ramal 4500) e que "há menções sobre envio de carga perigosa no anexo A, porém não estão relacionadas com frete expresso." Mas isso é o conteúdo do item 3 do FAQ (que trata de devolução de carga perigosa). A pergunta simulada nº 6, no entanto, é sobre envio com frete expresso — e existe uma entrada específica para isso: o item 32 do FAQ-Atendimento, que pergunta literalmente "Pode enviar carga perigosa com frete expresso?" e responde que sim, mediante autorização do Compliance e documentação ANTT atualizada, com a ressalva de que a autorização leva cerca de 2 dias.

Ou seja: a citação do assistente (FAQ-Atendimento, item 32) está correta e é diretamente pertinente à pergunta — não é uma fonte desconectada. O problema real, na minha leitura, não é que a informação "não existe" ou "não é relacionada", mas sim:
1. A fonte é o FAQ informal (explicitamente não validado por Compliance/Operações), então confiança Alta é inadequada — deveria ser Média ou Baixa com uma ressalva.
2. A resposta omite o caveat prático do próprio FAQ: a autorização leva ~2 dias, então o "expresso" na prática não é expresso — informação que evita que o cliente seja mal informado sobre prazo.

Por isso classifiquei como "parcialmente correta" e não "incorreta": o conteúdo central (sim, pode, com autorização) está sustentado por uma fonte real e citada corretamente, mas a confiança e a completude da resposta é que falham. Vale você reconferir o item 32 do FAQ para ver se concorda com essa leitura.

### Síntese

No geral, nossas avaliações convergem na direção (nenhum dos dois deixou passar o item 4, que é o mais grave), mas a diferença de profundidade aparece em dois eixos: fidelidade da citação (item 1) e checagem de qual trecho específico da fonte realmente responde à pergunta (item 6). São exatamente os dois tipos de erro mais perigosos num RAG corporativo — citar a seção errada e citar a fonte certa só que ler mal o que ela diz — então vale considerar incluir os dois como critérios formais no checklist de QA antes do go-live.

---

## Turno 3 — Usuário

Gere um documento em markdown SOMENTE com a sua avaliação. Depois gere um OUTRO documento separado com a comparação entre a sua avaliação e a minha.

---

## Turno 3 — Claude

*(Gerou dois arquivos markdown: `avaliacao-assistente-novatech.md`, contendo apenas a avaliação do Claude, e `comparativo-avaliacoes-novatech.md`, contendo o comparativo entre as duas avaliações. Conteúdo integral reproduzido nos turnos 1 e 2 acima.)*

Dois documentos gerados: a avaliação isolada e o comparativo. Vale destacar que, no comparativo, o item 6 ficou marcado como ponto de atenção para você reconferir — a divergência ali depende de qual item do FAQ-Atendimento cada avaliação teve em mente (item 3 vs. item 32).

---

## Turno 4 — Usuário

Agora gere um documento em markdown com o conteúdo deste chat: meus inputs e suas respostas.

---

## Turno 4 — Claude

*(Este documento.)*
