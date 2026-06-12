# Roadmap — Due Diligence de Propriedades Rurais

> Versão 1.0 — 2026-06-12 · Consolida `avaliacao-due-diligence-rural.md`, especificações funcional/técnica e `mcp-agentes.md`
> Classificação: **QW** quick win (≤ ~1 sprint, valor imediato) · **MH** must-have (núcleo do produto) · **NTH** nice-to-have · **DT/DF** dívida técnica/funcional

## 0. Imediato (independe de roadmap)

| Item | Tipo | Ref |
|---|---|---|
| Rotacionar e expurgar credenciais commitadas em `.env` (SERPRO, DataJud, JusBrasil — DomainViewer) | **Segurança** | DT-07 |

## 1. Quick wins (alto valor, baixo esforço)

| # | Item | Tipo | Refs | Observação |
|---|---|---|---|---|
| 1.1 | Provider CNDT (TST) + domínio no consolidate | QW | RF-30, RT-01 | Destrava dimensão trabalhista com lastro jurídico |
| 1.2 | Ledger lista suja MTE (ingestão semestral versionada) | QW | RF-30, RT-02 | Complementa 1.1 |
| 1.3 | Providers CEIS/CNEP/CEPIM (Portal Transparência) | QW | RF-30, RT-03 | API key gratuita |
| 1.4 | Provider DataJud (validação por nº CNJ) | QW | RF-20–22, RT-04 | Credencial já existe |
| 1.5 | Domínio mineral estruturado (SIGMINE → atributos por interseção) | QW | RF-50, RT-11 | Dado já baixado |
| 1.6 | Embargos ICMBio como domínio estruturado | QW | RF-60 | Camada já existe |
| 1.7 | Batimento de áreas CAR×SIGEF×CCIR×CAFIR com flags | QW | RF-02, RF-14 | Dados já no snapshot |
| 1.8 | Data-base por fonte declarada no laudo | QW | RF-72, DF-05 | Parcial via `_metadata` |
| 1.9 | Chave fiscal `{tipo: nirf\|cib}` no schema | QW | RF-03 | Preparação regulatória |

## 2. Must-have (núcleo do due diligence completo)

| # | Item | Tipo | Refs | Dependências |
|---|---|---|---|---|
| 2.1 | Harness de avaliação do RAG legal (golden set + métricas em CI) | MH | RT-09.1, DF-06 | — (pré-requisito do 2.2) |
| 2.2 | RAG: chunking norma-aware + vigência + retrieval híbrido/rerank | MH | RT-09.2–4 | 2.1 |
| 2.3 | Corpus RAG: jurisprudência STF/STJ + CMN/BCB + SUSEP + estaduais prioritárias | MH | RT-09.5, DF-06 | 2.1 |
| 2.4 | ONR domínio premium (gating, quota, cache 30d) | MH | RF-10–11, RT-05 | Convênio existente |
| 2.5 | Extração estruturada de matrícula + bloco `registral` | MH | RF-12, RT-06 | 2.4, schema v1.8 |
| 2.6 | CNIB por proprietário | MH | RF-13 | 2.4 |
| 2.7 | Schema canônico v1.8 (proveniência por fato + novos blocos) | MH | RT-07, DT-05 | — |
| 2.8 | Fact pack + rule engine + pós-validação de laudo | MH | RF-70–71, RT-08, DF-01/02 | 2.7 |
| 2.9 | Perfis de avaliação (transacao/financiamento/seguro/compliance) | MH | RF-70, DF-02 | 2.8 |
| 2.10 | Snapshot imutável por laudo | MH | DT-06 | 2.7 |
| 2.11 | Outorgas ANA (CNARH/REGLA) + cruzamento irrigação×outorga | MH | RF-40–41, RT-10 | — |
| 2.12 | Saneamento operacional: formato legado consolidate + DLQ Redis + rota FE geoanalysis | DT | DT-01/02/04 | — |
| 2.13 | LGPD formalizada (base legal, retenção, mascaramento, trilha de acesso) | DT | DT-09 | Pré-contrato bancos |

## 3. Nice-to-have / segunda onda

| # | Item | Tipo | Refs |
|---|---|---|---|
| 3.1 | SIAGAS (poços) + GeoSGB (suscetibilidade/hidrogeologia) | NTH | RF-42, RF-51, RT-10 |
| 3.2 | PRA/ASV estaduais onde houver dado | NTH | RF-61 |
| 3.3 | Débitos ITR / dívida ativa PGFN; protestos (CENPROT) | NTH | — |
| 3.4 | Monitoramento contínuo por assinatura | NTH→MH* | RF-80–81, RT-13 |
| 3.5 | Grafo societário/UBO (recursive CTE sobre QSA) | NTH | — |
| 3.6 | Relatório de ativo ambiental (carbono/PSA, saldo RL) | NTH | — |
| 3.7 | ~~Unificação cadastral em Postgres~~ → **promovido a must-have** (ver Trilha DETL D5; análise em `docs/avaliacao-gn-cli-dataprocessor.md` §4) | DT→MH | RT-12, DT-03 |
| 3.8 | Painel de freshness/cobertura por fonte e UF | DT | DT-08 |

\* vira must-have se o modelo de receita recorrente for priorizado.

## 4. Trilha MCP & Agentes (paralela — ver `mcp-agentes.md` §7)

| # | Item | Tipo | Gate |
|---|---|---|---|
| 4.1 | Servidores MCP read-only `gn-property` e `gn-legal` | QW | Wrapper de APIs existentes |
| 4.2 | Extração agêntica de matrícula | MH | Junto com 2.5 |
| 4.3 | Copiloto do analista (chat sobre snapshot) | MH | Após 4.1 |
| 4.4 | Agente de aquisição — piloto 2–3 portais de certidão | NTH | Medir custo/sucesso vs scraper |
| 4.5 | Investigação dirigida de red flags | NTH | Após 2.8 |
| 4.6 | MCP B2B multi-tenant (agentes de bancos/seguradoras) | NTH | Tese de produto; após 4.1–4.3 |

## 4b. Trilha DETL (gn-cli-dataprocessor — ver `docs/avaliacao-gn-cli-dataprocessor.md` §5.6)

| # | Item | Tipo |
|---|---|---|
| D1 | Containerização total (tippecanoe na imagem) + ACA Jobs cron p/ domínios leves | QW |
| D2 | Alertas de falha + dead-man's switch + painel freshness por fonte | MH |
| D3 | Fan-out por fila (KEDA) p/ car_geo/sigef_geo em workload profile dedicado | MH |
| D4 | GeoParquet raw + transform DuckDB/pyogrio + ingest PostGIS otimizado | MH |
| D5 | Migração tabular MSSQL→PG + promote unificado (absorve item 3.7) | MH |
| D6 | Supervisão agêntica (triagem de falhas → re-kick/issue/e-mail) | NTH |
| D7 | Higiene: docs únicas, DDL→gn-db-scripts, gn_commons, Py 3.14, golden files + contract tests | DT |

## 5. Sequência sugerida (macro)

```
Onda 0  ─ DT-07 (credenciais) ── imediato
Onda 1  ─ 1.1–1.9 + 2.12 + 4.1 ──────────── trabalhista/sanções, DataJud, saneamento, MCP base
Onda 2  ─ 2.1–2.3 (RAG) + 2.7 (schema v1.8) ─ fundação de qualidade
Onda 3  ─ 2.4–2.6 (registral) + 4.2 ───────── camada de maior gap
Onda 4  ─ 2.8–2.10 (laudo por perfil) + 4.3 ─ produto diferenciado
Onda 5  ─ 2.11, 2.13, 3.x, 4.4–4.6 ────────── expansão
```

## 6. Critérios de pronto (gates de qualidade)

- Toda nova fonte entra com: proveniência, data-base, teste de contrato, freshness monitorada.
- Toda mudança de RAG/prompt/modelo passa pelo golden set em CI.
- Nenhum dado derivado de agente entra no canônico sem validação de schema e rótulo `agent_derived`.
