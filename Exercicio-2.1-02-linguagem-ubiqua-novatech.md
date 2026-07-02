# Linguagem Ubíqua — Assistente de IA NovaTech

**Projeto:** Assistente de IA NovaTech  
**Fase:** Recorte de domínio  
**Responsável:** Product Specialist  
**Data:** Junho/2026  
**Status:** Rascunho para revisão do time  
**Fonte:** Documentação normativa — POL-001 v3.1, PROC-042 v1.0, PROC-042 v2.0, SLA-2024 v2024.1, FAQ-Atendimento (informal)

---

## Como usar este documento

Este glossário define os termos que **humanos e agentes** devem usar de forma idêntica em todo o projeto — no system prompt, nos chunks do RAG, nas respostas do assistente, nos critérios de avaliação do QA e nas conversas do time.

Cada entrada tem:
- **Definição NovaTech:** o que o termo significa neste domínio
- **Risco de interpretação pelo LLM:** por que um modelo sem essa definição erraria
- **Bounded context:** onde o termo vive
- **Não confundir com:** termos relacionados que o modelo poderia misturar

Os termos estão organizados por risco de interpretação, do mais crítico ao menos crítico.

---

## Grupo 1 — Termos com risco crítico de interpretação

*Estes termos têm definição formal na NovaTech que diverge do uso comum ou do uso em outros domínios. Uma interpretação genérica produz respostas erradas com impacto contratual ou financeiro direto.*

---

### Dias úteis

**Definição NovaTech:** Dias de segunda a sexta-feira, excluindo sábados, domingos e **feriados nacionais**. Feriados estaduais e municipais não são mencionados nos documentos e, portanto, não integram a definição.

**Risco de interpretação pelo LLM:** O modelo pode considerar apenas sábado e domingo, ignorando feriados. Ou pode incluir feriados estaduais/municipais por inferência. Ambos os erros afetam diretamente o cálculo de prazos contratuais.

**Onde aparece:** Prazo de devolução (7 dias úteis — POL-001 §3.1); triagem de chamado (4 horas úteis — POL-001 §3.3); coleta reversa (2 dias úteis — POL-001 §3.3); reembolso (5 dias úteis — POL-001 §3.3); SLAs de chamados gerais (SLA-2024 §2); prazo adicional de frete especial (PROC-042).

**Contextos:** BC-02 Devolução, BC-03 Prazos, BC-04 SLA e Atendimento

**Não confundir com:** "Horas úteis" — subconjunto do mesmo conceito, mas aplicado a métricas intradiárias como o SLA de triagem (4 horas úteis) e de primeira resposta.

---

### Incidente crítico

**Definição NovaTech:** Chamado que atende a **pelo menos um** dos seguintes critérios (SLA-2024 §3):
1. Carga com valor declarado acima de R$ 100.000 com status desconhecido há mais de 6 horas
2. Carga perigosa com qualquer irregularidade de documentação ou rastreamento
3. Mais de 5 chamados do mesmo cliente nas últimas 24 horas sobre o mesmo problema
4. Qualquer situação que envolva risco à segurança de pessoas

A classificação como incidente crítico altera os SLAs aplicáveis e, para clientes Gold, suspende a pausa do relógio de SLA fora do horário comercial.

**Risco de interpretação pelo LLM:** O modelo pode usar o conceito ITIL de "incidente" (qualquer interrupção não planejada) ou o sentido coloquial (situação grave sem critério formal). Os 4 critérios acima são taxativos — o modelo precisa checá-los, não inferir gravidade subjetivamente.

**Contextos:** BC-04 SLA e Atendimento, BC-03 Prazos

**Não confundir com:** "Chamado de alta prioridade" — expressão do FAQ (item 27) que não tem definição formal nos documentos normativos e não equivale a incidente crítico.

---

### Devolução (processo padrão)

**Definição NovaTech:** Processo formalizado na POL-001 para retorno de mercadoria **já entregue** ao destinatário, dentro do prazo de 7 dias úteis, aberto via Portal do Cliente com CT-e e documentação fotográfica. Cobre apenas mercadorias elegíveis — cargas perigosas, refrigeradas com ruptura de cadeia de frio e com lacre violado sem documentação ficam **fora** do processo padrão.

**Risco de interpretação pelo LLM:** O modelo pode:
- Tratar "devolução" como sinônimo de reembolso (são etapas distintas)
- Misturar com o processo de carga em trânsito (cobre apenas pós-entrega)
- Misturar com o processo de carga danificada (fluxo diferente, via sinistros@novatech.com.br)
- Generalizar para um processo de devolução de e-commerce, ignorando a exigência do CT-e

**Contextos:** BC-02 Devolução

**Não confundir com:** Interceptação de carga (PROC-088 — carga ainda em trânsito); sinistro de carga danificada (FAQ item 38 — processo via Jurídico, sem documento formal); coleta reversa (etapa dentro do processo de devolução, não sinônimo).

---

### Frete especial

**Definição NovaTech:** Modalidade de frete aplicável a cargas com peso **acima de 500 kg**, calculada pela fórmula `Valor base × Multiplicador regional × Fator de peso`. Distinto do frete padrão (abaixo de 500 kg), cujas regras não estão documentadas na base disponível.

**Risco de interpretação pelo LLM:** O modelo pode:
- Interpretar "especial" como urgente, prioritário ou premium — não é; é simplesmente a faixa de peso acima de 500 kg
- Usar os multiplicadores de uma das duas versões do PROC-042 sem sinalizar o conflito
- Calcular o prazo usando +2 dias (v1) quando deveria usar +3 dias (v2), ou vice-versa
- Responder sobre frete padrão usando as regras do frete especial por ausência de outra referência

**Contextos:** BC-01 Frete

**Não confundir com:** Frete padrão (abaixo de 500 kg — regras não disponíveis na base); frete expresso (mencionado no FAQ item 32 como possível para cargas perigosas, sem documento formal).

---

### Carga perigosa

**Definição NovaTech:** Carga classificada nas **classes 1 a 6 da ANTT**, conforme Resolução ANTT nº 5.947/2021:
- Classe 1: Explosivos
- Classe 2: Gases
- Classe 3: Líquidos inflamáveis
- Classe 4: Sólidos inflamáveis
- Classe 5: Oxidantes e peróxidos
- Classe 6: Substâncias tóxicas e infectantes

A classificação ANTT é o critério definitivo — não o julgamento do atendente ou do cliente sobre o que é "perigoso".

**Risco de interpretação pelo LLM:** O modelo pode usar uma definição popular ou baseada em senso comum ("coisas que explodem ou correm"), ignorando que a classe da ANTT é o critério formal. Isso afeta a elegibilidade para devolução padrão, os SLAs de incidente crítico e o cálculo de frete (PROC-043, não disponível).

**Contextos:** BC-02 Devolução, BC-01 Frete

**Não confundir com:** Carga de alto valor (critério diferente, usado para classificar incidente crítico); carga refrigerada (categoria separada com suas próprias regras de devolução).

---

### Tier de cliente

**Definição NovaTech:** Classificação contratual do cliente em três níveis — **Gold**, **Silver** ou **Standard** — com base em volume mensal de operações ou valor anual de contrato:
- Gold: contrato anual > R$ 500.000 **OU** > 200 operações/mês
- Silver: contrato anual entre R$ 100.000 e R$ 500.000 **OU** entre 50 e 200 operações/mês
- Standard: todos os demais

**Não existe tier Platinum** nem qualquer outro além dos três acima. O programa de fidelidade anterior foi descontinuado em 2022.

**Risco de interpretação pelo LLM:** O modelo pode:
- Aceitar que um cliente seja "Platinum" sem questionar
- Inferir um tier pelo contexto da conversa sem verificar o contrato
- Confundir tier com prioridade de atendimento (tier define SLA contratual; prioridade de atendimento em um chamado específico é determinada pela classificação do incidente)

**Contextos:** BC-04 SLA e Atendimento

**Não confundir com:** Prioridade do chamado (determinada pelo tipo de incidente, não apenas pelo tier); gerente de conta dedicado (benefício exclusivo do tier Gold).

---

### SLA de resposta vs. SLA de resolução

**Definição NovaTech (SLA-2024 §2 + FAQ item 41):**
- **Primeira resposta:** momento em que o time de atendimento dá qualquer retorno ao cliente sobre o chamado — mesmo que seja "estamos verificando". O relógio para aqui para a métrica de resposta.
- **Resolução:** momento em que o problema é efetivamente resolvido e o chamado é encerrado. O relógio para aqui para a métrica de resolução.

São dois SLAs independentes. O cumprimento de um não implica o cumprimento do outro.

**Risco de interpretação pelo LLM:** O modelo pode usar os dois termos como sinônimos ou apresentar apenas um deles ao explicar os SLAs de um tier. O FAQ item 41 é o único lugar que explica a distinção em linguagem simples — mas é fonte informal.

**Contextos:** BC-04 SLA e Atendimento

**Não confundir com:** Triagem (etapa interna do processo de devolução — 4 horas úteis — que não é um SLA contratual de atendimento ao cliente, mas um prazo interno de operação).

---

### Relógio de SLA (pausa e não-pausa)

**Definição NovaTech (SLA-2024 §5):** O relógio de SLA que mede os prazos de resposta e resolução **pausa fora do horário comercial** (08h–18h, dias úteis) para chamados gerais de qualquer tier. **Não pausa** para incidentes críticos de clientes Gold.

**Risco de interpretação pelo LLM:** O modelo pode apresentar os SLAs como prazos corridos (24h de corrido, não 24h úteis), ou aplicar a regra de não-pausa de Gold para todos os tiers, ou não mencionar a distinção quando ela é relevante para a pergunta do atendente.

**Contextos:** BC-04 SLA e Atendimento

**Não confundir com:** Dias úteis em geral — o relógio de SLA tem lógica própria (hora comercial), diferente do cálculo de prazo em dias úteis usado em devolução e coleta reversa.

---

## Grupo 2 — Termos técnicos e regulatórios

*Termos com definição técnica ou regulatória que o LLM pode reconhecer parcialmente mas sem precisão suficiente para o contexto da NovaTech.*

---

### CT-e (Conhecimento de Transporte Eletrônico)

**Definição NovaTech:** Documento fiscal eletrônico obrigatório para transporte de cargas no Brasil, emitido pela NovaTech. É o identificador primário de uma operação de transporte. **Sem o número do CT-e, o processo de devolução não pode ser aberto** (POL-001 §3.3).

**Risco de interpretação pelo LLM:** O modelo pode tratar CT-e como um número de pedido ou nota fiscal qualquer, ou não entender por que ele é exigido no processo de devolução.

**Contextos:** BC-02 Devolução

---

### Cadeia de frio (ruptura de)

**Definição NovaTech (POL-001 §3.2):** A cadeia de frio é considerada **rompida** quando a temperatura da carga refrigerada fica fora da faixa especificada na nota fiscal por **mais de 30 minutos contínuos**, conforme registro do sensor IoT embarcado. Carga com ruptura confirmada de cadeia de frio **não é elegível** para devolução pelo processo padrão.

**Risco de interpretação pelo LLM:** O modelo pode interpretar "ruptura de cadeia de frio" como qualquer variação de temperatura, sem considerar o critério de tempo (30 min contínuos) e a evidência exigida (sensor IoT). Isso afeta diretamente a elegibilidade para devolução.

**Contextos:** BC-02 Devolução

---

### Lacre de segurança violado

**Definição NovaTech (POL-001 §3.2):** Lacre físico de segurança da embalagem que foi aberto ou danificado. **Exceção ao processo padrão:** se a violação foi documentada no ato da entrega com assinatura do motorista e do recebedor, a carga **pode** ser elegível para devolução padrão. Sem essa documentação, é inelegível e vai para Gestão de Riscos.

**Risco de interpretação pelo LLM:** O modelo pode tratar lacre violado sempre como inelegível, ignorando a exceção da documentação assinada no ato da entrega — o que prejudicaria um cliente que seguiu o procedimento correto.

**Contextos:** BC-02 Devolução

---

### Coleta reversa

**Definição NovaTech:** Operação logística de retirada da mercadoria no endereço do cliente para retorno ao centro de distribuição da NovaTech, agendada em até **2 dias úteis após a aprovação da devolução**. É uma etapa do processo de devolução, não sinônimo de devolução.

**Risco de interpretação pelo LLM:** O modelo pode usar "coleta reversa" e "devolução" como sinônimos, perdendo a distinção de que a coleta é uma etapa posterior à aprovação da elegibilidade.

**Contextos:** BC-02 Devolução

---

### Multiplicador regional

**Definição NovaTech:** Fator numérico aplicado ao valor base do frete especial conforme a região de destino da carga. **Existem duas tabelas vigentes sem hierarquia formal** (PROC-042 v1 e v2). O contexto de qual versão aplicar depende da data de abertura do chamado (antes ou depois de 01/12/2023, conforme disposição transitória da v2).

**Risco de interpretação pelo LLM:** O modelo pode apresentar apenas um conjunto de multiplicadores sem mencionar a ambiguidade, levando o atendente a cobrar o cliente com o valor errado.

**Contextos:** BC-01 Frete

---

### Fator de peso

**Definição NovaTech:** Multiplicador adicional aplicado ao cálculo do frete especial baseado na faixa de peso da carga. Assim como os multiplicadores regionais, **existem dois conjuntos de valores** (v1 e v2 do PROC-042):

| Faixa | PROC-042 v1 | PROC-042 v2 |
|-------|-------------|-------------|
| 500 kg a 1.000 kg | 1,0 | 1,0 |
| 1.001 kg a 3.000 kg | 1,2 | 1,15 |
| Acima de 3.000 kg | 1,5 | 1,4 |

**Risco de interpretação pelo LLM:** Idem ao multiplicador regional — apresentar apenas uma versão sem sinalizar o conflito.

**Contextos:** BC-01 Frete

---

### Valor base

**Definição NovaTech:** Tarifa publicada na tabela mensal de fretes, disponível internamente em `\\novatech-fs\comercial\tabelas\frete-base-AAAAMM.xlsx`. **Não está disponível na base documental do assistente.** O assistente sabe que ela existe e é necessária para o cálculo, mas não tem acesso ao valor.

**Risco de interpretação pelo LLM:** O modelo pode tentar estimar ou inventar um valor base para completar um cálculo de frete, quando a resposta correta é informar que o valor base precisa ser consultado na tabela mensal.

**Contextos:** BC-01 Frete

---

### Operação (contagem para tier)

**Definição NovaTech:** Unidade de medida para classificação de tier. Uma "operação" equivale a uma entrega/transporte registrado no sistema. O critério de tier usa "operações/mês" — não número de chamados, não número de CT-es de devolução, não número de itens transportados.

**Risco de interpretação pelo LLM:** O modelo pode interpretar "operação" de forma genérica (qualquer interação com a NovaTech, incluindo chamados de suporte), inflando a contagem e classificando incorretamente o tier de um cliente.

**Contextos:** BC-04 SLA e Atendimento

---

### Triagem (de chamado de devolução)

**Definição NovaTech (POL-001 §3.3):** Etapa interna do processo de devolução em que o time de atendimento verifica elegibilidade, documentação e prazo do chamado aberto pelo cliente. Prazo: **4 horas úteis** a partir da abertura do chamado. É um prazo operacional interno — não é um SLA contratual de atendimento ao cliente.

**Risco de interpretação pelo LLM:** O modelo pode confundir o prazo de triagem (4h úteis, POL-001) com o SLA de primeira resposta (varia por tier, SLA-2024), respondendo que "o cliente receberá resposta em 4 horas" independentemente do seu tier — o que é errado para Gold (2h) e correto apenas coincidentemente para Silver.

**Contextos:** BC-02 Devolução, BC-04 SLA e Atendimento

---

## Grupo 3 — Termos com risco de escopo incorreto

*Termos que um LLM usaria corretamente em sentido geral, mas cujo escopo na NovaTech é mais restrito ou diferente do esperado.*

---

### Devolução parcial

**Definição NovaTech (POL-001 §3.4):** Devolução de um ou mais volumes individuais de uma entrega que envolveu múltiplos volumes. Segue o mesmo procedimento da devolução total. O reembolso é proporcional ao peso/valor do volume devolvido, conforme o CT-e.

**Risco de interpretação pelo LLM:** O modelo pode tratar devolução parcial como devolução de parte de um item (ex: metade de um produto), quando na NovaTech o conceito é aplicado a volumes físicos de uma entrega multi-volume.

**Contextos:** BC-02 Devolução

---

### Frete reverso

**Definição NovaTech:** Frete da coleta reversa no processo de devolução por desistência do cliente. Calculado com os **mesmos multiplicadores do frete original** (POL-001 §3.5). Não tem tabela própria — é derivado do cálculo de frete especial aplicado ao transporte original.

**Risco de interpretação pelo LLM:** O modelo pode apresentar o frete reverso como uma tarifa diferente ou nova, quando na verdade é o mesmo cálculo aplicado ao percurso inverso.

**Contextos:** BC-02 Devolução, BC-01 Frete

---

### Chamado

**Definição NovaTech:** Registro formal de solicitação ou problema aberto no Portal do Cliente (portal.novatech.com.br). É a unidade de trabalho do atendimento. O relógio de SLA começa no **timestamp de abertura do chamado** no sistema (Azure DevOps). Não inclui contatos por telefone ou e-mail que não gerem abertura formal de chamado.

**Risco de interpretação pelo LLM:** O modelo pode usar "chamado" como sinônimo de "contato" ou "solicitação verbal", quando o início do SLA depende da abertura formal no portal.

**Contextos:** BC-04 SLA e Atendimento, BC-02 Devolução

---

### Reembolso vs. crédito

**Definição NovaTech (POL-001 §3.3):** O resultado financeiro de uma devolução aprovada pode ser **reembolso** (devolução do valor em dinheiro) ou **crédito** (crédito para uso em fretes futuros). Os documentos disponíveis não especificam quando se aplica cada modalidade — o texto usa "reembolso ou crédito" sem critério de decisão documentado.

**Risco de interpretação pelo LLM:** O modelo pode afirmar categoricamente que o cliente receberá reembolso em dinheiro, quando a política deixa as duas modalidades em aberto.

**Contextos:** BC-02 Devolução

---

### Prazo expirado

**Definição NovaTech (POL-001 §3.5):** Solicitação de devolução feita após o prazo de 7 dias úteis do recebimento. Não é elegível para o processo padrão de devolução. Deve ser encaminhada ao Comercial para negociação caso a caso — não é automaticamente negada.

**Risco de interpretação pelo LLM:** O modelo pode dizer que o cliente "não pode mais devolver" quando o correto é que o processo padrão não se aplica, mas existe uma via de negociação.

**Contextos:** BC-02 Devolução

---

## Grupo 4 — Termos que não existem no domínio NovaTech

*Termos que um cliente ou atendente pode usar, mas que não têm correspondente na documentação normativa. O assistente deve reconhecê-los e corrigir, não aceitar.*

---

### Tier Platinum

**Status:** Não existe na NovaTech. O programa de fidelidade antigo (que usava esse nome) foi descontinuado em 2022. Os tiers vigentes são Gold, Silver e Standard.

**Comportamento esperado do assistente:** Informar que o tier Platinum não existe, corrigir sem invalidar o cliente, solicitar o número do contrato para identificar o tier correto.

---

### Frete expresso (para cargas perigosas)

**Status:** Mencionado no FAQ item 32 ("sim, mas precisa de autorização do Compliance"), mas **não existe documento formal** (POL nem PROC) que defina esse processo. O assistente não deve apresentar isso como política estabelecida.

**Comportamento esperado do assistente:** Informar que não há política formal documentada, citar a fonte informal (FAQ) com a ressalva adequada, recomendar consulta ao Compliance.

---

### Frete padrão (abaixo de 500 kg)

**Status:** Existe operacionalmente, mas **não há documento na base disponível** que defina suas regras. O assistente não deve inferir as regras do frete padrão a partir do frete especial.

**Comportamento esperado do assistente:** Informar que as regras do frete padrão não estão disponíveis na base de conhecimento e encaminhar para o time Comercial.

---

## Resumo: termos por bounded context

| Bounded Context | Termos principais |
|----------------|-------------------|
| BC-01 — Frete | Frete especial, Frete padrão, Valor base, Multiplicador regional, Fator de peso, Frete reverso, Desconto de volume |
| BC-02 — Devolução | Devolução (processo padrão), Devolução parcial, Coleta reversa, Triagem, CT-e, Cadeia de frio (ruptura), Lacre violado, Prazo expirado, Reembolso vs. crédito |
| BC-03 — Prazos | Dias úteis, Horas úteis, Prazo adicional do frete especial |
| BC-04 — SLA e Atendimento | Tier, Incidente crítico, SLA de resposta, SLA de resolução, Relógio de SLA, Chamado, Operação (contagem) |
| Todos | Dias úteis, Chamado |

---

## Termos a incluir no system prompt do assistente

Os termos abaixo devem ter suas definições explicitadas no system prompt — não apenas no glossário de referência — porque são os mais propensos a causar erros em respostas sem supervisão humana:

1. **Incidente crítico** — definição com os 4 critérios taxativos
2. **Dias úteis / horas úteis** — com a regra de feriados nacionais
3. **Relógio de SLA** — com a distinção pausa/não-pausa por tier e tipo de chamado
4. **Frete especial** — com instrução explícita de sinalizar o conflito documental entre v1 e v2
5. **Devolução (processo padrão)** — com os limites do escopo (pós-entrega, não carga danificada, não carga em trânsito)
6. **Tier** — com a lista fechada (Gold, Silver, Standard) e instrução para nunca aceitar Platinum
7. **Triagem vs. SLA de resposta** — com a instrução de não confundir os dois prazos
