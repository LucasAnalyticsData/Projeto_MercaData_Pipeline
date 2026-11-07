# Retail Pricing & Margin Pipeline

Pipeline de dados end-to-end para análise de pricing e margem em varejo alimentício omnichannel, seguindo arquitetura medallion (Bronze → Silver → Gold) sobre Databricks/ADLS Gen2.

![Python](https://img.shields.io/badge/Python-3.10-blue)
![PySpark](https://img.shields.io/badge/PySpark-3.x-orange)
![Databricks](https://img.shields.io/badge/Databricks-Community%20Edition-red)
![SQL](https://img.shields.io/badge/SQL-Azure%20SQL-lightgrey)

## Contexto de negócio

Equipe de analytics em tempo real precisava de uma camada Gold confiável para decidir preço e margem sem auditoria manual por SKU. O desafio central não era volume de dado — era qualidade: preços inválidos e custos ausentes que corrompem silenciosamente o cálculo de margem se não forem isolados antes da camada analítica.

## Arquitetura

```
ADLS Gen2 (raw)
      │
      ▼
Bronze ──► Silver (validação + regras de negócio) ──► Gold (KPIs)
                  │
                  ▼
         Tabela de quarentena
      (preço ≤0 ou > limite, motivo_rejeicao_amigavel)
```

- Ingestão: notebook de conexão ADLS Gen2 → Databricks (PySpark).
- Qualidade de dado: regras de validação separam preço tecnicamente inválido (bloqueado, nunca entra no catálogo ativo) de margem negativa (não bloqueia — fica como diagnóstico/flag na Gold, SKU continua ativo).
- Camada Gold: KPIs de margem, cobertura de custo cadastrado e distribuição de preço por departamento/categoria, consumidos por dashboard executivo (Metabase).

## Decisões técnicas e por quê

| Decisão | Motivo |
|---|---|
| Quarentena em vez de descartar linhas inválidas | Preserva auditabilidade — toda rejeição é rastreável por motivo_rejeicao_amigavel, não é um "silent drop" |
| Margem negativa não bloqueia o SKU | É sinal de negócio (possível erro de cadastro de custo ou elasticidade de fornecedor), não um erro técnico — bloquear junto com preço inválido misturaria dois problemas diferentes |
| Parquet como storage intermediário | Evita custo de parsing repetido de CSV e preserva schema/dtypes entre estágios |
| Agregação por distinct-SKU separada de contagem de ocorrência de regra | Um mesmo SKU pode disparar múltiplas regras — misturar as duas contagens infla o problema reportado ao negócio |

## Estrutura do repositório

```
notebooks/
  00_config_squad2_dupla1.py                          # credenciais, conectividade, observabilidade, alertas
  01_conexao_adls_squad2_dupla1_Parte_Lucas.py         # ingestão raw (ADLS Gen2) → bronze
  02_eda_ecommerce_squad2_dupla1.py                    # EDA exploratória da base
  03_bronze_ingestao_delta_squad2_dupla1.py            # bronze: ingestão append-only idempotente
  04_silver_ecommerce_squad2_dupla1.py                 # silver: validação de regras + quarentena
  05_gold_kpis_produtos_categorias_squad2_dupla1.py    # gold: KPIs de margem, catálogo e qualidade
  06_views_sqlserver_squad2_dupla1.sql                 # espelho/views no Azure SQL para o BI
docs/
  guia_dashboard_executivo_gold.md
  padronizacao_dashboards.docx
```

> Os notebooks são exportados no formato nativo do Databricks (`# Databricks notebook source`,
> células `# COMMAND ----------`) e podem ser reimportados diretamente no workspace.

## Resultados

- 99,97% dos SKUs ativos com custo real cadastrado (cobertura validada para decisão de preço sem auditoria manual)
- Pipeline de quarentena isola erros de preço na origem, antes de qualquer cálculo de margem

## Stack

PySpark · Databricks · Azure Data Lake Storage Gen2 · Azure SQL · Metabase (BI)

## Autor

Lucas — estagiário em Engenharia de Dados & IA, foco em pipelines de qualidade de dado para varejo omnichannel.
