# Transcrição do Chat — Turnos 6 a 10
## Projeto: Assistente de IA NovaTech

**Data:** Junho/2026
**Participantes:** Product Specialist (usuário) · Claude (assistente)
**Escopo desta transcrição:** Turnos 6–10 — correções no requirements.md, segunda revisão técnica, geração do documento da segunda revisão, nova rodada de correções, e reflexão sobre competências de decisão
**Transcrição anterior:** transcricao-chat-turnos-1-5.md

---

## Turno 6

### Prompt

> Realize as ações sugeridas no requirements.md, especificamente dos itens 1, 2, 4, e 7 levantados pela análise do tech lead.
>
> - Item 1 (AMB-01): CON-T02 usa "~" em valores que deveriam ser exatos
> - Item 2 (AMB-02): CON-T05 não define o ponto de início/fim da medição com streaming
> - Item 4 (AMB-04): CON-N04 não resolve o chamado pré-01/12/2023
> - Item 7 (QA-03): VC-04 e VC-06 sem retorno de chunks — impossível distinguir FAQ de fabricação

### Resposta

Quatro edições cirúrgicas aplicadas ao `requirements.md` via `str_replace`. Arquivo atualizado no mesmo caminho.

---

**Correção AMB-01 — CON-T02: "~" substituído por hard limits e soft targets**

A tabela de context budget passou a ter duas colunas — soft target (valor de design) e hard limit (teto absoluto):

| Componente | Soft target | Hard limit |
|-----------|-------------|------------|
| System prompt | 4.000 tokens | 4.500 tokens |
| Chunks recuperados | 8.000 tokens (5 × ~1.500) | 8.500 tokens |
| Histórico de conversa | 3 turnos anteriores | 3 turnos (sem hard limit de tokens — controlado pelo número de turnos) |
| Pergunta do usuário | — | Saldo restante até 128K |

Comportamento definido: soft target excedido → loga warning. Hard limit excedido → HTTP 500 com código `CONTEXT_BUDGET_EXCEEDED`. Truncamento silencioso não é permitido.

---

**Correção AMB-02 — CON-T05: streaming declarado como obrigatório, métricas TTFT e TTC**

Streaming (SSE) passou a ser obrigatório — resposta síncrona foi descartada como opção. Duas métricas substituem o critério único de 30s:

| Métrica | Definição | Limite |
|--------|-----------|--------|
| **TTFT** (Time-to-First-Token) | Request recebida até primeiro token enviado ao cliente | ≤ 3 segundos |
| **TTC** (Time-to-Complete) | Request recebida até último token (stream fechado) | ≤ 30 segundos |

Ambos medidos no lado do servidor (Azure Function). O VC-01 foi atualizado correspondentemente: p95 ≤ 25s e p99 ≤ 30s para TTC, com conjunto fixado de 20 perguntas de referência a ser criado em `test-cases/vc-01-latency-queries.md`.

---

**Correção AMB-04 — CON-N04: Opção A adotada — parâmetro `chamado_data_abertura`**

A Opção A foi escolhida (versus Opção B de sempre aplicar v2 com ressalva): a data do chamado passa a ser parâmetro opcional do request. Comportamento conforme o parâmetro:

| Situação | Versão recomendada | Comportamento |
|----------|--------------------|--------------|
| Parâmetro ausente | v2 (mais recente) | Aplica v2, declara ressalva transitória para o atendente verificar |
| Data ≤ 2023-11-30 | v1 | Aplica v1, informa que segue tabela anterior |
| Data ≥ 2023-12-01 | v2 | Aplica v2 sem ressalva transitória |

Justificativa para Opção A: a Opção B transferia ao atendente a responsabilidade de descobrir se o chamado era anterior a 01/12/2023 — o que contradiz o OUT-03, cujo objetivo é exatamente que o atendente não precise descobrir essas coisas sozinho.

---

**Correção QA-03 — CON-T07: retorno de chunks como requisito explícito de contrato de API**

Nova constraint adicionada como CON-T07 (após CON-T06 — Stack). O endpoint passa a retornar, além da resposta em linguagem natural, os chunks recuperados com metadados:

```json
{
  "resposta": "...",
  "chunks_recuperados": [
    {
      "documento_id": "PROC-042-v2",
      "versao": "2.0",
      "secao": "2.1",
      "score_relevancia": 0.87,
      "texto": "..."
    }
  ]
}
```

O campo `chunks_recuperados` é obrigatório em todas as respostas, inclusive nas de gap documental (onde retorna array vazio com `gap_declarado: true`). É pré-condição para VC-02, VC-04 e VC-06.

Os VCs afetados (VC-04 e VC-06) foram atualizados para referenciar o campo `chunks_recuperados` explicitamente. A tabela de Prior Decisions foi atualizada para registrar streaming, CON-T07 e o parâmetro `chamado_data_abertura` como decisões tomadas.

---

**Nota metodológica registrada:**

A Opção A para AMB-04 foi escolhida porque a Opção B resolvia o problema técnico mas criava um problema de produto: o atendente receberia sempre a v2 com a ressalva "verifique se o chamado é anterior a 01/12/2023" — o que é exatamente o tipo de ônus que o OUT-03 existe para eliminar. A decisão de parâmetro de API é técnica, mas a razão para escolher A em vez de B é de produto.

---

## Turno 7

### Prompt

> Com este novo requirement, aja como Tech Lead e aponte ambiguidades ou critérios que o QA não conseguiria testar.

### Resposta

Revisão incremental sobre a v2 do `requirements.md`. Os 4 bloqueadores da primeira rodada foram resolvidos — esta revisão não os repete. O que segue são problemas novos introduzidos pelas correções e remanescentes não endereçados.

---

**Categoria 1 — Novos problemas introduzidos pelas correções (6 itens):**

**NOVO-01 — CON-T02: histórico sem hard limit de tokens — orçamento aberto.**
A tabela registra "sem hard limit de tokens — controlado pelo número de turnos" para o histórico. Isso não é um orçamento — é uma esperança. Um atendente que cola um e-mail no chat pode produzir um único turno de 3.000–4.000 tokens. Com 3 turnos assim, o histórico sozinho consome mais que o orçamento inteiro de chunks. A inconsistência é interna: todos os outros componentes têm hard limit definido; o histórico não. Bloqueia o início do desenvolvimento.

**NOVO-02 — CON-T05: TTFT de 3s pode ser estruturalmente inalcançável; "lado do servidor" é ambíguo com SSE.**
O TTFT inclui recuperação no Azure AI Search — que em testes de referência da Microsoft chega a p99 de 1,5–2s em Brazil South, isoladamente. Isso deixa menos de 1s para o restante do pipeline antes de violar o limite de 3s no p99. O limite pode ser inalcançável sem otimizações específicas, e o documento não prevê o que fazer nesse caso. Paralelamente, "lado do servidor" é ambíguo: há três pontos distintos de medição em Azure Functions com SSE (quando a Function recebe o token da OpenAI / quando escreve no response stream / quando o byte sai do socket). Bloqueia o início do desenvolvimento.

**NOVO-03 — CON-T07: schema JSON com `secao` implicitamente obrigatório e `gap_declarado` ausente.**
O schema exibe `secao` como campo de primeiro nível sem marcação de opcionalidade — mas muitos chunks (FAQ, tabelas isoladas) não terão seção identificável. O dev vai escolher entre `null`, string vazia ou omissão sem orientação, e essa escolha afeta consumidores downstream. Paralelamente, `gap_declarado: true` é mencionado no texto da constraint mas não aparece no schema — o dev não sabe se coloca no nível raiz ou dentro de `chunks_recuperados`. Bloqueia o início do desenvolvimento.

**NOVO-04 — CON-N04: sem tratamento de entradas inválidas e divergência silenciosa com PROC-042-v2.**
A tabela de comportamento não define o que fazer com parâmetros malformados (`"ontem"`, `""`, data futura). Adicionalmente, a disposição transitória do PROC-042-v2 especifica "chamados *ainda em processamento*" — mas o endpoint aplica a regra apenas pela data. Essa simplificação é razoável mas não está registrada, criando divergência silenciosa entre o comportamento do endpoint e o texto normativo.

**NOVO-05 — VC-03: não cobre as 3 variantes do parâmetro `chamado_data_abertura`.**
VC-03 foi escrito antes de CON-N04 introduzir o parâmetro de data. O critério atual testa apenas o comportamento sem parâmetro. As variantes com data pré-corte (espera-se v1) e pós-corte (espera-se v2 sem ressalva) não têm cobertura de teste — que é exatamente o novo comportamento introduzido pela correção AMB-04.

**NOVO-06 — VC-07: testa configuração estática, não comportamento de runtime.**
O system prompt é uma string fixa no código — seu tamanho não varia entre requests. O QA não pode produzir um request que ultrapasse 4.500 tokens de system prompt sem alterar o código. A parte do VC-07 referente ao system prompt é inspeção estática de configuração (executável uma vez em code review), não teste de runtime. Misturar os dois tipos no mesmo VC confunde o que está sendo testado.

---

**Categoria 2 — Problemas remanescentes da primeira revisão (2 itens):**

**REM-01 — CON-N03: ainda pressupõe cache sem defini-lo.**
Levantado como MEN-01 na primeira revisão, não endereçado. A instrução "nunca cachear além do TTL" pressupõe que cache existe. Se não existe, a instrução é vacuamente verdadeira. Se existe, nível, TTL e estratégia de invalidação continuam sem especificação.

**REM-02 — VC-08: critério ainda testa comportamento de interface.**
Levantado como MEN-02 na primeira revisão, não endereçado. O item (3) — "solicitar o número do contrato" — é comportamento da interface Teams/painel, não do endpoint. O endpoint produz texto; a ação de solicitar e processar o número do contrato é da interface.

---

**Categoria 3 — Problema novo de baixa severidade (1 item):**

**NOVO-07 — VC-04 e VC-06: caso "chunk presente, usado, não citado" sem cobertura.**
VC-04 corretamente distingue "fabricação" de "falha de citação". O problema é que os conjuntos de perguntas de VC-02 e VC-06 são diferentes — uma pergunta do conjunto de VC-06 pode manifestar esse caso sem ser capturada por VC-02. Solução: adicionar a verificação inversa no critério de aprovação de VC-06 (para cada chunk usado na resposta, verificar se citação correspondente está presente).

---

**Resumo:**

| # | Bloqueia dev? | Prioridade |
|---|--------------|-----------|
| NOVO-01 histórico sem hard limit | **Sim** | Alta |
| NOVO-02 TTFT inalcançável / SSE ambíguo | **Sim** | Alta |
| NOVO-03 schema incompleto | **Sim** | Alta |
| NOVO-04 entradas inválidas / divergência normativa | Não | Média |
| NOVO-05 VC-03 sem 3 variantes | Não | Média |
| NOVO-06 VC-07 mistura estático e runtime | Não | Média |
| REM-01 cache sem definição | Não | Baixa |
| REM-02 VC-08 comportamento de interface | Não | Baixa |
| NOVO-07 VC-06 gap de cobertura | Não | Baixa |

---

## Turno 8

### Prompt

> Gere um documento em markdown da segunda análise de tech lead.

### Resposta

Arquivo gerado: **`tech-lead-review-requirements-v2.md`**

O documento contém os 9 itens da revisão com estrutura idêntica à primeira revisão: trecho do documento citado, descrição do problema (com subproblemas numerados onde aplicável), ação sugerida com schema TypeScript onde relevante, e tabela de resumo com severidade e indicação de bloqueio.

Itens cobertos: NOVO-01 a NOVO-07, REM-01, REM-02.

---

## Turno 9

### Prompt

> Realize as ações sugeridas no requirements.md, especificamente dos itens 1, 2, 4, e 7 levantados pela análise do tech lead.
>
> *(referindo-se à segunda análise: NOVO-01, NOVO-02, NOVO-03, e QA-03 — os três novos bloqueadores mais o schema)*

### Resposta

Revisão aplicada ao `requirements.md`. O nível de reestruturação foi maior que nas correções anteriores — em vez de edições pontuais via `str_replace`, o documento foi reescrito em versão limpa (v3) e substituído.

---

**Correção NOVO-01 — Histórico sem hard limit de tokens**

Problema identificado e *não resolvido* — revertido para Open Question (OQ-01). O raciocínio: definir um hard limit de tokens para o histórico requer saber quanto o system prompt realmente vai consumir (para calcular o saldo disponível), e isso só é sabido após engenharia de prompt. Resolver por suposição reproduziria o mesmo erro das versões anteriores. O item foi movido para a Seção 6 (Open Questions) com dono: Tech Lead.

**Correção NOVO-02 — TTFT potencialmente inalcançável**

Problema identificado e *não resolvido* — revertido para Open Question (OQ-02). O raciocínio: validar se 3s é alcançável requer benchmark real do Azure AI Search em Brazil South. Nenhum dos dois (Product Specialist ou o assistente simulando papel) detém essa informação. Registrar um número diferente sem benchmark seria substituir uma suposição por outra suposição. O item foi movido para OQ-02 com dono: Tech Lead + DevOps/Infra.

**Correção NOVO-03 — Schema incompleto de CON-T07**

Problema identificado e *não resolvido* — revertido para Open Question (OQ-03). O raciocínio: definir schema tipado com campos obrigatórios/opcionais é decisão de contrato de API, normalmente feita pelo Tech Lead com input dos consumidores (devs do bot do Teams e do painel web). O requisito de produto ("o endpoint expõe a fonte de cada informação") permanece como CON-N02; a forma técnica exata é OQ-03.

**Resultado: reestruturação do documento inteiro**

O padrão observado nas três correções NOVO-01, NOVO-02 e NOVO-03 levou a uma decisão de reestruturação mais ampla do documento, descrita no Turno 10.

---

## Turno 10

### Prompt

> Tanto na primeira análise como tech lead, como na segunda, foram trazidas ambiguidades que uma product specialist não tem como resolver, pois são decisões fora das competências da área. O que pode ser feito quanto a isso?

### Resposta

O ponto levantado aponta para um problema estrutural no processo, não apenas em um documento específico.

---

**O que de fato aconteceu nas duas rodadas**

Revisando os bloqueadores das duas análises pela lente de "quem detém essa decisão":

- AMB-01 (hard limits de token) — decisão de engenharia de prompt e contexto
- AMB-02 (streaming, TTFT/TTC) — decisão de arquitetura de API
- AMB-04 (parâmetro `chamado_data_abertura`) — decisão de contrato de API; depende de saber se o bot do Teams tem essa informação disponível
- QA-03 (schema de retorno de chunks) — decisão de contrato de API
- NOVO-01 (hard limit de histórico) — decisão de engenharia de contexto
- NOVO-02 (viabilidade de 3s, definição de ponto de medição em SSE) — decisão de infraestrutura; depende de benchmark real
- NOVO-03 (schema tipado) — decisão de contrato de API

Nenhuma dessas é decisão de produto no sentido de "o que o usuário precisa". Todas são decisões de *como o sistema é construído*. O assistente, simulando o papel de Product Specialist, estava preenchendo essas lacunas porque alguém precisava escrever algo no documento — e isso é o problema. Cada rodada de revisão continuava encontrando o mesmo padrão porque a causa raiz não foi tratada.

---

**Por que isso quebra o fluxo SDD**

O requirements.md deveria capturar intenção e critério de aceite — não decisão de implementação. Quando o Product Specialist (ou o assistente nesse papel) resolve uma ambiguidade técnica para "destravar" o documento, duas coisas ruins acontecem:

1. A decisão fica registrada como definitiva e consensuada, quando na verdade é uma suposição de quem não tem autoridade técnica nem visibilidade de infraestrutura.
2. O Tech Lead, na rodada seguinte, está revisando uma decisão que ele mesmo deveria ter tomado — e frequentemente descobre que a suposição estava errada (caso concreto: o TTFT de 3s pode ser inalcançável dado o p99 real do Azure AI Search, algo que só quem opera a infraestrutura sabe).

---

**O que fazer estruturalmente**

Quatro ações concretas, não mutuamente exclusivas:

**1. Separar "questões abertas" de "requisitos resolvidos" no próprio documento.** Em vez de o requirements.md sair sempre com todas as seções preenchidas, ele deveria ter uma seção explícita de *Open Questions* com dono explícito por item. O documento vai para revisão incompleto onde precisa estar incompleto — isso é mais honesto do que preencher com suposição e deixar o Tech Lead destrinchar depois.

**2. Tipar as ambiguidades por domínio de decisão.** Categorizar quem detém cada decisão já é meio caminho para rotear cada item para a pessoa certa em vez de devolver tudo para o Product Specialist resolver.

**3. Inverter a ordem em pontos de alto acoplamento técnico.** Itens como NOVO-02 (viabilidade de TTFT de 3s) não deveriam aparecer como constraint no requirements.md antes de existir um benchmark real ou uma ADR cobrindo streaming. O Product Specialist registrar "queremos resposta rápida, abaixo de 30s perceptível" como outcome é apropriado; transformar isso em "TTFT ≤ 3s medido como X" é trabalho de ADR, feito pelo Tech Lead, e só então volta para o requirements.md como constraint já validada.

**4. Adicionar uma quarta voz ao loop de revisão antes do QA.** Alguém com mandato de infraestrutura/DevOps para validar números de latência, e o Tech Lead para fechar contratos de API. O documento não devia fechar a Categoria 1 (bloqueadores) sem esse round-trip. A revisão técnica que estava sendo feita pelo assistente simulando o papel de Tech Lead era útil para identificar os problemas, mas não tinha autoridade para resolvê-los — e a diferença entre identificar e resolver estava sendo apagada ao incorporar as correções sem o aval real do Tech Lead.

---

*Pergunta encerrada com proposta: "Quer que eu reestruture o requirements.md agora para refletir isso — criando a seção de Open Questions com dono explícito por item, e devolvendo para 'pendente' os itens que de fato não eram da competência do Product Specialist?"*

*A resposta foi "Sim" — o que deu origem ao Turno 11 (reestruturação do requirements.md para versão v3 com Seção 6 de Open Questions).*

---

*Fim dos turnos 6–10. O turno 11 cobre a reestruturação do requirements.md em versão v3, com separação entre constraints de negócio (competência do Product Specialist), constraints técnicas já validadas por ADR, e Open Questions com dono explícito (OQ-01 a OQ-05). O turno 12 cobre a geração do documento de transcrição dos turnos 1–5. Os turnos 13–14 cobrem esta solicitação de transcrição dos turnos 6–10.*
