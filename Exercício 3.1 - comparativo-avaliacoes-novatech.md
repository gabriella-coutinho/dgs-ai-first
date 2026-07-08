# Comparativo de Avaliações — Assistente NovaTech (Staging)

Este documento compara a avaliação própria do Product Specialist (feita antes da análise do Claude) com a avaliação gerada pelo Claude, ambas baseadas no Anexo A.

---

## Quadro-resumo

| # | Avaliação Claude | Avaliação Product Specialist | Status |
|---|---|---|---|
| 1 | Parcialmente correta | Correta | Divergência |
| 2 | Parcialmente correta | Correta | Divergência |
| 3 | Parcialmente correta | Parcialmente correta | Convergência |
| 4 | Incorreta | Incorreta | Convergência |
| 5 | Correta | Correta (com ressalva) | Convergência, com nuance adicional do PS |
| 6 | Parcialmente correta | Incorreta | Divergência |

---

## Pontos de convergência

### Item 3 — Devolução de carga perigosa classe 3
Concordância total. Ambas as avaliações identificaram o mesmo problema: o conteúdo normativo está correto (classe 3 não é elegível para devolução padrão), mas o encaminhamento sugerido pelo assistente ("escalar para supervisor") não corresponde ao processo real, que é contatar a Gestão de Riscos pelo ramal 4500.

### Item 4 — Carga danificada em transporte
Concordância total na classificação (Incorreta). A justificativa do PS destaca o prazo de 48h para registro e a condição de negligência comprovada para reembolso integral; a avaliação do Claude enfatiza a combinação problemática de alta confiança sem fonte oficial e a ausência de menção ao encaminhamento real ao Jurídico. As duas leituras chegam à mesma conclusão por ângulos complementares: a informação só existe em fonte informal e foi apresentada como se fosse política validada.

### Item 5 — SLA do cliente Enterprise
Concordância na classificação, com uma nuance importante trazida pelo PS que o Claude não havia explicitado: a resposta foi construída por **inferência** (ausência do tier na lista de tiers documentados + a nota explícita de que não existem outros tiers), e não por recuperação direta de uma frase pronta no documento. Essa observação reforça por que a confiança Baixa é a escolha correta e adiciona uma camada de análise sobre o *mecanismo* de geração da resposta que vale ser incorporada à avaliação final.

---

## Pontos de divergência

### Item 1 — Prazo de devolução Standard
- **PS:** Correta, com sugestão de ajuste de fraseado ("até 7 dias úteis" em vez de "7 dias úteis" fechado).
- **Claude:** Parcialmente correta.

A avaliação do PS validou o conteúdo contra o tópico geral de "regras de devolução", sem verificar se a seção especificamente citada pelo assistente (3.2) sustenta a resposta. A seção 3.2 do POL-001 trata das exceções ao prazo (cargas perigosas, refrigeradas, lacre violado) — não do prazo em si, que está na seção 3.1, nem do procedimento com fotos, que está na 3.3. O conteúdo da resposta está correto, mas a citação de fonte não corresponde ao trecho real, o que compromete a rastreabilidade da resposta em um cenário de auditoria ou conferência pelo atendente.

### Item 2 — Prazo de resolução Silver
- **PS:** Correta, apontando que o SLA-2024 confirma 48h para chamados gerais do Silver e que o FAQ corrobora a informação.
- **Claude:** Parcialmente correta.

Concorda-se que o número 48h está correto para chamados gerais. A divergência está em dois pontos não capturados pela avaliação do PS: (a) a tabela especifica "até 48h **úteis**", e o relógio de SLA pausa fora do horário comercial — a resposta do assistente omite a unidade "úteis"; (b) a pergunta original do cliente foi genérica, sem indicar se tratava de chamado geral ou incidente crítico, e para incidente crítico o prazo de resolução do Silver é de 8h, não 48h. A ausência dessa qualificação foi o motivo da classificação como parcialmente correta.

### Item 6 — Carga perigosa com frete expresso
- **PS:** Incorreta, com a justificativa de que o FAQ menciona apenas a orientação de buscar a Gestão de Riscos (ramal 4500) para carga perigosa, e que as menções sobre envio de carga perigosa no anexo não estão relacionadas a frete expresso.
- **Claude:** Parcialmente correta.

Esta é a divergência mais relevante para revisão. A justificativa do PS parece referir-se ao **item 3** do FAQ-Atendimento, que trata da *devolução* de carga perigosa e de fato orienta contato com a Gestão de Riscos. A pergunta simulada nº 6, no entanto, é sobre *envio* de carga perigosa com frete expresso — e existe uma entrada específica e diretamente pertinente para esse cenário: o **item 32** do FAQ-Atendimento, que responde afirmativamente, mediante autorização do Compliance e documentação ANTT atualizada, com a ressalva de que a autorização leva cerca de 2 dias.

Nessa leitura, a citação do assistente (FAQ-Atendimento, item 32) está correta e relacionada ao tema da pergunta — não é uma fonte desconectada. O problema não é a inexistência de relação entre a fonte e a pergunta, mas sim:
1. a fonte é um documento informal, não validado por Compliance ou Operações, o que torna a confiança Alta atribuída pelo assistente inadequada;
2. a resposta omite o caveat prático do próprio FAQ sobre o prazo real da autorização, o que pode gerar expectativa incorreta no cliente.

Recomenda-se reconferir o item 32 do FAQ-Atendimento para validar qual das duas leituras — a do PS ou a do Claude — reflete corretamente a fonte, já que isso muda a classificação de "Incorreta" para "Parcialmente correta".

---

## Síntese

As duas avaliações convergem na direção geral e, mais importante, coincidem no item mais crítico da amostra (item 4), o que reforça a confiabilidade da validação cruzada. As divergências se concentram em dois eixos recorrentes:

1. **Fidelidade da citação** (item 1): validar se a seção específica citada pelo assistente realmente sustenta o conteúdo da resposta, e não apenas se o conteúdo está correto em algum lugar do documento.
2. **Leitura do trecho exato da fonte** (item 6): ao avaliar uma citação, é preciso localizar e ler o item/seção específico citado, não apenas confirmar que o tema geral (carga perigosa) aparece no documento.

Ambos os eixos são relevantes para o checklist de QA pré-go-live, já que representam os dois modos de falha mais perigosos em um pipeline de RAG: citar a fonte errada e citar a fonte certa, mas interpretar mal o que ela diz.
