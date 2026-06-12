# Avaliação Crítica — gn-cli-dataprocessor (DETL)

> Versão 1.0 — 2026-06-12 · Complementa `avaliacao-due-diligence-rural.md` e `spec/roadmap.md`
> Contexto confirmado pelo owner: o CLI é refactor de uma refatoração anterior; documentação espalhada e não sanitizada; migração para ACA Jobs é to-do antigo; CAR e SIGEF-GEO têm input na casa de **centenas de GB**; objetivo declarado: **operação sem operadores humanos**.

## 1. Veredito

DETL com desenho acima da média (catálogo declarativo, contratos staging→promote→finalize, tracking de run, riscos autodeclarados R-801..R-810), porém: refactor inacabado congelado no estado intermediário, operação manual disfarçada de pipeline (frequency declarada mas sem scheduler; tippecanoe via WSL), dualidade MSSQL+PG duplicando o núcleo, testes estreitos onde o risco é largo (readers e promote), e duas árvores de documentação em drift.

## 2. Pontos fortes

- Catálogo declarativo como centro (`downloads.yaml`, domain catalog com `common.by_type`): fonte nova ≈ configuração.
- Contratos formais de carga com `run_id`, dedup, índices pós-carga, full_refresh; validação de identificadores com teste de SQL injection.
- Paralelismo real no `car_geo` (buckets por UF, waves, prefetch, prioridade de workers) — necessário para a escala do SICAR.
- OTel/App Insights presentes; runbook de modos de falha documentado.

## 3. Críticas principais

| # | Crítica | Evidência |
|---|---|---|
| C1 | Dualidade de engines duplica o núcleo: promote tabular em T-SQL (procs `raw_*.sp_promote_*`) e promote geo em Python/PostGIS | `src/scripts/sql/DDL/060_promote_tabular_procedures.sql` vs `src/writers/postgis_geometry_promoter.py` |
| C2 | Refactor no meio do caminho: dois `BasePipeline`, `car_geo` em ~9 módulos, `Protocol` tipando métodos privados (god-object fatiado em mixins) | `pipelines/base.py` + `pipelines/base_pipeline.py`; `DownloadExecutorRuntime` |
| C3 | Operação manual: `frequency` declarada sem scheduler; execução via CLI/ACA manual; `.env` obrigatório; tippecanoe via WSL | `downloads.yaml`; `architecture/as-is/06-config-env.md` |
| C4 | Isolado da plataforma: não usa `gn_commons` (tracking/locks/naming próprios); Python 3.11 vs 3.14 padrão | R-802, R-801 |
| C5 | Testes só em `tests/critical/` (config/planejamento); sem golden files de readers nem contract tests de promote | risco de corrupção silenciosa de dado cadastral |
| C6 | Higiene: duas árvores de docs (PT+EN), manifests/introspecção commitados, DDL/views no repo errado | R-803, R-807, R-808; `arquitetura/` vs `architecture/` |
| C7 | Downloads por browser (Playwright) sem alerta de quebra — quem percebe é humano | `browser_mode: true` (embargos IBAMA, SNCR) |

## 4. MSSQL × PG — recomendação

A assimetria decide: **PG é inamovível** (PostGIS = coração geo), **MSSQL é opcional**. A comparação correta é o custo *marginal* de mover o tabular para o PG existente versus o custo *total* do MSSQL: instância + ODBC em toda imagem + dialeto T-SQL com bus factor 1 + staging/promote duplicado + join lógico entre engines no consolidate + dois regimes de backup/monitoramento. Mesmo com fatura Azure menor, o custo de engenharia da dualidade supera a diferença e cresce a cada domínio novo.

Migração de baixo risco: schemas `raw_*` no PG → promote portado para plpgsql **ou** unificado em Python/SQLAlchemy Core junto com o promoter PostGIS → domain catalog aponta tabulares para PG (desenho declarativo torna isso quase config) → migrar as 4 views lidas pelo consolidate → desligar MSSQL do caminho DETL (report/portal migram depois). Congelar já a criação de domínios novos em MSSQL. *(Promovido no roadmap: item 3.7 → must-have.)*

## 5. Autoexecução sem operadores — alternativas tecnológicas

### 5.1 Princípio de desenho

Separar três papéis que hoje se confundem na figura do operador:

```
EXECUTAR   → infraestrutura de jobs (cron + fila), deterministicamente
ALERTAR    → telemetria com alarme por FALHA e por AUSÊNCIA (dead-man's switch)
SUPERVISIONAR → camada de triagem que decide re-tentar/ajustar/escalar (única candidata a agente)
```

Agente **não move centenas de GB**; pipeline determinístico move. Agente substitui o *julgamento* do operador, não o músculo.

### 5.2 Comparativo de executores

| Opção | Para quê serve aqui | Prós | Contras | Veredito |
|---|---|---|---|---|
| **ACA Jobs** (cron + event-driven/KEDA) | Executor padrão de todos os domínios | Cron nativo; trigger por fila (KEDA) = fan-out por shard; scale-to-zero; workload profile dedicado para jobs grandes; já era o plano | Limites de CPU/mem por réplica no plano consumption (4 vCPU/8 GiB) — exige workload profile dedicado p/ CAR; validar `replicaTimeout` p/ execuções muito longas | **Adotar (camada base)** |
| **AKS CronJobs** | Mesmo papel, se já houver AKS operado | Controle total, node pools sob medida | Custo operacional de cluster; vocês não querem operar infra | Só se AKS já for inevitável por outro motivo |
| **Azure Batch** (+ Spot VMs) | O músculo para CAR/SIGEF-GEO se ACA não bastar | VMs de qualquer tamanho; spot = 60–90% desconto p/ trabalho re-executável; pools com autoscale a zero; dependências entre tasks | Mais um serviço para conhecer; integração menos "container-native" | **Adotar sob demanda** (só para os 2 domínios monstruosos, se o profile dedicado do ACA não der) |
| **ACA Dynamic Sessions ("sandboxes")** | — | Execução efêmera de código não confiável (código gerado por LLM, REPL) | Não é para ETL de volume; sessões pequenas e efêmeras | **Não se aplica** a este problema |
| **GitHub Actions agendado** | Apenas gatilho/cron barato chamando ACA Jobs | Zero infra | Runner fraco e limite de 6h — jamais como executor de dado | Aceitável só como disparador; prefira cron nativo do ACA |
| **Data Factory/Synapse** | Orquestrador gerenciado com triggers/alertas | Agendamento, retry, monitor prontos | Atividade custom roda container do mesmo jeito; vira camada extra de config p/ pipeline 100% Python | Não recomendo: paga-se a camada sem usar o valor dela |
| **Orquestradores Python (Dagster / Prefect / Airflow)** | Orquestração quando dependências entre datasets e backfills ficarem complexos | Dagster: modelo de *assets* com freshness policy = exatamente a necessidade declarada; sensores; UI de lineage | Mais uma plataforma para operar (ou pagar cloud); para ~20 fontes com dependências simples, o catálogo + control tables já cobre | **Adiar com gatilho claro**: adotar Dagster quando (a) dependências entre datasets virarem grafo real, ou (b) backfill histórico virar rotina |
| **Temporal/Durable Functions** | Workflows duráveis de aplicação | Retomada exata de execução | Overkill para batch ETL; modelo intrusivo no código | Não recomendo |
| **Agent schedulers** (ex.: scheduled deployments de agentes) | **Supervisão**, não execução | Triagem com julgamento: ler logs, decidir re-kick, abrir issue, escalar | Variância e custo por token; nunca no caminho do dado | **Adotar na camada 3** (ver 5.4) |

### 5.3 Alertas sem operador (desenho concreto)

1. **Falha**: Azure Monitor alert rule sobre execução de ACA Job com status failed → Action Group (e-mail/Teams/webhook). OTel/App Insights já existem no CLI.
2. **Ausência (o alerta mais importante)**: *dead-man's switch* — query agendada sobre as control tables (`control.job/job_event` no PG): "domínio X sem promote bem-sucedido há mais que `frequency` + tolerância" → alerta. Cobre o caso "ninguém rodou", que alerta de falha não cobre. A coluna `frequency` do `downloads.yaml` finalmente vira contrato executável.
3. **Quebra de fonte** (C7): downloads browser_mode emitem métrica de sucesso por fonte; N falhas consecutivas → alerta específico "portal mudou".
4. **Painel de freshness por fonte/UF** (já era DT-08 do roadmap): mesma query, visão Grafana/Workbook.

### 5.4 Camada de supervisão agêntica (o "operador virtual")

Gatilho: webhook do alerta → agente com acesso read-only a logs (App Insights), control tables e código do CLI. Ações permitidas, em ordem: (a) diagnosticar e re-disparar o job com parâmetros ajustados (ex.: shard que falhou, retry com menos workers); (b) quarentenar a fonte e abrir issue com diagnóstico e log relevante; (c) escalar por e-mail com resumo legível (email-service existente). Nunca altera dado; só orquestra re-execução e comunica. Pode ser um scheduled agent diário ("verificar saúde do DETL") + reativo por webhook. É a materialização do "não quero operadores": o humano só lê o e-mail de exceção.

### 5.5 CAR/SIGEF-GEO em centenas de GB — alternativas de processamento

O gargalo não é orquestração, é **single-machine transform + ingest**. Opções complementares:

1. **Fan-out por fila (maior alavanca, menor mudança)**: o scheduler interno de buckets/waves do `car_geo` vira *sharding de infraestrutura* — mensagens `{uf, tema}` numa fila (Storage Queue/Service Bus), ACA Job event-driven (KEDA) processa 1 shard por execução. Ganhos: paralelismo horizontal real, retomada por shard (falhou MG-APP, reprocessa só MG-APP), código *mais simples* que o pool de workers atual, e custo proporcional.
2. **GeoParquet como camada raw intermediária**: shapefile → GeoParquet no blob (uma vez), e o load PostGIS passa a ler Parquet (COPY binário, colunar, resumível). Benefícios: re-runs sem re-download, transforms fora do banco, formato 5–10× menor que SHP, e habilita o item 3.
3. **DuckDB + extensão spatial para o transform**: ler SHP/GeoParquet, filtrar temas/colunas, reprojetar e particionar **sem tocar o Postgres**, em uma VM/job efêmero — engole centenas de GB num nó só com custo mínimo. PostGIS recebe apenas o resultado final via COPY. (pyogrio/GDAL continuam para os casos que DuckDB não cobre.)
4. **Ingest PostGIS otimizado**: tabelas particionadas por UF, drop/recreate de índices em volta do load (já fazem pós-load), `COPY` binário em vez de INSERT, e staging `UNLOGGED`.
5. **Azure Batch + Spot** apenas se o profile dedicado do ACA não comportar o pico — o transform CAR é o caso perfeito de spot (re-executável por shard).
6. **Questionar o volume na entrada**: carregar somente os temas consumidos pela plataforma (catálogo de 9 zonas canônicas ADR-PLAT-021) e manter o resto só em GeoParquet frio — o banco não precisa do que o produto não lê.

### 5.6 Sequência recomendada (trilha DETL)

```
D1. Containerizar 100% (tippecanoe na imagem, mata WSL) + ACA Jobs cron p/ domínios leves
D2. Alertas: falha + dead-man's switch + painel freshness (control tables já existem)
D3. Fan-out por fila p/ car_geo/sigef_geo (KEDA) em workload profile dedicado
D4. GeoParquet raw + transform DuckDB/pyogrio; ingest otimizado
D5. Migração tabular MSSQL→PG (promovida a must-have) + promote unificado
D6. Supervisão agêntica (webhook de alerta → triagem → re-kick/issue/e-mail)
D7. Higiene: árvore única de docs, DDL→gn-db-scripts, gn_commons, Python 3.14,
    golden files de readers + contract tests de promote
Gatilho p/ Dagster: dependências entre datasets virarem grafo real ou backfills rotineiros
```

## 6. Saneamento da documentação ("bagunça documentada")

Caso ideal para uma passada agêntica única e barata (Claude Code batch sobre o repo): inventariar `arquitetura/` + `architecture/` + READMEs órfãos → gerar árvore única canônica (`architecture/` com as-is verificado contra o código) → mover o histórico para `archive/` → abrir PR de consolidação para revisão humana. O agente faz o trabalho braçal de reconciliação; o humano só arbitra conflitos. Mesma técnica depois vira rotina de verificação de drift docs×código (mensal, com diff).
