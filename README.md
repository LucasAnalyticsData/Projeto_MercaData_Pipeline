# Retail Pricing & Margin Pipeline

Pipeline de dados end-to-end — **Bronze → Silver → Gold** sobre **Databricks Serverless** e **ADLS Gen2** — que transforma eventos brutos de catálogo em KPIs de preço, margem e qualidade de dado para decisão executiva, sem auditoria manual por SKU.

Projeto real de estágio em Engenharia de Dados & IA, operando sobre uma base de **3.805 SKUs ativos / 135 categorias**, com pipeline de quarentena auditável, observabilidade por execução e alertas automáticos (Teams + e-mail).

![Python](https://img.shields.io/badge/Python-3.10-blue)
![PySpark](https://img.shields.io/badge/PySpark-3.x-orange)
![Databricks](https://img.shields.io/badge/Databricks-Serverless-red)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-OSS-00ADD8)
![Azure](https://img.shields.io/badge/Azure-ADLS%20Gen2%20%7C%20SQL-0078D4)
![Status](https://img.shields.io/badge/pipeline-produção-brightgreen)

---

## TL;DR

| | |
|---|---|
| **O que é** | Pipeline medallion que ingere catálogo de e-commerce, valida qualidade de dado linha a linha, calcula margem e expõe 9 tabelas de KPI para BI executivo (Looker/Metabase), via Azure SQL Server |
| **Por que importa** | Decisão de preço/margem passou de "confiar no cadastro" para "cobertura de custo validada e rastreável": **99,97%** dos SKUs ativos com custo real cadastrado, zero auditoria manual |
| **Escala atual** | 3.805 SKUs ativos · 135 categorias · 9 tabelas Gold · 7 regras oficiais de qualidade + 1 condicional de margem |
| **O que torna isso não-trivial** | Databricks Serverless impõe restrições reais de runtime (sem `spark.conf.set` global, sem `DeltaTable.merge()` estável, sem driver nativo de SQL Server) — cada uma virou uma decisão de arquitetura documentada abaixo, não um workaround escondido |
| **Resiliência** | Idempotência em toda camada (reprocessar nunca duplica), quarentena nunca derruba o pipeline, alertas (Teams/e-mail) nunca bloqueiam a execução |

---

## Contexto de negócio

A equipe de analytics em tempo real precisava de uma camada Gold confiável para decidir preço e margem sem auditoria manual por SKU. O desafio central nunca foi volume de dado — é **qualidade**: preços inválidos e custos ausentes corrompem silenciosamente o cálculo de margem se não forem isolados *antes* da camada analítica. Um SKU com preço zerado que entra sem barreira em um `AVG(margem)` não gera um erro visível — gera uma métrica errada que o gestor usa para decidir.

A resposta de arquitetura foi: **nunca descartar silenciosamente, nunca bloquear sem justificativa de negócio, e nunca deixar o BI calcular o que o pipeline já pode calcular de forma consistente.**

---

## Arquitetura

```
                         ADLS Gen2 — container RAW (produtor, somente leitura)
                                            │
                                            ▼
   01 · Extração  ──────────────────────────────────────────────────────────
   Lê só as janelas ainda ausentes do manifesto + Bronze. Nunca escreve,
   move ou renomeia nada na origem.
                                            │ DataFrames em memória
                                            ▼
   03 · Bronze (Delta) ──────────────────────────────────────────────────────
   Append-only, idempotente (anti-join por arquivo_origem). Log bruto —
   nenhuma regra de negócio aqui.
                                            │ leitura incremental (watermark)
                                            ▼
   04 · Silver (Delta) ──────────────────────────────────────────────────────
   Resolve a versão vigente por janela de origem · aplica 7 regras oficiais
   (planilha) + 1 condicional de margem · upsert manual (anti-join+overwrite)
                              │                              │
                   registros válidos                 registros rejeitados
                              │                              ▼
                              │                  ┌─────────────────────────┐
                              │                  │   QUARENTENA auditável  │
                              │                  │  motivo_rejeicao_*      │──► Teams/e-mail
                              │                  └─────────────────────────┘     (novo item)
                              ▼
   05 · Gold (Delta, append por ciclo_id) ───────────────────────────────────
   9 tabelas de KPI: catálogo, categoria, departamento, faixa de preço,
   qualidade de dados, SKUs rejeitados, + 3 históricos
                                            │ sincronização incremental
                                            ▼
                        Azure SQL Server (espelho — fonte de verdade é o Delta)
                   6 tabelas "_atual" (snapshot do ciclo corrente) + histórico completo
                                            │
                                            ▼
                                    BI executivo (Looker / Metabase)

   Observabilidade transversal: log_pipeline + log_data_quality (Delta, por run_id),
   espelhados no SQL Server — qualquer número estranho no dashboard é rastreável até
   a execução e a regra exata que falhou, sem abrir notebook.
```

**Por que não existe Landing Zone física:** "já ingerido" é decidido por dois registros complementares no próprio container do Lakehouse — um **manifesto JSON por execução** (`controle/ingestao/<tabela>/<run_id>.json`, barato de ler, carrega metadados de auditoria) e o **Bronze** (`arquivo_origem`, a trava final de idempotência). O container RAW pertence ao produtor dos dados: o pipeline nunca decide o que já leu escrevendo na própria origem — essa era a implementação original, corrigida depois de um achado crítico de auditoria (ver tabela abaixo).

---

## Decisões de engenharia (e os incidentes reais por trás delas)

Esta é a parte que diferencia um pipeline de tutorial de um pipeline que rodou em produção contra as restrições reais de uma plataforma. Cada linha abaixo é uma decisão **forçada por um erro real em execução**, não uma escolha estética:

| Decisão | Restrição / incidente real que a forçou | Impacto |
|---|---|---|
| **Upsert manual (anti-join + overwrite) em vez de `DeltaTable.merge()`** | `MERGE` e `OPTIMIZE` nativos lançam `SparkKeyProviderException` neste workspace Serverless — a credencial OAuth por chamada não é honrada pelo MERGE nativo. Migrar para Unity Catalog com external location destrava os dois. | Mesmo resultado de negócio (upsert completo), sem depender de infraestrutura ainda não disponível |
| **Escrita no SQL Server via `format("sqlserver")` nativo, nunca `jdbc`** | Serverless bloqueia DML via JDBC genérico (`UNSUPPORTED_DATA_SOURCE_WRITE`) — só libera formatos governados. Achado em execução real (02/10). Leitura continua via `jdbc`: `SELECT` não é DML, não é bloqueado. | Escrita funcional sem driver JDBC customizado |
| **Tabelas `_atual` no SQL Server em vez de views `CREATE OR ALTER VIEW`** | A primeira versão criava as views via `pymssql` porque o conector nativo não executa DDL. O simples `import pymssql` **travava o kernel inteiro com `SIGABRT`** — incompatibilidade binária da extensão FreeTDS com o runtime Serverless, não uma exceção Python capturável. Revertido. | BI lê estado atual sem risco de o pipeline inteiro cair por causa de uma lib de infraestrutura. Script de views (`06_*.sql`) mantido como alternativa manual opcional |
| **Container RAW nunca escrito pelo pipeline** | Achado crítico reportado em auditoria pelo parceiro de dupla (03/10): a versão original movia/renomeava arquivos dentro do RAW para "arquivar" o que já tinha sido lido. O RAW pertence ao produtor dos dados — o pipeline nunca deveria decidir o que já leu escrevendo na própria origem. | Causa raiz eliminada: controle de ingestão passou para manifesto JSON + Bronze, 100% no container do Lakehouse |
| **`adls_options` em todo read/write, nunca `spark.conf.set()` global** | Serverless bloqueia configuração global de sessão. | Credencial OAuth por chamada, sem estado de sessão compartilhado |
| **Existência de tabela Delta checada por metadado (`_delta_log` via SDK)** | Um `read` de teste é mais caro e depende de credencial Hadoop que nem sempre está disponível no momento do check. | Checagem de primeira execução barata e desacoplada do driver de leitura |
| **Watermark só avança após o lote inteiro ser publicado** | Ordem de commit é Bronze → manifesto → log (em 03) e merge → quarentena → watermark (em 04). Falha antes do último passo reprocessa o lote inteiro — os passos anteriores são idempotentes, então reprocessar nunca duplica. | Pipeline seguro a reprocessamento por erro humano ou falha de job, em qualquer ponto |
| **Dupla contagem corrigida em `gold_kpi_qualidade_dados`** | A regra de negócio "SKU ativo com preço inválido" estava sendo injetada como linha extra na tabela técnica de qualidade — inflava `SUM(qtd_rejeicoes)` para qualquer SKU que fosse "ativo" e "preço inválido" ao mesmo tempo. Achado em auditoria (02/10). | Corrigido removendo a duplicação na fonte, não documentando a inconsistência |
| **Diagnóstico `DEPARTAMENTO` para categorias-pai** | Categorias de nível raiz (que só agrupam subcategorias, nunca recebem produto diretamente) apareciam como `SEM_SKU` no dashboard — um falso alarme estrutural, não um problema de dado. | 12 categorias-departamento passam a ter diagnóstico correto; dashboard do cliente para de sugerir um problema que nunca existiu |

---

## Qualidade de dado: quarentena, não descarte

Toda linha que falha uma regra técnica **não é descartada** — vai para uma tabela de quarentena com `motivo_rejeicao`, auditável e nunca gravada duas vezes para a mesma versão do registro. A separação de responsabilidade é deliberada:

| Tipo | Exemplo | Comportamento |
|---|---|---|
| **Técnica (bloqueante)** | `sku` fora do padrão, `preco_lista` fora da faixa, `is_ativo` não-boolean, `preco_custo` inválido | Registro vai para quarentena — nunca entra no catálogo ativo |
| **Negócio (sinal, não bloqueio)** | Margem negativa/zero (SKU ativo com `preco_custo >= preco_lista`) | **Não bloqueia o SKU.** Vira flag/diagnóstico na Gold — é sinal de possível erro de cadastro de custo ou elasticidade de fornecedor, misturar com erro técnico confundiria dois problemas diferentes |
| **Referencial (alerta)** | Produto com `id_categoria` órfã | Decisão explícita do gestor (28/09): **alertar, não bloquear** |

**Regras oficiais implementadas** (fonte: planilha `regras_estagio_engdados_squad2_.xlsx`, aba "Squad 2" — 100% rastreáveis até o código):

| Tabela | Nº | Tipo | Regra |
|---|---|---|---|
| produtos | 1 | Técnica | `sku` não nulo, `5 < len(sku) < 60` |
| produtos | 2 | Técnica | `preco_lista` não nulo, `0 < preco_lista < 5000` |
| produtos | 3 | Técnica | `is_ativo` não nulo **e** tipo boolean |
| produtos | 4 | Negócio (KPI) | Nº de SKUs novos no lote; alerta se > 50 (possível carga de teste) |
| produtos | 5 | Negócio (alerta) | SKU ativo com `preco_lista <= 0` |
| produtos | 6 | Condicional | `preco_custo` não nulo e `> 0`, só quando a coluna existe na Silver |
| produtos | — | Negócio (fora da planilha) | Margem negativa/zero — alerta, não bloqueia |
| categorias | 1 | Técnica | `id_categoria` e `nome_categoria` não nulos |
| categorias | 2 | Negócio (alerta) | Nº de categorias raiz mudou entre ciclos |

### Observabilidade por execução

Duas tabelas Delta, uma linha por execução × camada × tabela (e × check, no caso de qualidade), ligadas por `run_id`:

- **`log_pipeline`** — volume lido/gravado/rejeitado, status, tempo, erro
- **`log_data_quality`** — um check por regra técnica e de negócio, com % de violação sobre as chaves únicas do lote

Ambas espelhadas no SQL Server pela Gold: **qualquer número estranho no dashboard é rastreável até o `run_id` e a regra exata que falhou, sem precisar abrir notebook.**

### Alertas automáticos (Microsoft Teams + e-mail)

A cada item **novo** em quarentena (nunca reavisa sobre o que já estava lá), um cartão vai para um webhook do Teams via `urllib` (stdlib — mesma decisão que evitou o driver nativo que travava o kernel). Canal de e-mail via `smtplib` (stdlib), independente do Teams: o gestor configura um, o outro, os dois, ou nenhum, só criando (ou não) os secrets correspondentes no Databricks Secret Scope. **Nenhum dos dois canais bloqueia o pipeline** — falha de rede, secret ausente ou erro de autenticação geram só um aviso impresso.

---

## Camada Gold — modelo de dados

9 tabelas, todas em **append por `ciclo_id`** (nunca overwrite — o histórico nunca é perdido):

| Tabela Gold | Granularidade | Decisão que apoia |
|---|---|---|
| `gold_kpi_catalogo_resumo` | 1 linha/ciclo | Saúde do catálogo: ativos, preço, cobertura, concentração (HHI), frescor, alertas |
| `gold_kpi_categoria_performance` | 1/categoria | Onde está o valor do catálogo e quais categorias exigem ação (classe de valor + diagnóstico) |
| `gold_kpi_categoria_raiz` | 1/departamento | Quais departamentos concentram sortimento e valor |
| `gold_kpi_faixa_preco` | departamento × faixa | Posicionamento de preço por departamento |
| `gold_kpi_qualidade_dados` | tabela × regra | Que regra está barrando produtos de entrar no catálogo, e com que frequência (`pct_rejeicoes` nativo, calculado em Spark) |
| `gold_kpi_rejeitados_produtos` | 1/SKU rejeitado | **Quais** SKUs estão em quarentena, com preço rejeitado, motivo, e se já existe versão válida na Silver |
| `gold_hist_catalogo_resumo` · `gold_hist_categoria_share` · `gold_hist_qualidade_dados` | série temporal | Evolução de catálogo, mix de categorias e qualidade de dado ao longo dos ciclos |

**Como o BI lê "agora" sem contar ciclo duplicado:** como as tabelas `gold_kpi_*` acumulam um ciclo por execução, somar sem filtrar conta o mesmo SKU várias vezes. O SQL Server mantém 6 tabelas `<tabela>_atual`, regravadas a cada ciclo só com o ciclo corrente — **o BI lê `_atual` para o estado de agora e `gold_kpi_*`/`gold_hist_*` para a evolução no tempo.**

**Definições que evitam viés estatístico comum:**
- **Classe de valor no catálogo (A/B/C)** — baseada em **exposição de valor de lista**, não em curva ABC clássica de giro/consumo (essa exigiria histórico de vendas, fora do escopo). Documentado explicitamente para não ser mal-interpretada pelo cliente.
- **HHI de concentração** (Σ share² por categoria, 0–10.000) — acima de 2.500 indica concentração alta.
- **Margem calculada só sobre SKUs ativos com custo informado** — SKU sem `preco_custo` não entra nem no numerador nem no denominador, para não subestimar a margem média artificialmente.
- **Preço médio por departamento é ponderado** (valor ÷ SKUs ativos), nunca média de médias; mediana e P90 vêm direto dos produtos, porque não são somáveis.

---

## Estrutura do repositório

```
notebooks/
  00_config_squad2_dupla1.py                          # credenciais (Secret Scope), conectividade ADLS/SQL,
                                                        # utilitários Delta, manifesto, log, alertas Teams/e-mail
  01_conexao_adls_squad2_dupla1_Parte_Lucas.py         # extração: lê só janelas ainda não ingeridas da origem RAW
  02_eda_ecommerce_squad2_dupla1.py                    # EDA: volumetria, nulos, distribuição de preço/margem,
                                                        # regras aplicadas ao dado cru, integridade referencial
  03_bronze_ingestao_delta_squad2_dupla1.py            # bronze: append-only idempotente, sem regra de negócio
  04_silver_ecommerce_squad2_dupla1.py                 # silver: 7 regras oficiais + condicional de margem,
                                                        # upsert manual, quarentena auditável
  05_gold_kpis_produtos_categorias_squad2_dupla1.py    # gold: 9 KPIs, append por ciclo, espelho SQL Server
  06_views_sqlserver_squad2_dupla1.sql                 # alternativa opcional: views "_atual" via DDL manual
docs/
  guia_dashboard_executivo_gold.md                     # dicionário de dados e guia de leitura dos KPIs
  padronizacao_dashboards.docx                         # padrão visual/semântico para o BI executivo
```

> Os notebooks são exportados no formato nativo do Databricks (`# Databricks notebook source`,
> células `# COMMAND ----------`) e podem ser reimportados diretamente no workspace.

### Como executar

```bash
# Ordem de execução (00 e 01 são herdados via %run, não rodam sozinhos):
03_bronze_ingestao_delta_squad2_dupla1      # 01 é herdado por este
04_silver_ecommerce_squad2_dupla1
05_gold_kpis_produtos_categorias_squad2_dupla1
```

Requer um Databricks Secret Scope com credenciais de ADLS Gen2 (`tenant-id`, `client-id`, `client-secret`, `storage-account`, `container`), Azure SQL (`sqlserver-host`, `sqlserver-database`, `sqlserver-user`, `sqlserver-password`) e, opcionalmente, webhook do Teams e credenciais SMTP — todos lidos sob demanda, nunca hardcoded.

---

## Resultados

- **99,97%** dos SKUs ativos com custo real cadastrado — cobertura validada para decisão de preço sem auditoria manual
- **3.805 SKUs ativos / 135 categorias** monitorados a cada ciclo, com diagnóstico automático por categoria
- Pipeline de quarentena isola erros de preço **na origem**, antes de qualquer cálculo de margem chegar ao BI
- Zero "silent drop": toda rejeição é rastreável por `motivo_rejeicao` até o `run_id` que a gerou
- 3 incidentes reais de plataforma (OAuth no MERGE, bloqueio de JDBC DML, `SIGABRT` do driver nativo) resolvidos com decisões de arquitetura documentadas, não workarounds frágeis

## Stack

**Processamento:** PySpark · Delta Lake · Databricks Serverless
**Storage:** Azure Data Lake Storage Gen2 (raw) · Delta (bronze/silver/gold)
**Serving:** Azure SQL Server (espelho via conector nativo `sqlserver`)
**Observabilidade:** tabelas Delta de log/DQ por `run_id` · webhooks Teams · SMTP
**BI:** Looker Studio / Metabase

## Autor

Lucas — estagiário em Engenharia de Dados & IA, foco em pipelines de qualidade de dado para varejo omnichannel.
