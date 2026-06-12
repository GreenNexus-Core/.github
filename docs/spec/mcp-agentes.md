# MCP e Agentes no Processo de Due Diligence — Análise de Arquitetura

> Versão 1.0 — 2026-06-12 · Companion de `especificacao-tecnica.md`

## 1. Tese

**O pipeline canônico não vira agente. Agentes operam acima e nas bordas dele.**

Due diligence para crédito/seguro exige reproduzibilidade e auditoria — um run com os mesmos insumos deve produzir o mesmo laudo. Agentes (loops LLM com ferramentas) introduzem variância e custo por natureza; portanto:

- **Núcleo determinístico** (consolidate, enrichers, rule engine, persistence): permanece código.
- **Agentes** entram onde o problema é *aberto, heterogêneo ou interativo* — exatamente onde escrever código determinístico é caro ou impossível.
- **MCP** entra como a *interface padrão* entre os dados canônicos da GN e qualquer cliente LLM — interno ou externo.

## 2. O que é cada coisa (para alinhamento)

- **MCP (Model Context Protocol)**: protocolo aberto que expõe ferramentas/recursos/prompts a clientes LLM (Claude, copilotos, agentes). Um servidor MCP é essencialmente uma API com contrato descoberto dinamicamente pelo modelo.
- **Agente**: loop modelo→ferramenta→resultado→modelo até concluir uma tarefa. "Tempo real" aqui significa interativo/streaming (copiloto) ou orientado a eventos (triagem de monitoramento), em oposição ao batch do pipeline.

## 3. MCP — onde faz sentido

### 3.1 GN como *provedora* de servidores MCP (recomendado — alto valor)

Expor os serviços existentes como ferramentas MCP read-only:

| Servidor MCP | Backend por trás | Ferramentas típicas |
|---|---|---|
| `gn-property` | property/persistence | `get_snapshot`, `search_property(car\|sigef\|cpf)`, `get_fact_pack` |
| `gn-legal` | legal-service (RAG) | `search_norms(query, dimensao, uf)`, `get_norm(urn)` |
| `gn-geo` | geoanalysis | `get_timeseries(zone, index)`, `get_events(prodes\|deter\|firms)` |

Um único investimento habilita quatro consumidores:
1. **Copiloto do analista (RF-90)** — chat sobre a propriedade com citação de fatos/normas;
2. **ai-analysis** — o evaluator pode consumir os mesmos contratos em vez de payloads ad-hoc;
3. **Operação interna** — Claude Code/Desktop dos devs consultando snapshots em diagnóstico;
4. **Produto B2B futuro** — banco/seguradora pluga o *seu* agente nos dados GN via MCP com OAuth/escopo por tenant (diferencial competitivo: "seus agentes, nossos dados auditados").

Guardrails: somente leitura; auth herdada do auth-service; escopo por tenant; logging de cada tool call (trilha LGPD); respostas sempre com proveniência.

### 3.2 GN como *consumidora* de MCP externo (cautela)

Não usar MCP como via de ingestão de fontes governamentais — não existem servidores MCP maduros para SICAR/INCRA/ANA etc., e ingestão exige determinismo e versionamento que o pipeline atual já dá. Exceção pontual: ferramentas MCP de browser/automação usadas *por agentes de aquisição* (ver 4.1).

## 4. Agentes — casos com ROI claro

### 4.1 Agente de aquisição para a cauda longa de fontes (maior ROI)
**Problema**: certidões estaduais/municipais, portais de SEMA, cartórios, juntas comerciais — dezenas de portais heterogêneos, com layout que muda. Escrever e manter um scraper por UF não escala.
**Desenho**: agente com browser/computer-use que, dado `{tipo_certidao, ente, NI}`, navega, emite, baixa o PDF e o entrega ao pipeline (que valida hash, extrai e persiste com proveniência). O agente *não escreve no canônico* — só produz artefatos que o pipeline determinístico processa.
**Critério de sucesso**: custo por certidão obtida < custo de manter scraper dedicado; taxa de sucesso por portal monitorada; fallback humano.

### 4.2 Extração agêntica de matrícula (RT-06)
Matrículas longas, manuscritas ou de má qualidade se beneficiam de um loop: ler → recortar/ampliar trecho → reler → cruzar com CCIR/SIGEF → emitir JSON com confidence. Permanece *bounded* (uma matrícula por tarefa), com saída validada por schema — agência na tarefa, determinismo no resultado.

### 4.3 Investigação dirigida de red flags ("deep research" interno)
Quando o rule engine levanta um flag material (ex.: sobreposição com TI + processo possessório), um agente investiga **sob demanda**: cruza snapshot (via MCP `gn-property`), RAG (`gn-legal`), DataJud e mídia adversa na web, e produz um *anexo de investigação* com citações — claramente rotulado como camada interpretativa, separado do laudo determinístico.

### 4.4 Copiloto do analista (tempo real de verdade)
Caso interativo/streaming: o analista pergunta "por que o score ambiental caiu?" e o agente responde lendo o fact pack e o laudo, citando fatos. É o consumo natural dos servidores MCP do §3.1. Baixo risco (read-only), alto valor de UX.

### 4.5 Triagem de monitoramento (RF-80) — agente opcional
Eventos (novo DETER, entrada em lista suja) são triados por **regras** (materialidade = interseção geográfica + perfil). LLM entra só para redigir a notificação contextualizada. Agente completo aqui é overengineering; manter event-driven determinístico.

## 5. Onde agentes NÃO fazem sentido

- Substituir o scatter-gather do consolidate ("agente que decide quais domínios consultar") — o plano de consulta é determinístico por design (gn_commons.control.plan) e deve continuar assim.
- Cálculo de métricas/severidades — rule engine.
- Escrita direta em blocos canônicos — todo dado derivado de agente entra via pipeline com proveniência `agent_derived` e validação de schema.

## 6. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Variância no laudo | Agentes nunca decidem veredito; anexos interpretativos separados e rotulados |
| Custo descontrolado | Orçamento de tokens por tarefa; tiers de modelo (ADR-PLAT-029); batch API para extração em lote |
| Prompt injection via documentos externos (PDFs, portais) | Conteúdo externo tratado como não-confiável; agente de aquisição sem credenciais sensíveis no contexto; allowlist de domínios |
| LGPD em tool calls | Logging/auditoria por chamada; escopo mínimo por token |
| Lock-in de runtime | Contratos MCP são portáveis por definição; runtime do agente (SDK Anthropic, Azure) fica atrás de interface própria |

## 7. Sequência recomendada

1. **MCP read-only sobre snapshot + RAG** (gn-property, gn-legal) → destrava copiloto e uso operacional interno. Esforço baixo: é wrapper sobre APIs existentes.
2. **Extração agêntica de matrícula** junto com RT-06 (já necessária para o domínio registral).
3. **Copiloto do analista** consumindo os MCP do passo 1.
4. **Agente de aquisição** para 2–3 portais-piloto de certidão estadual (medir custo/sucesso antes de generalizar).
5. **Investigação dirigida** quando fact pack + rule engine (RT-08) estiverem maduros.
6. MCP B2B multi-tenant como tese de produto, após 1–3 validados internamente.
