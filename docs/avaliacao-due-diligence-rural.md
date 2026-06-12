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

*Documento gerado a partir de varredura do código e da documentação de governança da organização; validar itens regulatórios marcados com "verificar" com fonte oficial antes de comprometer roadmap.*
