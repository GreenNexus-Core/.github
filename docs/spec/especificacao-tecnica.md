# Especificação Técnica — Due Diligence de Propriedades Rurais

> Versão 1.0 — 2026-06-12 · Deriva de `docs/avaliacao-due-diligence-rural.md` e da especificação funcional
> IDs: `RT-xx` item técnico · `DT-xx` dívida técnica

## 1. Princípios arquiteturais (invariantes)

1. **Pipeline canônico determinístico.** consolidate → persistence → enrichers → ai-analysis permanece determinístico e reproduzível. Agentes/LLM nunca substituem o pipeline; operam acima dele (ver `mcp-agentes.md`).
2. **Fronteiras de integração por domínio** (ADR-PLAT-026): cadastral/transacional → hub; geoespacial por geometria → geoanalysis; climático → climate; normativo → legal-ingestion/legal.
3. **Proveniência por fato**: todo campo carrega `{fonte, data_base, documento/url, hash}` (schema v1.8).
4. **Snapshot imutável por laudo**: cada `gn_evaluation` aponta para um blob congelado; merges posteriores não alteram o que fundamentou o laudo.
5. **Regras decidem, LLM narra**: severidades e hard-stops em rule engine versionado; saída LLM estruturada e pós-validada.

## 2. Especificações por iniciativa

### RT-01 · Provider CNDT (hub-service)
- Cliente TST por CPF/CNPJ; retorno PDF + status; parse do código de validação.
- Domínio `cndt` no consolidate (padrão `BaseDomainService` existente, por entidade do snapshot).
- Cache com TTL = validade da certidão (180 dias) com refresh configurável; PDF content-addressed (hash) no storage de evidências.

### RT-02 · Ledger lista suja (dataprocessor + property-service)
- Job semestral: download XLSX MTE → normalização CPF/CNPJ → upsert em `sancoes.lista_suja(ni, nome, uf, municipio, publicacao_id, first_seen, last_seen)`.
- Cada publicação é imutável (`publicacao_id`, data, hash do arquivo). Consulta responde "presente na publicação vigente?" e "histórico de presença".
- Domínio `sancoes_trabalhistas` agrega lista suja + CNDT + JT no snapshot.

### RT-03 · Providers CEIS/CNEP/CEPIM (hub-service)
- API Portal da Transparência (REST, API key) por CPF/CNPJ; domínio `sancoes_administrativas`.

### RT-04 · Provider DataJud (hub-service)
- Endpoint `_search` por tribunal (sintaxe Elasticsearch), consulta por `numeroProcesso`.
- Uso: validação/enriquecimento dos processos descobertos via JusBrasil; nunca discovery primário (API não indexa CPF/CNPJ).
- Normalização de classe/assunto via tabelas TPU/CNJ; classificação temática determinística (mapa assunto→tema) com fallback LLM apenas para resíduo, marcado como inferido.

### RT-05 · ONR como domínio premium (hub + consolidate + persistence)
- Novo domínio `registral`, disparado fora do scatter-gather padrão: fila própria, acionado por `evaluation_profile` ou ação explícita.
- Guardrails de custo: quota por tenant, orçamento por run, idempotência por (CNS, matrícula, janela de 30 dias).
- Artefatos: PDF content-addressed + metadados de emissão.

### RT-06 · Extração estruturada de matrícula
- Pipeline: PDF → Document AI (LLM multimodal ou Azure Document Intelligence) → schema Pydantic `RegistralMatricula` versionado em gn-schemas → validações determinísticas (CPFs válidos, soma de frações ideais, datas coerentes, batimento de área com CCIR) → persistência em `gn_properties.registral[]` com `confidence` por campo e link para o PDF-evidência.
- Extração agêntica (loop com ferramentas de recorte/releitura) para matrículas longas/manuscritas — ver `mcp-agentes.md` §4.2.
- Golden set de matrículas anotadas para regressão de extração.

### RT-07 · Schema canônico v1.8
- `provenance` obrigatório por bloco/fato; `gn_properties.registral[]`; `sancoes_*`; `hidrico` (outorgas, poços); `mineral` (processos ANM estruturados); chave fiscal `{tipo: nirf|cib}`.
- `gn_evaluation` ganha `profile`, `fact_pack_ref`, `snapshot_ref` (imutável).

### RT-08 · Fact pack + rule engine (ai-analysis)
- Compilador determinístico snapshot → fatos atômicos `{id, valor, fonte, data_base}`; números pré-calculados.
- Rule engine versionado (regras Python puras ou cel-python) produz findings candidatos com severidade por perfil; estende o padrão `legal_thresholds` do geoanalysis.
- LLM consome fact pack + findings e produz narrativa estruturada (`output_config.format`); pós-validador rejeita claims sem `fact_id`, números divergentes ou normas inexistentes no índice RAG; caminho Anthropic usa Citations API (blocos `document` com `citations: enabled`).
- Golden set de laudos com regressão em CI (taxa de claim não-citado como métrica de bloqueio).

### RT-09 · Evolução do Legal RAG (legal-service + legal-ingestion)
1. Harness de avaliação (golden set pergunta→normas, context precision/recall, faithfulness) em CI — pré-requisito.
2. Chunking norma-aware (artigo/parágrafo/inciso, URN LexML, metadados hierárquicos).
3. Grafo de vigência (revoga/altera/regulamenta) em Postgres; flag de vigência por chunk; texto compilado como fonte preferencial.
4. Retrieval híbrido (tsvector/BM25 + pgvector HNSW + fusão RRF) e reranker cross-encoder no top-50→top-10.
5. Corpus: súmulas/teses STF-STJ, estaduais prioritárias, resoluções CMN/BCB, circulares SUSEP.
6. Avaliação de embeddings no golden set antes de qualquer troca.

### RT-10 · Integrações hídrica e geológica
- ANA: integradores WFS/REST CNARH+REGLA (padrão dos integradores PRODES/DETER no geoanalysis para o recorte geográfico; hub para consulta por NI).
- SIAGAS (poços) e GeoSGB (suscetibilidade, hidrogeologia) como camadas de sobreposição + atributos.
- Cruzamentos: índice de irrigação (espectral) × outorga; poço × outorga.

### RT-11 · Domínio mineral estruturado
- SIGMINE (já baixado) → interseção PostGIS com atributos (processo, fase, substância, titular, % área) em vez de camada visual; CECAV como restrição.

### RT-12 · Unificação do plano cadastral em Postgres
- Migrar leitura de CAR/SIGEF/CAFIR/SNCR das views MSSQL para o schema `public` PG (functions `get_*` já existem); MSSQL permanece só para report/portal até migração própria.

### RT-13 · Monitoramento contínuo
- Scheduler por assinatura: deltas por domínio (novo embargo, alerta DETER no polígono, entrada na lista suja, CCIR vencendo) → evento no message bus → re-avaliação incremental + notificação.

## 3. Dívidas técnicas

| ID | Dívida | Origem | Risco |
|---|---|---|---|
| DT-01 | Consolidate gravando formato legado de storage | progress 2026-05-18 | Quebra de consumidores (já quebrou legal-service) |
| DT-02 | Mensagem stuck em `sobreposicao` / DLQ ausente (F-PLAT-REDIS-16) | redis-key-catalog | Perda silenciosa de domínio no snapshot |
| DT-03 | Dualidade MSSQL+PG no plano cadastral | as-is | Custo, latência, dois pipelines de carga |
| DT-04 | FE geoanalysis desalinhado de rota (`/geo-analysis` vs `/api/v1`) | fit-gap websummit | Funcionalidade quebrada standalone |
| DT-05 | Sem proveniência estruturada por fato | schema ≤ v1.7 | Bloqueia auditoria bancária |
| DT-06 | Snapshot mutável após laudo (merge contínuo) | persistence | Laudo não reproduzível |
| DT-07 | Credenciais reais commitadas em `.env` (DomainViewer: SERPRO, DataJud, JB) | gn-core-image-processor | **Segurança — rotacionar e expurgar imediatamente** |
| DT-08 | Sem freshness SLA/painel por fonte | — | Laudo com dado velho sem aviso |
| DT-09 | LGPD: base legal/retenção/mascaramento não formalizados | — | Risco contratual com bancos/seguradoras |
