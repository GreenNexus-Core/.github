# Especificação Funcional — Due Diligence de Propriedades Rurais

> Versão 1.0 — 2026-06-12 · Deriva de `docs/avaliacao-due-diligence-rural.md`
> IDs: `RF-xx` requisito funcional · `DF-xx` dívida funcional

## 1. Visão de produto

O produto entrega um **due diligence completo, transparente e auditável** de imóveis rurais, com quatro finalidades (perfis de laudo):

| Perfil | Consumidor típico | Ênfase |
|---|---|---|
| `transacao` | Compradores, imobiliárias, advogados | Registral (matrícula, ônus, cadeia dominial), divergências de área, indisponibilidade |
| `financiamento` | Bancos, cooperativas de crédito | Hard-stops CMN/BCB (embargo, desmate ilegal, lista suja), CNDs, garantias |
| `seguro` | Seguradoras | Risco físico (clima, queimada, geologia), sinistralidade ambiental, uso do solo |
| `compliance` | Produtor/empresa (aderência contínua) | Conformidade ambiental, fiscal, trabalhista; monitoramento |

Cada perfil define **pesos, hard-stops e seções do laudo** — mesmo snapshot canônico, vereditos diferentes.

## 2. Requisitos funcionais por camada

### 2.1 Cadastral (implementado — manutenção)
- **RF-01** Consulta unificada por CAR, SIGEF, SNCR, CAFIR, CCIR, CPF/CNPJ (existente).
- **RF-02** Batimento de áreas CAR × SIGEF × CCIR × CAFIR com flag de divergência percentual e severidade. *(novo)*
- **RF-03** Chave do imóvel fiscal modelada como `{tipo: nirf|cib, valor}` — preparação NIRF→CIB. *(novo)*

### 2.2 Registral (novo — prioridade máxima)
- **RF-10** Pré-qualificação de matrículas via `serpro_ccir.areas_registradas[]` e campo `matricula` do SIGEF (decidir o que pedir ao ONR sem custo).
- **RF-11** Obtenção de certidão/visualização de matrícula via ONR como **domínio premium on-demand**, com gating por perfil (`transacao|financiamento`), quota por tenant e cache pela validade legal (30 dias).
- **RF-12** Extração estruturada da matrícula (PDF → `gn_properties.registral[]`): proprietários, cadeia dominial, área registrada, ônus (hipoteca, alienação fiduciária, penhora, usufruto, servidão, indisponibilidade), averbações (RL, georreferenciamento).
- **RF-13** Consulta CNIB (indisponibilidade) por CPF/CNPJ dos proprietários.
- **RF-14** Batimento área registrada × CCIR × SIGEF × CAR com flag de risco para transação.

### 2.3 Jurídica
- **RF-20** Provider DataJud: validação/enriquecimento de processos por nº CNJ (classe, assuntos, movimentos, última movimentação). Descoberta permanece com JusBrasil.
- **RF-21** Classificação de processos por tema (possessória, ambiental, execução fiscal, trabalhista) e por vínculo (pessoa × imóvel).
- **RF-22** "Carimbo DataJud" de atualidade em todo processo exibido no laudo.

### 2.4 Trabalhista e sanções (novo)
- **RF-30** Índice composto de risco trabalhista por CPF/CNPJ, declarando cada sinal com fonte e data-base:
  - CNDT/TST (emissão automatizada, código de validação);
  - Lista suja MTE como **ledger temporal** (first_seen/last_seen por publicação semestral);
  - Processos JT (JusBrasil + DataJud);
  - CEIS/CNEP/CEPIM (API Portal da Transparência);
  - CRF/FGTS (PJ).
- **RF-31** Proibição de booleano agregado "sem risco trabalhista" — laudo sempre decompõe sinais.

### 2.5 Hídrica (novo)
- **RF-40** Outorgas ANA (CNARH/REGLA) por geometria e por CPF/CNPJ.
- **RF-41** Cruzamento irrigação detectada (geoanalysis) × outorga vigente → flag "uso de água sem outorga aparente".
- **RF-42** Poços SIAGAS dentro do imóvel × outorga/cadastro → flag "captação subterrânea sem registro".

### 2.6 Mineral e geológica (novo)
- **RF-50** Domínio estruturado SIGMINE: processos ANM incidentes com fase, substância, titular, % de incidência.
- **RF-51** Camadas GeoSGB: suscetibilidade geológica (movimento de massa, inundação) e contexto hidrogeológico — entrada do perfil `seguro`.

### 2.7 Ambiental (evolução)
- **RF-60** Embargos ICMBio como domínio estruturado (hoje só camada de mapa).
- **RF-61** Status PRA/termo de compromisso e ASV onde houver dado estadual acessível.

### 2.8 Avaliação e laudo
- **RF-70** `evaluation_profile` com pesos/hard-stops por finalidade; regras determinísticas versionadas decidem severidade, LLM narra e fundamenta.
- **RF-71** Todo finding com `fact_ids[]` e `norm_references[]` verificáveis; dimensões sem dado declaram `nao_avaliado_falta_dado`.
- **RF-72** Laudo declara data-base de cada fonte consumida.

### 2.9 Monitoramento contínuo (novo produto)
- **RF-80** Assinatura por propriedade: re-checagem periódica de embargos, alertas DETER, lista suja, vencimento CCIR/CND, novas averbações; notificação via email-service.
- **RF-81** Eventos materiais geram re-avaliação incremental (não full re-run).

### 2.10 Copiloto do analista (novo — ver `mcp-agentes.md`)
- **RF-90** Chat sobre o snapshot: perguntas em linguagem natural sobre uma propriedade, com respostas citando fatos e normas (read-only).

## 3. Dívidas funcionais

| ID | Dívida | Impacto |
|---|---|---|
| DF-01 | `gn_evaluation` ausente na maioria dos snapshots (audit interno) | Laudo incompleto em produção |
| DF-02 | Laudo agnóstico à finalidade (sem perfis) | Mesmo veredito para crédito e transação |
| DF-03 | Flags SERPRO CCIR/CND/faturamento e JusBrasil OFF por padrão sem política clara por tenant | Cobertura imprevisível do laudo |
| DF-04 | Matrícula apenas como texto do SIGEF/CCIR | Camada registral inexistente |
| DF-05 | Ausência de declaração de data-base por fonte no laudo | Auditabilidade fraca p/ banco/seguradora |
| DF-06 | RAG legal sem jurisprudência e sem normas CMN/BCB/SUSEP | Fundamentação incompleta p/ crédito/seguro |
