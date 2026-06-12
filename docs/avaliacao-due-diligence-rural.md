# Avaliação Profunda — Due Diligence de Propriedades Rurais (backends gn-api-*)

> Data da avaliação: 2026-06-12
> Escopo: serviços `gn-api-*` e repositórios de apoio (gn-schemas, gn-cli-dataprocessor, gn-db-scripts, gn-platform-governance), com foco em due diligence completo de imóveis rurais para transação imobiliária, financiamento rural, seguro rural e aderência à legislação brasileira (fundiária, ambiental, fiscal e tributária).
>
> Método: varredura de código e documentação via busca org-wide (schemas canônicos v1.5–v1.7, ADRs PLAT-021/022/026/029, MANIFESTO, auditorias `claude_audit`, functional docs do projeto Sentinela). Itens marcados "verificar" dependem de confirmação fora do código.

---

## 1. Sumário executivo

A plataforma já cobre **muito bem a camada cadastral e a camada ambiental-geoespacial** (CAR, SIGEF, SNCR, CAFIR, CCIR, sobreposições, PRODES/DETER/FIRMS/MapBiomas, clima, RAG normativo e laudo LLM). As maiores lacunas para um due diligence completo estão nas camadas **registral (matrícula real, ônus, cadeia dominial — ONR ainda "to-be")**, **hídrica (outorgas ANA/estaduais)**, **mineral estruturada (SIGMINE existe só como camada de mapa)**, **trabalhista/sanções (lista suja, CEIS/CNEP)** e **judicial ampla (DataJud tem credencial mas não tem provider implementado)**.

Arquiteturalmente o desenho é sólido (consolidate → persistence canônico → enrichers → ai-analysis), com governança madura (ADRs, schema versionado). Os pontos de eficiência a atacar são: dualidade MSSQL+Postgres no plano cadastral, rastreabilidade de evidência por fato (proveniência), perfis de avaliação por finalidade (transação/crédito/seguro) e monitoramento contínuo pós-laudo.

---

## 2. O que já temos (as-is)

### 2.1 Arquitetura macro

```
gn-cli-dataprocessor (ingestão batch de camadas públicas)
        │
        ▼
[Fase 1] gn-api-consolidate-service ── scatter-gather de ~14 domain services
        │   car / sigef / sncr / cafir (views SQL) · sobreposicao (PostGIS, 16 categorias)
        │   entidades (SERPRO PF/PJ) · serpro_ccir · serpro_cnd · legal (JusBrasil)
        │   ibama_debitos / ibama_embargos / ibama_regularidade (DirectData via hub)
        ▼
gn-api-persistence-service ── fronteira única de I/O (AFS/Blob), snapshot canônico
        │   v1.5 → v1.7 (gn-schemas), merge atômico GET-merge-PUT
        ▼
[Fase 2] enrichers via Redis Streams (ADR-PLAT-026 define o "dono" de cada integração)
        │   geoanalysis: Sentinel-2 L2A (Planetary Computer/STAC), MapBiomas Col.10,
        │     PRODES, DETER, FIRMS · 7 índices espectrais · 9 zonas canônicas
        │     alinhadas ao CAR (ADR-PLAT-021) · legal_thresholds (déficit APP/RL por
        │     bioma, flags PRODES/DETER, queimada recorrente, consolidada pré-2008)
        │   climate: NASA POWER, CHIRPS, ERA5 (cache Postgres)
        │   legal: RAG pgvector com 13 dimensões jurídicas (ADR-PLAT-022)
        ▼
gn-api-ai-analysis-service ── LLM evaluator → gn_evaluation
            (narrative + 4 scorings + findings com norm_references;
             model tiers/delivery ADR-PLAT-029; Azure OpenAI ou Anthropic c/ caching)
```

Apoio: `gn-api-hub-service` (integrador stateless SERPRO/JusBrasil/DirectData; ONR previsto), `gn-api-legal-ingestion-service` (catálogo declarativo de normas + pipeline download/parse/tag/embedding), `gn-api-report-design/portal` (laudos/relatórios), `gn-web-landanalysis` (novo FE, fase F2).

### 2.2 Matriz de cobertura por camada do due diligence

| Camada | Fonte/integração | Status |
|---|---|---|
| **Cadastral** | CAR/SICAR (temas ambientais, área do imóvel, RL, APP) | ✅ implementado (batch + views + temporal) |
| | SIGEF/INCRA (parcelas certificadas, matrícula como texto) | ✅ implementado |
| | SNCR (consulta pública Serpro) | ✅ implementado |
| | CAFIR (NIRF) | ✅ implementado — ⚠️ verificar migração CAFIR→CIB (Receita/SINTER) |
| | CCIR (SERPRO) — áreas registradas, vencimento | ✅ implementado |
| | Entidades PF/PJ (SERPRO CPF/CNPJ, QSA) | ✅ implementado |
| **Registral** | Matrícula, ônus, gravames, cadeia dominial (ONR/SREI, cartórios) | ❌ gap — ONR "to-be" no macro-diagrama; matrícula hoje é só um campo textual do SIGEF/CCIR |
| | CNIB (indisponibilidade de bens) | ❌ gap |
| **Fiscal/tributário** | CND federal (SERPRO, incl. imóvel rural) | ✅ implementado (flag) |
| | Débitos ITR/DITR, dívida ativa PGFN | ❌ gap |
| | Faturamento (SERPRO) | 🔶 implementado, flag OFF |
| **Ambiental** | Embargos/débitos/regularidade IBAMA (DirectData) | ✅ implementado |
| | Embargos/autos ICMBio | 🔶 camada de mapa; sem domínio estruturado no snapshot |
| | Desmatamento PRODES/DETER, queimadas FIRMS, LULC MapBiomas | ✅ implementado (Sentinela fases 0/A/B/C) |
| | Sobreposições: UC, TI/aldeias (FUNAI), quilombolas, assentamentos (INCRA), florestas públicas, massa d'água (SNIRH) | ✅ implementado (16 categorias) |
| | PRA (adesão/termo de compromisso), CRA, ASV, licenças estaduais LP/LI/LO | ❌ gap (coberto só na base normativa RAG) |
| **Hídrico** | Outorgas ANA (CNARH/REGLA) e órgãos estaduais | ❌ gap — só norma no RAG; massa d'água existe como camada |
| **Mineral** | SIGMINE/ANM (processos minerários) | 🔶 download da camada nacional existe; sem domínio estruturado (fase, substância, titular, incidência) |
| **Jurídico** | Processos cíveis/criminais (JusBrasil) | ✅ implementado |
| | DataJud/CNJ | 🔶 API key presente em env; **sem provider implementado** |
| **Trabalhista/sanções** | Lista suja trabalho escravo (MTE), CEIS/CNEP, CADIN | ❌ gap (só vocabulário no RAG) |
| **Crédito/protestos** | Protestos (CENPROT/IEPTB), SCR Bacen | ❌ gap |
| **Normativo (RAG)** | Corpus federal multi-tema (13 dimensões: ambiental, fundiário, cadastral, registral, tributário, trabalhista rural, hídrico, carbono, etc.) | ✅ implementado |
| **Avaliação** | gn_evaluation (LLM, rastreável a normas) | ✅ implementado — agnóstico à finalidade (sem perfis por caso de uso) |

---

## 3. Gaps priorizados e o que implementar

### P0 — destravam o produto "due diligence completo"

1. **Camada registral (matrícula)** — maior lacuna para transação e crédito.
   - Integração ONR/SREI (pesquisa prévia, certidão digital de matrícula, acompanhamento registral) no `hub-service` (ADR-PLAT-026 já reserva essa categoria para o hub).
   - **Extração estruturada da matrícula** (PDF → JSON canônico): cadeia dominial, proprietários, área registrada, ônus (hipoteca, alienação fiduciária, penhora, usufruto, servidão, indisponibilidade), averbações (RL averbada, georreferenciamento). Pipeline: Document AI (Azure Document Intelligence ou LLM multimodal) + schema Pydantic versionado em `gn-schemas` + validação humana opcional.
   - **CNIB** (indisponibilidade de bens) por CPF/CNPJ do proprietário.
   - Novo bloco canônico `gn_properties.registral[]` + batimento de áreas **matrícula × CCIR × SIGEF × CAR × CAFIR** com flags de divergência (hoje o CCIR já traz `areas_registradas[]`, que é o gancho natural).

2. **Provider DataJud (CNJ)** — credencial já existe; implementar no hub ao lado do JusBrasil (decisão de design já registrada no property-service: "legal" é genérico, providers somam — Escavador/DataJud/CNJ). Classificar processos por tema (possessória, ambiental, trabalhista, execução fiscal) e por vínculo (pessoa × imóvel).

3. **Sanções e trabalho escravo** — baixo custo, alto valor para crédito/seguro/ESG:
   - Lista suja MTE (CSV público), CEIS/CNEP/CADIN (API Portal da Transparência), como domínios `sancoes_*` no consolidate, consultados por CPF/CNPJ das entidades já presentes no snapshot.

4. **Perfis de avaliação por finalidade** no ai-analysis: hoje o laudo é agnóstico. Criar `evaluation_profile` (transação | financiamento | seguro | compliance) com pesos e hard-stops distintos (ex.: embargo IBAMA ativo = impeditivo de crédito pelas regras do CMN/MCR; divergência de área registral = risco alto em transação). Implementável como rule engine determinístico **antes** do LLM (ver §5.3).

### P1 — completam as camadas ambiental/hídrica/mineral

5. **Outorgas de uso de água**: ANA (CNARH/REGLA, dados abertos SNIRH) + principais órgãos estaduais (DAEE-SP, IGAM-MG, etc., onde houver dado aberto). Diferencial técnico: cruzar **pivôs/irrigação detectados pelo geoanalysis** (índices espectrais já existem) × outorgas vigentes → flag "uso de água sem outorga aparente".
6. **Domínio mineral estruturado**: transformar a camada SIGMINE em domínio de sobreposição com atributos (processo ANM, fase, substância, titular, % de incidência sobre o imóvel); incluir CECAV (cavidades naturais) como camada de restrição.
7. **Regularização ambiental**: status PRA/termo de compromisso (SICAR estadual onde disponível), CRA, ASV; estruturar embargos ICMBio como domínio (hoje só camada).
8. **Fiscal**: débitos ITR/dívida ativa PGFN (lista de devedores é pública); acompanhar a transição **CAFIR → CIB (SINTER/Receita)** e o CNIR (vínculo Receita-INCRA) — risco regulatório de a chave NIRF mudar de semântica (verificar cronograma oficial).

### P2 — diferenciais de produto

9. **Monitoramento contínuo pós-laudo** (subscription): re-checagem periódica de embargos, alertas DETER, lista suja, vencimento CCIR/CND, novas averbações — vocês já têm "Robô de acompanhamento de RL" como módulo; generalizar para "Robô de due diligence" com eventos no message bus e notificação (email-service já existe).
10. **Grafo de relacionamentos** (UBO/beneficiário final): QSA do SERPRO já vem no snapshot; materializar grafo (Postgres recursive CTE basta; Neo4j só se a navegação virar produto) para risco de pessoas em cadeia societária.
11. **Protestos e certidões estaduais/municipais** para o perfil "transação" (CENPROT; certidões de distribuidores cíveis/fiscais — avaliar providers como Escavador/DirectData).
12. **Carbono/ESG**: a dimensão "carbono" já existe no RAG (REDD+, PSA, CRA, Lei 14.119/2021); conectar com o saldo de RL/vegetação nativa do geoanalysis para relatório de ativo ambiental (oportunidade, não só risco).

---

## 4. Sugestões de tecnologia e integração

| Necessidade | Sugestão | Observação |
|---|---|---|
| Extração de matrícula/certidões | Azure Document Intelligence **ou** LLM multimodal (Claude/GPT) com schema Pydantic + few-shots; armazenar PDF como evidência content-addressed (hash) | Encaixa no padrão hub→consolidate→persistence já existente |
| Consultas judiciais | DataJud (público, gratuito) como base; JusBrasil/Escavador como complemento pago | Provider plugável — decisão já registrada no CHANGELOG do property-service |
| Sanções | API Portal da Transparência (CEIS/CNEP), CSV lista suja MTE | Cron no dataprocessor + domínio no consolidate |
| Outorgas | Dados abertos ANA/SNIRH (CNARH/REGLA via WFS/CSV) | Mesmo padrão dos integradores WFS já feitos (PRODES/DETER) |
| Mineral | SIGMINE WFS/shape (já baixado) + atributos por interseção PostGIS | Reaproveitar pipeline de sobreposição |
| Rule engine determinístico | `cel-python`/JSON-Logic ou regras Python versionadas em `gn-schemas` | LLM narra; regras decidem hard-stops (consistência regulatória e auditabilidade) |
| Orquestração | **Manter** Redis Streams + workers (não migrar para Temporal/Airflow agora) | Corrigir os débitos operacionais (DLQ, mensagem stuck `F-PLAT-REDIS-16`, idempotência) tem ROI maior que troca de stack |
| Observabilidade de dado | Painel de *freshness* por fonte (data do último batch SICAR/SIGEF/embargos por UF) + cobertura por UF | O laudo deve declarar a data-base de cada fonte (hoje parcialmente via `_metadata`) |

---

## 5. Eficiência processual e arquitetural

1. **Unificar o plano cadastral em Postgres/PostGIS.** Hoje o consolidate lê 4 views MSSQL (CAR/SIGEF/CAFIR/SNCR) + PostGIS para overlay/funções — dois engines, dois pipelines de carga, dois pontos de falha. O `gn-db-scripts` já mostra o schema `public` PG com car/sigef/sncr/cafir e as functions `get_properties_by_*`. Consolidar a leitura no PG e aposentar as views MSSQL reduz custo e latência (manter MSSQL só para report/portal enquanto necessário).
2. **Proveniência por fato (cadeia de evidência).** Requisito para crédito/seguro: todo campo do snapshot e todo finding do laudo deve carregar `{fonte, data_base, url/documento, hash}`. O ai-analysis-advisor já aponta isso ("citação verificável... requisito para crédito/compliance") — promover de prática de prompt para **estrutura do schema canônico** (v1.8).
3. **Snapshot bitemporal / laudo reproduzível.** Para auditoria (seguro/financiamento) é preciso reconstituir "o que se sabia na data do laudo". O persistence já versiona schema; adicionar imutabilidade do snapshot usado por cada `gn_evaluation` (apontar `request_id` → blob congelado, sem merge posterior).
4. **Separar score determinístico de narrativa LLM.** As `legal_thresholds`/flags já são determinísticas no geoanalysis; estender o padrão para todos os domínios (embargo ativo, CCIR vencido, CND positiva, sobreposição TI > x%) e deixar o LLM consumir o veredito, não produzi-lo. Reduz variância e custo (casa com ADR-PLAT-029).
5. **Sanear débitos conhecidos** apontados nas auditorias internas: consolidate gravando formato legado (bug documentado), mensagem stuck em `sobreposicao`, FE geoanalysis desalinhado de rota (`/geo-analysis` vs `/api/v1`), `gn_evaluation` populado em poucos snapshots (pergunta aberta no property-flow-audit). Fechar esses itens antes de adicionar novas fontes.
6. **LGPD.** O produto cruza CPF, processos judiciais e sanções: formalizar base legal por finalidade, minimização nos snapshots (mascarar CPF em logs/relatórios), política de retenção por categoria e trilha de acesso — vira requisito contratual com bancos/seguradoras.
7. **Aderência regulatória do laudo por finalidade** (apoia o §3.4): mapear o checklist do Manual de Crédito Rural/resoluções CMN-BCB (vedação de crédito em área embargada/desmate ilegal), exigências de seguradoras (SUSEP) e cartorárias (provimentos CNJ) como regras versionadas — o RAG já tem o corpus para citar a norma exata.

---

## 6. Roadmap sugerido (ordem de ataque)

| Fase | Entrega | Camadas |
|---|---|---|
| 1 (curto) | Sanções (lista suja/CEIS/CNEP) + DataJud provider + domínio ICMBio + perfis de avaliação com rule engine | jurídico, trabalhista, ambiental |
| 2 | ONR/matrícula (pesquisa + certidão + extração estruturada) + CNIB + batimento de áreas | registral, fundiário |
| 3 | Outorgas ANA + cruzamento irrigação×outorga + domínio mineral SIGMINE + PRA/licenças | hídrico, mineral, ambiental |
| 4 | ITR/PGFN + CIB/CNIR + protestos/certidões + monitoramento contínuo | fiscal, crédito, produto |
| transversal | Proveniência no schema (v1.8), snapshot imutável por laudo, unificação PG, saneamento de débitos, LGPD | plataforma |

---

## Adendo (2026-06-12) — refinamentos após feedback

Correções de estado: JusBrasil é o provider judicial atual no hub (DataJud entra como provider adicional, não substituto); **ONR já contratado/conveniado, desabilitado por custo**; NIRF→CIB entra na lista de acompanhamento regulatório.

### A. DataJud — desenho do provider

A API pública do DataJud (Res. CNJ 331/2020) é um índice de **metadados processuais por tribunal**, consultado via `_search` (sintaxe Elasticsearch). Limitação central: **não indexa CPF/CNPJ** nos campos pesquisáveis (LGPD) e a busca por nome de parte é inconsistente entre tribunais. Desenho recomendado:

- **Descoberta** (CPF/CNPJ → lista de processos): permanece com JusBrasil (ou Escavador como segundo provider).
- **Enriquecimento e validação** (nº CNJ → classe, assuntos, movimentos, grau, órgão julgador): DataJud, gratuito e oficial. Todo processo vindo do JusBrasil ganha um "carimbo DataJud" (existe/ativo/última movimentação) — barateia o refresh contínuo e dá fonte oficial ao laudo.
- Classificar processos por tema (possessória, ambiental, execução fiscal, **trabalhista/JT**) e por vínculo (pessoa × imóvel) no domínio `legal`.

### B. ONR — habilitar com controle de custo

Não reativar como domínio automático do consolidate. Tratar como **domínio premium on-demand**:

1. **Gating por perfil**: só dispara em `evaluation_profile = transação | financiamento` (ou clique explícito "puxar matrícula"), nunca no run padrão.
2. **Pré-qualificação gratuita**: usar `serpro_ccir.areas_registradas[]` (matrícula/transcrição, CNS, livro/ficha) + campo `matricula` do SIGEF para decidir *quais* matrículas pedir — pedir 1 certidão certa, não N.
3. **Cache por validade legal**: certidão de matrícula tem validade de 30 dias — reutilizar dentro da janela (content-addressed por CNS+matrícula+data); visualização (mais barata) para triagem, certidão só quando a finalidade exige fé pública.
4. **Repasse de custo**: precificar o relatório "com registral" como tier separado; quota por tenant + aprovação na UI.
5. O PDF vira evidência content-addressed e alimenta a extração estruturada (bloco `registral` proposto no §3.1).

### C. NIRF → CIB

Adicionado ao backlog regulatório: acompanhar a transição CAFIR→CIB (SINTER/Receita) e o CNIR; modelar a chave do imóvel fiscal como `{tipo: nirf|cib, valor}` desde já para não quebrar schema quando a mudança ocorrer.

### D. Legal RAG — plano de evolução (maior alavanca)

Ordem de ataque sugerida:

1. **Harness de avaliação primeiro** (pré-requisito de tudo): golden set de ~100–200 pares pergunta→normas/artigos esperados, métricas de context precision/recall e faithfulness (estilo RAGAS), rodando em CI. Sem isso, qualquer troca de embedding/chunking é fé.
2. **Chunking norma-aware**: chunk = unidade citável (artigo/parágrafo/inciso) com metadados hierárquicos (lei → capítulo → artigo) e identificador canônico (padrão LexML URN). O retrieval passa a devolver citações verificáveis, não trechos soltos.
3. **Vigência e relações**: grafo norma→norma (revoga, altera, regulamenta) em Postgres + flag de vigência por chunk; recuperar o decreto regulamentador junto da lei; nunca citar norma revogada sem marcar. Texto compilado do Planalto como fonte preferencial.
4. **Retrieval híbrido + rerank**: BM25/tsvector + denso (pgvector HNSW) com fusão RRF; reranker cross-encoder (ou LLM-rerank barato) no top-50→top-10. O `query_builder` multi-dimensão atual vira o estágio de query expansion.
5. **Corpus**: jurisprudência (súmulas/teses STF-STJ — já apontado como faltante no legal-advisor), normas estaduais das UFs prioritárias, resoluções CMN/BCB (crédito rural) e circulares SUSEP (seguro) — são essas que dão o "para quê" do laudo.
6. **Embeddings**: avaliar no golden set (não trocar por moda): bge-m3 (open) vs voyage/text-embedding-3-large; dimensão reduzida se custo de índice pesar.

### E. ANA e CPRM/SGB

- **ANA/SNIRH**: CNARH (cadastro de usuários de recursos hídricos) + REGLA (outorgas federais) via dados abertos/geoserviços SNIRH (mesmo padrão WFS já usado para `massa_dagua` e PRODES/DETER — validar endpoint exato no catálogo de metadados SNIRH). Cruzamentos de alto valor: outorga × ponto de captação dentro do imóvel; **irrigação detectada pelo geoanalysis sem outorga correspondente**.
- **CPRM/SGB**: (a) **SIAGAS** — poços tubulares cadastrados; cruzar poço dentro do imóvel × outorga/cadastro de uso → flag "captação subterrânea sem registro"; (b) **GeoSGB** — geologia, hidrogeologia (potencial de aquífero), suscetibilidade a movimento de massa/inundação (relevante para seguro rural) via WMS/WFS; complementa o SIGMINE no domínio mineral.
- Dono sugerido (ADR-PLAT-026): geoespacial por geometria → `geoanalysis`; cadastros transacionais (CNARH por CPF/CNPJ) → `hub`.

### F. Sanções trabalhistas — estratégia composta (não depender da planilha)

A "lista suja" (Cadastro de Empregadores, hoje regido pela Portaria Interministerial 18/2024) é mesmo só uma planilha semestral (~613 empregadores em abr/2026; nome sai após 2 anos). Sozinha, cobertura baixíssima. O caminho viável é um **índice de risco trabalhista composto**, por CPF/CNPJ das entidades já presentes no snapshot:

| Sinal | Fonte | Confiabilidade | Acesso |
|---|---|---|---|
| **CNDT** (Certidão Negativa de Débitos Trabalhistas) | TST | Alta — documento oficial com código de validação | Emissão automatizável por CPF/CNPJ; é o instrumento com valor legal (usado em licitação/crédito) |
| Lista suja | MTE | Alta, cobertura baixa | XLSX semestral → ingerir como **ledger temporal** (first_seen/last_seen por publicação, nunca lookup pontual) |
| Processos na Justiça do Trabalho | JusBrasil (descoberta) + DataJud-JT (validação) | Média-alta | Já existe a infra; classificar por classe/assunto trabalhista |
| CEIS/CNEP/CEPIM | API Portal da Transparência | Alta | REST com API key gratuita |
| TACs e ACPs do MPT | Consulta pública MPT | Média | Scraping/monitoramento |
| Regularidade FGTS (CRF) | Caixa | Alta (só PJ) | Consulta pública por CNPJ |

Regra de produto: o laudo declara **cada sinal separadamente com fonte e data-base** ("CNDT negativa em DD/MM; ausente da lista suja vigente de DD/MM; 2 reclamatórias ativas na JT") — nunca um booleano "sem risco trabalhista". A CNDT é o quick win com lastro jurídico; a lista suja vira histórico versionado, não verdade única.

### G. Preparação de dados para o LLM (anti-alucinação)

Princípio: **o LLM não decide nem calcula — narra e fundamenta**. Camadas:

1. **Fact pack determinístico**: antes do LLM, um compilador transforma o snapshot em fatos atômicos com ID estável, valor, fonte e data-base (`F-012: embargo IBAMA ativo desde 2021-03-15 | fonte: DD/IBAMA | data-base: 2026-06-10`). Todos os números pré-calculados (déficits, percentuais, batimentos de área) — o modelo nunca faz aritmética.
2. **Rule engine antes do LLM**: hard-stops e severidades saem de regras versionadas (estende o padrão `legal_thresholds` do geoanalysis a todos os domínios); o LLM recebe veredito + evidência e produz narrativa e fundamentação.
3. **Saída estruturada validada**: schema fechado (structured outputs / `output_config.format` na Anthropic, equivalente no Azure OpenAI) com `findings[].fact_ids[]` e `findings[].norm_references[]` obrigatórios; pós-validação determinística rejeita/reprocessa qualquer claim cujo fact_id não exista, cujo número não bata com o fact pack (whitelist de numerais) ou cuja norma/artigo não exista no índice do RAG.
4. **Citations nativas**: no caminho Anthropic, passar o fact pack e os trechos normativos como blocos `document` com `citations: {enabled: true}` — o modelo devolve citações ancoradas por offset no documento fonte, verificáveis mecanicamente (no Azure OpenAI, emular via fact_ids).
5. **Disciplina de nulos**: campo por dimensão `status ∈ {avaliado, nao_avaliado_falta_dado}` ligado ao `skipped_stages` existente; o prompt proíbe inferir o que não está no fact pack — "não avaliado" é resposta válida e obrigatória.
6. **Golden set de laudos**: snapshots dourados → findings esperados; regressão em CI a cada mudança de prompt/modelo/tier (ADR-PLAT-029), com métrica explícita de taxa de claim não-citado.
7. Prompt caching (já em uso) favorece esse desenho: system prompt + corpus normativo estável primeiro, fact pack volátil por último.
