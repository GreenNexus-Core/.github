# Especificação — Plataforma SaaS GreenNexus (Frontend do zero)

> Versão 1.0 — 2026-06-12 · Substitui conceitualmente os `gn-web-*` (que servem apenas de inspiração/lições aprendidas)
> Companion: `roadmap-execucao.md` (atividades encadeadas) · Backend de referência: specs em `docs/spec/`

## 1. Visão e princípios

**Uma plataforma, dois aplicativos, um fluxo central.** O usuário pesquisa uma propriedade (CAR, SIGEF ou clique no mapa), acompanha o processamento em tempo real e recebe um dossiê navegável com mapa profissional e laudo — com profundidade definida pelo plano (Básico / Standard / Premium).

Princípios de produto:
1. **Mapa é o palco, não um widget** — a propriedade renderizada com fit-bounds é o momento "uau" do produto; tudo orbita em torno dela.
2. **Processamento transparente** — o usuário vê cada domínio sendo consultado (SSE já existe no consolidate); espera com confiança ou sai e é notificado.
3. **Preço sem surpresa** — Premium (cadeia dominial) sempre apresenta orçamento calculado antes de executar.
4. **Entitlements no servidor** — tier/quota decididos no backend; o frontend apenas reflete (nunca esconde-no-cliente).
5. **Profissional por padrão** — densidade de informação de ferramenta B2B, dark mode, atalhos, estados vazios/erro desenhados, p95 < 2,5s no primeiro mapa.

## 2. Decisões de stack

| Camada | Decisão | Justificativa |
|---|---|---|
| Framework | **React 19 + TypeScript + Vite** | Continuidade de competência do time; ecossistema |
| Estrutura | **Monorepo pnpm + Turborepo**: `apps/web` (usuário), `apps/admin` (back-office), `packages/{ui,map,schemas,api-client,i18n,auth}` | Reuso entre os dois apps; `packages/schemas` espelha gn-schemas (Zod) |
| Rotas/estado servidor | **TanStack Router + TanStack Query** | Cache por snapshot/run_id, invalidação por evento SSE |
| Formulários | react-hook-form + Zod | Validação compartilhada com schemas canônicos |
| Design system | **Tailwind + shadcn/ui (Radix)** + tokens próprios GN | Controle total de identidade, a11y via Radix, custo zero de licença |
| Gráficos | Recharts (dashboards) + visx p/ séries temporais do geoanalysis | |
| **Mapa** | **Engine abstraction** (`packages/map`, API neutra): **MapLibre GL primário** (vetor, PMTiles, GeoJSON) + **adapter Leaflet exclusivo para WMS** + deck.gl p/ COG/heatmaps — decisão validada no landanalysis F2.1 | Ver §2.1 |
| Realtime | SSE nativo (consolidate já emite) com reconexão + fallback polling | Vocabulário de eventos já existe (`consolidate.dispatch.*`) |
| Telemetria | OpenTelemetry web + App Insights; product analytics (PostHog self-host) | Funil pesquisa→laudo→upgrade é a métrica do negócio |
| i18n | pt-BR base, en preparado (`packages/i18n`) | Já havia padrão no landanalysis |
| Testes | Vitest + Testing Library; Playwright e2e (fluxo central como smoke obrigatório de CI) | |

### 2.1 Estratégia de engine — lição aprendida do landanalysis (F2.1)
**MapLibre "puro" não cobre o caso GN** — conclusão já validada nas discussões do gn-web-landanalysis: parte das sobreposições vem de **WMS de órgãos públicos** (catálogo dinâmico via GetCapabilities, CRS fora do EPSG:3857, GetFeatureInfo), e o MapLibre só consome raster pré-tilado em Web Mercator. A decisão comprovada lá (F2.1-01..03, `02-monorepo-layout.md`) é a que esta plataforma adota e evolui:

- **`packages/map` com API neutra** (`<MapCanvas engine="maplibre|leaflet">`): **MapLibre primário** para tudo que é vetorial — PMTiles (`pmtiles://`), GeoJSON do snapshot, clique/identify, e o editor de estilo, já que `line-dasharray`/`line-width`/`line-color`/`*-opacity` são paint properties nativas com expressão por atributo (ex.: cor por `nome_tema`).
- **Adapter Leaflet exclusivo para WMS** (chunk lazy ~43KB, carrega só quando a rota usa) — `<WmsLayer/>` só roda em engine leaflet, como no `@gn/maps` do landanalysis. Portar esse pacote (componentes e testes existentes) em vez de reescrever.
- **Plano de convergência** (para o Leaflet morrer um dia, como já era o intent F3+): cada fonte WMS recorrente migra para o pipeline DETL→PMTiles/COG; para WMS de uso eventual, avaliar tile-proxy de reprojeção (WMS→XYZ 3857) no BFF. Critério de aposentadoria do adapter: zero camadas WMS no catálogo.
- deck.gl apenas para COG/heatmaps do geoanalysis.

## 3. Modelo de produto — tiers e entitlements

| Capacidade | **Básico** | **Standard** | **Premium** |
|---|---|---|---|
| Pesquisa CAR / SIGEF / mapclick | ✅ | ✅ | ✅ |
| Domínios cadastrais: CAR, SIGEF, CAFIR, SNCR, sobreposições (16), CCIR, CND, JusBrasil | ✅ | ✅ | ✅ |
| Mapa + camadas + relatório cadastral | ✅ | ✅ | ✅ |
| ai-analysis (laudo LLM), legal (RAG), geoanalysis (temporal), climate | — | ✅ | ✅ |
| Tier LLM | — | **FAST (default)** com upgrade por análise para **HIGH** (custo adicional exibido) | FAST/HIGH |
| **Cadeia dominial** (ONR: certidões de matrícula + extração + grafo dominial) | — | — | ✅ com **orçamento dinâmico** |
| Monitoramento contínuo (fase 2 do produto) | — | add-on | add-on |

- Entitlements resolvidos no backend (claim no JWT + endpoint `/entitlements`); `model_tier`/`delivery` do ai-analysis **já existem** (ADR-PLAT-029) — o switch FAST→HIGH é UI sobre contrato pronto.
- **Orçamento Premium** (pré-execução, sempre): `nº de matrículas` detectado no snapshot Básico/Standard (`serpro_ccir.areas_registradas[]` + SIGEF) × `preço por matrícula` (custo ONR + margem) + `estimativa LLM` (tokens médios por matrícula × preço do tier) → tela de cotação → aceite explícito → execução. Falha parcial = cobrança parcial proporcional + crédito.
- Catálogo de produto: reaproveitar/evoluir o schema `product_management` existente (products, product_modules) como fonte do catálogo de planos.

## 4. Arquitetura de aplicação e BFF

```
apps/web (usuário)  ──┐
                      ├── gn-api-platform-bff (novo, fino):
apps/admin (interno) ─┘    sessão, entitlements, agregação de status,
                           cotação premium, billing, rate-limit por tenant
                                │
        auth ─ hub ─ consolidate ─ persistence ─ geoanalysis ─ climate ─ legal ─ ai-analysis ─ report-*
```

**BFF é necessário** (decisão): hoje o FE legado fala com N serviços; com billing/entitlements/cotação entra lógica que não pertence nem ao browser nem aos serviços de domínio. O BFF é deliberadamente fino (sem regra de negócio de domínio) e é o único origin do browser (CORS simples, tokens httpOnly).

## 5. Fluxo central (espinha dorsal do produto)

### 5.1 Pesquisa
- **UnifiedSearchBar** com detecção automática de input: código CAR (`UF-XXXXXXX-…`), parcela SIGEF, coordenadas, CPF/CNPJ (Standard+) — validação Zod, sugestões com debounce.
- **Mapclick**: camada nacional de imóveis (PMTiles do CAR/área do imóvel) sempre ativa em zoom alto; clique → identifica candidatos (point-in-polygon via função PG existente `get_properties_by_*`) → card de confirmação.
- Histórico de pesquisas e propriedades favoritas por usuário.

### 5.2 Processamento (modal de acompanhamento)
- Disparo do consolidate → **modal-stepper por domínio** alimentado por SSE (`consolidate.dispatch.{car,sigef,sncr,cafir,layers,hub.*}` → estados consultando/ok/vazio/falha/pulado-por-plano).
- Comportamentos obrigatórios: reconexão SSE com recuperação de estado (run status na persistence); **modo background** ("continuar navegando") com Central de Processamentos + toast + e-mail opcional ao concluir; falha parcial não bloqueia o resultado (domínio com badge de indisponível + retry pontual).
- Tempo estimado por domínio com base em percentis históricos (telemetria own).

### 5.3 Resultado (dossiê da propriedade)
Layout de 3 zonas: **mapa** (60–70% da tela), **painel de domínios** (tabs/accordion), **barra de ações** (gerar relatório, upgrade de análise, monitorar, compartilhar).
- Render imediato das geometrias (GeoJSON do snapshot) com **fit-bounds animado** na propriedade (padding, maxZoom), highlight pulsante de 1,5s na entrada.
- Multi-CAR/multi-parcela: seletor de unidades com fit no conjunto e foco por unidade.
- Tabs por domínio (Básico: CAR, SIGEF, CAFIR, SNCR, Temas/sobreposição, CCIR, CND, Jusbrasil) + (Standard: Laudo IA, Legal/normas, Geoanalysis com timeline e índices, Clima) + (Premium: Cadeia dominial com grafo e linha do tempo de ônus).
- Deep-link por aba/estado do mapa (URL = estado), pré-requisito para compartilhamento e suporte.

## 6. Mapa e camadas

### 6.1 Fontes
| Tipo | Fonte | Render |
|---|---|---|
| Geometrias da propriedade e temas do imóvel | GeoJSON do snapshot | `geojson` source MapLibre |
| Camadas nacionais (sobreposições: TI, UC, assentamentos, embargos, mineração, massa d'água, florestas públicas…) | **PMTiles vetorial** (pipeline DETL `tiles` já existe) | `vector` source via protocolo pmtiles |
| Camadas WMS legacy/externas (geoservers de órgãos, catálogo dinâmico) | WMS GetMap/GetCapabilities | **Adapter Leaflet** (`<WmsLayer/>`) até migração p/ PMTiles/COG (§2.1) |
| Heatmaps/índices do geoanalysis (NDVI etc.), contornos | COG + contours (endpoints `/heatmaps/...` já existem) | raster tiles ou deck.gl BitmapLayer |
| Basemaps | OSM vetorial (estilo próprio) + satélite (provider a contratar) | style switcher |

### 6.2 Painel de camadas (componente-chave)
- Árvore de camadas com grupos (Imóvel / Cadastral / Ambiental / Fundiário / Mineral / Hídrico / Análises), busca, liga/desliga, reordenação por drag, slider de opacidade por camada, legenda viva.
- **Editor de estilo por camada**: cor (paleta GN + custom), espessura, **tipo de linha** (sólida/tracejada/pontilhada → `line-dasharray`), preenchimento e transparência, cor por atributo (ex.: `nome_tema` do CAR com a paleta canônica das 9 zonas ADR-PLAT-021).
- **Presets**: padrão GN, presets por finalidade (ex.: "Análise ambiental" liga APP/RL/embargos), presets do usuário (persistidos por tenant/usuário); export/import de preset (JSON) para padronização entre equipes.
- Interações: hover tooltip configurável, clique → ficha do feature, medição (distância/área), desenho de AOI (fase 2), comparador antes/depois (swipe) para camadas temporais do geoanalysis.

### 6.3 Catálogo de camadas como contrato
`packages/map/catalog.ts` gerado a partir de um **catálogo servido pelo BFF** (id, fonte, tipo, estilo default, min/maxzoom, tier mínimo, legenda) — fim do `vector-layers.ts` de 350 linhas hard-coded do FE legado; camada nova = entrada no catálogo, sem deploy do front.

## 7. Área do usuário (módulos)

1. **Dashboard** — KPIs do tenant (pesquisas no período, processamentos em andamento, laudos gerados, consumo de quota), atalhos, últimas propriedades.
2. **Pesquisa & Propriedade** — fluxo central (§5).
3. **Minhas propriedades** — lista/cards com thumbnail de mapa estático, filtros (UF, status, plano usado), tags do usuário.
4. **Central de processamentos** — fila viva (SSE), histórico de runs com status por domínio, retry, custo incorrido (Premium).
5. **Relatórios** — laudos gerados (report-portal), viewer com sumário navegável, export PDF, link de compartilhamento com expiração; templates conforme plano (report-design já suporta templates e `car_block`).
6. **Monitoramento** *(fase 2)* — assinaturas por propriedade, feed de eventos (novo embargo, DETER, lista suja), configuração de alertas.
7. **Equipe & acesso** — gestão de membros, papéis (§8), convites por e-mail (email-service).
8. **Plano & faturamento (self-service)** — plano atual, consumo vs quota, upgrade/downgrade, métodos de pagamento, faturas/NF, histórico de cotações Premium aceitas.
9. **Configurações** — perfil, notificações, presets de mapa, API keys *(fase 2: API pública + MCP B2B, conecta com `mcp-agentes.md` §3.1)*.

## 8. RBAC

**Modelo**: tenant (organização) → membros com papéis; permissões = `recurso:ação`; enforcement no backend (BFF valida, serviços re-validam), frontend apenas oculta/desabilita.

| Papel (tenant) | Descrição |
|---|---|
| `owner` | Tudo + billing + excluir tenant |
| `admin` | Gestão de membros, configurações, tudo de analyst |
| `analyst` | Pesquisar, processar, gerar laudos, aceitar cotação Premium **se** tiver `billing.approve` delegado |
| `viewer` | Ver propriedades/laudos; não dispara processamento |
| `billing` | Faturas, métodos de pagamento, aprovação de cotações |

- Aprovação de gasto: cotações Premium acima de limite configurável exigem papel com `billing.approve` (workflow de aprovação in-app + e-mail).
- Auditoria: toda ação sensível (processar, aceitar cotação, exportar laudo, mudar papel) em log de auditoria visível ao `owner/admin` (e ao admin interno).
- Backoffice (apps/admin) tem RBAC próprio: `support`, `ops`, `finance`, `superadmin`.

## 9. Faturamento e pagamento

**Modelo de cobrança híbrido**: assinatura mensal/anual por tier (com quota de pesquisas/processamentos) + **uso medido** (Premium por matrícula; upgrade LLM HIGH por análise; excedente de quota).

| Componente | Decisão/Recomendação |
|---|---|
| Gateway | **Decisão (2026-06-12, revisada): Stripe US como gateway único** — subsidiária BR descartada; modelo de referência Anthropic/OpenAI: self-serve SMB com cartão + Enterprise sales-led com Stripe Invoicing (wire/transferência USD). Mitigações obrigatórias do cross-border: (a) presentment em **BRL** com taxas embutidas no preço (custo ~2,9% + 1,5% intl + 1% conversão — embutir + margem de câmbio); (b) **aviso de IOF** no checkout ("sua fatura pode incluir IOF de compra internacional") — reduz tickets; (c) onboarding orienta cartão habilitado p/ compra internacional **antes** do checkout; (d) ligar Adaptive Acceptance, network tokens, Smart Retries e card account updater; (e) **crédito como meio primário** — débito BR cross-border tem baixa aprovação: manter aceito, sem prometer; (f) auto-refill via saved payment method + off-session PaymentIntents (MIT) e credit grants do Billing. Interface `PaymentProvider` mantida: se a taxa de aprovação BR doer no funil, adiciona-se processador local por segmento/BIN sem rework. Nota Enterprise: pagamento por wire de invoice estrangeira pode acionar retenções no cliente (IRRF/CIDE/ISS-import) — tratar em pricing de contrato |
| Nota fiscal | Emissão NFS-e via integrador (eNotas/NFE.io) disparada por fatura paga — obrigatório Brasil |
| Metering | Eventos de consumo (`run.completed`, `matricula.processed`, `llm.high.used`) publicados pelo BFF → agregação por período → invoice items |
| Cotação Premium | Objeto `quote` persistido: insumos (n matrículas, preço unitário vigente, estimativa LLM), validade 7 dias, aceite com identidade do aprovador (auditoria), execução vinculada ao quote |
| Dunning | Retentativas, e-mails, suspensão suave (read-only) após X dias |
| Trial/POC | Standard trial 14 dias com N pesquisas, sem cartão (decisão de growth, default proposto) |

## 10. Administração interna (apps/admin)

1. **Tenants & assinaturas** — CRUD, plano, quotas, flags por tenant (feature flags já existem na plataforma), suspensão, impersonation com trilha.
2. **Catálogo de produto** — planos, preços, preço por matrícula, preços LLM por tier (versionados, com vigência — mudança de preço não afeta quote aceito).
3. **Operações** — visão das runs (todas as tenants), reprocessamento, painel de freshness das fontes (consome control tables do DETL — conecta com trilha D2), status de integrações (SERPRO/ONR/JusBrasil/DirectData com circuit breaker state).
4. **Financeiro** — faturas, inadimplência, créditos/reembolsos, conciliação gateway, margem por tenant (custo ONR/LLM × receita).
5. **Suporte** — busca por tenant/propriedade/run, linha do tempo do processamento, logs vinculados (deep-link App Insights), reenvio de e-mails.
6. **Auditoria & segurança** — trilha de ações (interno + tenants), gestão de papéis internos.

## 11. Requisitos não-funcionais

- **Performance**: TTI < 3s em 4G; primeiro render do mapa < 2,5s; bundle inicial < 300KB gz (mapa em chunk próprio); virtualização de listas.
- **Acessibilidade**: WCAG 2.1 AA (Radix ajuda); mapa com alternativas textuais (tabela de camadas ativas).
- **Segurança**: tokens httpOnly via BFF; CSP estrita; rate-limit por tenant; sem dado sensível em localStorage; LGPD (consentimento, export/erasure por tenant).
- **Resiliência**: SSE com retry/backoff + recuperação por polling; UI funcional com domínio indisponível.
- **Observabilidade**: web vitals + funil de produto (pesquisa→processamento→laudo→upgrade) + erro por release (sourcemaps).

## 12. Dependências de backend (novas, para o roadmap)

| ID | Dependência | Consumidor |
|---|---|---|
| B-01 | BFF `gn-api-platform-bff` (sessão, entitlements, catálogo de camadas, agregação de status) | tudo |
| B-02 | Serviço/módulo de billing (planos, metering, quotes, webhooks gateway, NFS-e) | §9 |
| B-03 | Endpoint de cotação Premium (conta matrículas do snapshot + tabela de preços vigente) | §3/§9 |
| B-04 | RBAC multi-tenant no auth-service (papéis, convites, claims de entitlement) | §8 |
| B-05 | Catálogo de camadas servido (BFF) sobre PMTiles existentes | §6.3 |
| B-06 | Domínio registral/ONR (Onda 3 do roadmap da plataforma) — **pré-requisito do Premium** | §3 |
| B-07 | Eventos de consumo padronizados nos serviços (run.completed etc.) | metering |
