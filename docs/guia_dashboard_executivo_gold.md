# Guia do Dashboard Executivo — Catálogo de Produtos & Categorias
**Squad 2 / Dupla 1 · Looker Studio · Fonte: Camada Gold (SQL Server `_atual`)**

---

## ETAPA 1: DIAGNÓSTICO DE REGRA DE NEGÓCIO E KPIs DE ALTO IMPACTO

### 1.1 Tese do Dashboard

> **"O catálogo está saudável, rentável e completo o suficiente para maximizar faturamento?"**

Perguntas estratégicas que o painel responde:

| # | Pergunta do C-Level | Onde está a resposta na Gold |
|---|---|---|
| 1 | Qual a exposição total do meu catálogo ativo? | `valor_lista_ativo` (catalogo_resumo) |
| 2 | Estou vendendo abaixo do custo em alguma categoria? | `margem_pct_media_ativos`, `diagnostico_codigo = MARGEM_NEGATIVA` |
| 3 | Quantos SKUs estão "perdidos" por dado ruim? | `skus_sem_versao_valida`, `taxa_rejeicao_produtos_pct` |
| 4 | Meu catálogo está concentrado demais em poucas categorias? | `indice_hhi_valor`, `pct_valor_top3_categorias` |
| 5 | Quais categorias devo priorizar para reativar/descontinuar? | `diagnostico_codigo`, `classe_valor_catalogo` |
| 6 | O dado está fresco o suficiente para eu confiar neste painel? | `horas_desde_ultima_ingestao`, `alerta_catalogo_defasado` |

### 1.2 Matriz de KPIs Prioritários

#### Camada 1 — Scorecards Executivos (Linha de KPIs)

| KPI | Campo Gold (`catalogo_resumo_atual`) | O que mede | Tipo Visual | Alavanca de Negócio |
|---|---|---|---|---|
| **Valor de Lista Ativo** | `valor_lista_ativo` | Exposição monetária total dos SKUs ativos | Scorecard (R$) | Dimensão do catálogo — se cai, estou perdendo sortimento |
| **Margem Média** | `margem_pct_media_ativos` | Margem bruta média ponderada dos ativos com custo | Scorecard (%) | Rentabilidade — abaixo de X%, revisar pricing |
| **SKUs Ativos** | `pct_skus_ativos` | % do catálogo que está ativo | Scorecard (%) | Cobertura de oferta — inativo = receita perdida |
| **SKUs c/ Preço Inválido** | `skus_ativos_preco_invalido` | Ativos na quarentena com preço ≤ 0 | Scorecard (contagem) | Receita bloqueada — SKU ativo sem preço não vende |
| **SKUs em Quarentena** | Contagem de `rejeitados_produtos_atual` | Produtos fora do catálogo por dado ruim | Scorecard (contagem) | Completude — cada SKU fora é receita potencial perdida |

#### Camada 2 — Análise Tática (Gráficos)

| KPI | Fonte Gold | Tipo Visual | Decisão que suporta |
|---|---|---|---|
| **Pareto de Valor por Departamento** | `categoria_raiz_atual` → `valor_lista_ativo`, `share_valor_pct` | Barras horizontais + linha acumulada | Onde está concentrado o valor — risco de dependência |
| **Distribuição por Faixa de Preço** | `faixa_preco_atual` (escopo `TODO O CATÁLOGO`) | Barras empilhadas | Posicionamento de preço — estou competitivo? |
| **Diagnóstico por Categoria** | `categoria_performance_atual` → `diagnostico_codigo` | Rosca / Pizza | Saúde do catálogo — quantas categorias precisam de ação |
| **Margem por Departamento** | `categoria_raiz_atual` → `margem_pct_media_ativos` | Barras horizontais (ranked) | Quais departamentos dão lucro vs. queimam margem |
| **Evolução do Catálogo** | `hist_catalogo_resumo` → série `total_skus`, `skus_ativos`, `valor_lista_ativo` | Linha temporal (por `ciclo_ts`) | Tendência — o catálogo está crescendo ou encolhendo? |
| **Evolução da Qualidade** | `hist_qualidade_dados` → `qtd_rejeicoes` por `regra` | Linha temporal empilhada | A qualidade melhora ou piora ciclo a ciclo? |

#### Camada 3 — Detalhe Operacional (Tabela)

| KPI | Fonte Gold | Tipo Visual | Decisão que suporta |
|---|---|---|---|
| **Lista de SKUs Rejeitados** | `rejeitados_produtos_atual` | Tabela com filtro e ordenação | Ação pontual — corrigir SKU por SKU na origem |
| **Performance por Categoria (completa)** | `categoria_performance_atual` | Tabela com heatmap condicional | Drill-down — qual categoria precisa de intervenção |

---

## ETAPA 2: ARQUITETURA & LAYOUT DO DASHBOARD

### 2.1 Wireframe Lógico — 4 Páginas

```
┌─────────────────────────────────────────────────────────────┐
│  PÁGINA 1: VISÃO EXECUTIVA (o C-Level abre aqui)           │
│                                                             │
│  ┌─── Cabeçalho (#1F4E8C) ────────── [Filtro Período] ──┐  │
│  │  🏪 MercaData — Catálogo de Produtos                  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
│  │Valor   │ │Margem  │ │SKUs    │ │Preço   │ │Quaren- │   │
│  │Lista   │ │Média   │ │Ativos  │ │Inválido│ │tena    │   │
│  │R$ 129K │ │41,74%  │ │100%    │ │0       │ │319     │   │
│  │(#D5E8F0)│(#D5E8F0)│(#D5E8F0)│(#D5E8F0)│(#D5E8F0)│   │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘   │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐        │
│  │ Pareto de Valor por  │  │ Diagnóstico por      │        │
│  │ Departamento (barras)│  │ Categoria (rosca)    │        │
│  │ #1F4E8C              │  │ paleta categórica    │        │
│  └──────────────────────┘  └──────────────────────┘        │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Alertas do Ciclo (cards condicionais)        │          │
│  │ 🔴 Margem negativa  🟡 Catálogo defasado     │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  PÁGINA 2: ANÁLISE DE CATEGORIAS & MARGEM                  │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐        │
│  │ Margem por Depto     │  │ Faixa de Preço       │        │
│  │ (barras horizontais) │  │ (barras empilhadas)  │        │
│  └──────────────────────┘  └──────────────────────┘        │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Tabela: Performance por Categoria            │          │
│  │ Cols: nome | skus_ativos | valor | margem%   │          │
│  │       | classe | diagnóstico                 │          │
│  │ Heatmap condicional em margem (verde→vermelho)│         │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  PÁGINA 3: EVOLUÇÃO TEMPORAL                                │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Evolução: SKUs Ativos + Valor Lista (2 eixos)│          │
│  │ Linha #1F4E8C (SKUs) + #5B8FC9 (Valor)      │          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Evolução: Rejeições por Regra (área empilhada)│         │
│  │ Paleta categórica                             │          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Evolução: Share de Valor das Top 5 Categorias│          │
│  │ (linha, por ciclo)                            │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  PÁGINA 4: QUARENTENA & QUALIDADE DE DADOS                 │
│                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐                          │
│  │Taxa de │ │SKUs sem│ │Rejeiç. │                          │
│  │Rejeição│ │versão  │ │24h     │                          │
│  └────────┘ └────────┘ └────────┘                          │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐        │
│  │ Rejeições por Regra  │  │ Rejeições por Tabela │        │
│  │ (barras horizontais) │  │ (barras horizontais) │        │
│  └──────────────────────┘  └──────────────────────┘        │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Tabela: SKUs Rejeitados (drill-down)         │          │
│  │ Cols: sku | preço_rejeitado | motivo         │          │
│  │       | tem_versão_válida | preço_vigente    │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Guia de UI/UX — Regras da Padronização

| Elemento | Especificação | Referência |
|---|---|---|
| **Cabeçalho** | Faixa `#1F4E8C`, texto branco, filtros à direita | Padronização §3.1 |
| **Fundo geral** | `#F7F7F7` | Padronização §2 |
| **Cards KPI** | Fundo `#D5E8F0`, número negrito tamanho grande, rótulo abaixo em `#262626` | Padronização §3.2 |
| **Barras** | Primária `#1F4E8C`, secundária `#5B8FC9` | Padronização §4.1 |
| **Pizza/Rosca** | Verde `#2E7D32` (Saudável), Amarelo `#F2B705` (Alerta), Vermelho `#C0392B` (Erro), Cinza `#8C8C8C` (Neutro) | Padronização §4.2 |
| **Linhas temporais** | Principal `#1F4E8C`, meta/referência cinza tracejado | Padronização §4.3 |
| **Fonte** | Arial (padrão Looker), títulos negrito | Padronização §5 |
| **Eixos/rótulos** | Cinza escuro `#595959`, grades leves | Padronização §4.1 |
| **Texto** | Branco em fundo escuro, `#262626` em fundo claro | Padronização §2 |

### 2.3 Mapeamento: Diagnóstico → Cor Categórica

| `diagnostico_codigo` | Significado | Cor | Hex |
|---|---|---|---|
| `SAUDAVEL` | Sem ação necessária | Verde | `#2E7D32` |
| `PRECO_HETEROGENEO` | Preços muito dispersos | Amarelo | `#F2B705` |
| `CLASSE_A_COM_INATIVOS` | Categoria de valor com inativos | Amarelo | `#F2B705` |
| `MARGEM_NEGATIVA` | Vendendo abaixo do custo | Vermelho | `#C0392B` |
| `SEM_OFERTA_ATIVA` | Nenhum SKU ativo | Vermelho | `#C0392B` |
| `SEM_SKU` | Categoria vazia | Cinza | `#8C8C8C` |

---

## ETAPA 3: PASSO A PASSO DE CONSTRUÇÃO NO LOOKER STUDIO

### 3.1 Fontes de Dados (SQL Server)

Conectar **8 data sources** no Looker Studio, todas no SQL Server (conector Microsoft SQL Server):

| # | Tabela SQL Server | Granularidade | Uso no Dashboard |
|---|---|---|---|
| 1 | `gold_kpi_catalogo_resumo_atual` | 1 linha (ciclo atual) | Scorecards Pág. 1 |
| 2 | `gold_kpi_categoria_performance_atual` | 1 por categoria | Tabela + gráficos Pág. 2 |
| 3 | `gold_kpi_categoria_raiz_atual` | 1 por departamento | Pareto + Margem Pág. 1-2 |
| 4 | `gold_kpi_faixa_preco_atual` | depto × faixa | Distribuição Pág. 2 |
| 5 | `gold_kpi_qualidade_dados_atual` | tabela × regra | Barras Pág. 4 |
| 6 | `gold_kpi_rejeitados_produtos_atual` | 1 por SKU rejeitado | Tabela Pág. 4 |
| 7 | `gold_hist_catalogo_resumo` | 1 por ciclo (histórico) | Linhas temporais Pág. 3 |
| 8 | `gold_hist_qualidade_dados` | tabela × regra × ciclo | Evolução qualidade Pág. 3 |

> **CRÍTICO**: As tabelas `_atual` têm só o ciclo corrente — **NÃO precisam de filtro** por `ciclo_id`. As tabelas `gold_hist_*` acumulam ciclos — use `ciclo_ts` como dimensão temporal.

### 3.2 Renomeação de Campos no Looker Studio

Para que o C-Level leia sem precisar de dicionário:

| Campo Gold (original) | Nome no Looker Studio |
|---|---|
| `total_skus` | Total de SKUs |
| `skus_ativos` | SKUs Ativos |
| `pct_skus_ativos` | % SKUs Ativos |
| `valor_lista_ativo` | Valor de Lista Ativo (R$) |
| `margem_pct_media_ativos` | Margem Média (%) |
| `skus_ativos_preco_invalido` | SKUs c/ Preço Inválido |
| `taxa_rejeicao_produtos_pct` | Taxa de Rejeição (%) |
| `indice_hhi_valor` | Índice de Concentração (HHI) |
| `pct_valor_top3_categorias` | Concentração Top 3 (%) |
| `skus_sem_versao_valida` | SKUs Fora do Catálogo |
| `skus_margem_negativa_ou_zero` | SKUs c/ Margem Negativa |
| `horas_desde_ultima_ingestao` | Horas desde Última Atualização |
| `nome_categoria_raiz` | Departamento |
| `classe_valor_catalogo` | Classe de Valor |
| `diagnostico_codigo` | Diagnóstico |
| `diagnostico_acao` | Ação Sugerida |
| `share_valor_pct` | Participação no Valor (%) |
| `margem_valor_ativo` | Margem Bruta (R$) |
| `faixa_preco` | Faixa de Preço |
| `qtd_rejeicoes` | Rejeições |
| `motivo_rejeicao` | Motivo da Rejeição |
| `preco_rejeitado` | Preço Rejeitado |
| `preco_vigente_silver` | Preço Vigente |
| `tem_versao_valida_na_silver` | Tem Versão Válida? |

### 3.3 Campos Calculados Necessários

#### No data source `catalogo_resumo_atual`:

**1. `pct_skus_ativos` (exibição correta como %)**
- Problema: Gold armazena na escala 0–100 (ex: `94.2`). Se tipo = Porcentagem no Looker, vira `9420%`.
- Solução A (campo calculado): `pct_skus_ativos / 100` → Tipo = Porcentagem
- Solução B (sem campo calculado): Tipo = Número, formato personalizado `0.00"%"`
- **Recomendação**: Solução B (menos overhead, mesma leitura visual)

**2. `margem_pct_media_ativos` (mesmo problema)**
- Mesma lógica: Tipo = Número, formato `0.00"%"`

**3. Todos os campos `pct_*` e `*_pct` seguem o mesmo padrão** (Gold grava em 0–100):
- `taxa_rejeicao_produtos_pct`
- `pct_valor_top3_categorias`
- `pct_skus_ativos_com_custo_informado`
- `share_valor_pct` (em `categoria_raiz_atual` e `categoria_performance_atual`)
- `pct_skus_no_escopo` e `pct_valor_no_escopo` (em `faixa_preco_atual`)

> **REGRA UNIVERSAL**: Todo campo `%` da Gold → Looker = Tipo **Número**, formato `0.00"%"`, agregação **Nenhum** (ou Soma se for única linha).

#### No data source `categoria_performance_atual`:

**4. `cor_diagnostico` (para formatação condicional)**
```
CASE
  WHEN diagnostico_codigo = "SAUDAVEL" THEN "#2E7D32"
  WHEN diagnostico_codigo IN ("PRECO_HETEROGENEO", "CLASSE_A_COM_INATIVOS") THEN "#F2B705"
  WHEN diagnostico_codigo IN ("MARGEM_NEGATIVA", "SEM_OFERTA_ATIVA") THEN "#C0392B"
  ELSE "#8C8C8C"
END
```

### 3.4 Configuração — Página 1: Visão Executiva

#### Cabeçalho
1. Retângulo: largura total, altura ~80px, cor `#1F4E8C`
2. Texto à esquerda: "MercaData — Catálogo de Produtos" | Arial 18pt, negrito, branco
3. Controle de período à direita (se usar `gold_hist_*` em algum gráfico desta página): Filtro de intervalo de datas sobre `ciclo_ts`

#### Scorecards (5 cards, largura uniforme)

| Card | Data Source | Campo | Tipo Dado | Agregação | Formato | Comparação |
|---|---|---|---|---|---|---|
| Valor de Lista Ativo | `catalogo_resumo_atual` | `valor_lista_ativo` | Número | Soma | `R$ #,##0` | `delta_valor_lista_ativo` como métrica de comparação (Número, Nenhum cálculo) |
| Margem Média | `catalogo_resumo_atual` | `margem_pct_media_ativos` | Número | Soma | `0.00"%"` | `delta_margem_pct_media_ativos` como métrica de comparação |
| SKUs Ativos | `catalogo_resumo_atual` | `pct_skus_ativos` | Número | Soma | `0.00"%"` | — |
| SKUs c/ Preço Inválido | `catalogo_resumo_atual` | `skus_ativos_preco_invalido` | Número | Soma | `#,##0` | — |
| SKUs em Quarentena | `rejeitados_produtos_atual` | — | — | **Contagem de registros** | `#,##0` | — |

> **Atenção "SKUs em Quarentena"**: Diferente dos demais, este card usa a **tabela `rejeitados_produtos_atual`** (não `catalogo_resumo_atual`), com agregação **Contagem de registros** (COUNT), não Soma de um campo. Isso porque `gold_kpi_rejeitados_produtos_atual` tem 1 linha por SKU rejeitado — contar as linhas = contar os SKUs.

Para cada card:
- Background: `#D5E8F0`
- Número: Arial 28pt, negrito, `#262626`
- Rótulo: Arial 12pt, regular, `#262626`
- Comparação (quando existir): seta ▲/▼, verde se positivo (exceto Preço Inválido, onde verde = diminuiu)

#### Gráfico: Pareto de Valor por Departamento
- **Data source**: `gold_kpi_categoria_raiz_atual`
- **Tipo**: Gráfico de barras horizontais
- **Dimensão**: `nome_categoria_raiz` (renomear → "Departamento")
- **Métrica**: `valor_lista_ativo` (Soma)
- **Ordenação**: `valor_lista_ativo` DESC
- **Cor das barras**: `#1F4E8C`
- **Rótulos de dados**: ativados, formato `R$ #,##0`
- **Limite de barras**: Top 10 (agrupar restante em "Outros")

#### Gráfico: Diagnóstico por Categoria (Rosca)
- **Data source**: `gold_kpi_categoria_performance_atual`
- **Tipo**: Gráfico de rosca
- **Dimensão**: `diagnostico_codigo`
- **Métrica**: Contagem de registros (= número de categorias por diagnóstico)
- **Cores fixas** (atribuição manual):
  - `SAUDAVEL` → `#2E7D32`
  - `PRECO_HETEROGENEO` → `#F2B705`
  - `CLASSE_A_COM_INATIVOS` → `#F2B705`
  - `MARGEM_NEGATIVA` → `#C0392B`
  - `SEM_OFERTA_ATIVA` → `#C0392B`
  - `SEM_SKU` → `#8C8C8C`
- **Legenda**: visível, à direita
- **Percentual**: exibir no slice

#### Bloco de Alertas
Campo calculado sobre `catalogo_resumo_atual`:
```
CASE
  WHEN alerta_margem_negativa = true THEN "🔴 SKU(s) vendendo abaixo do custo"
  WHEN alerta_preco_ativo_invalido = true THEN "🟡 SKU ativo com preço ≤ 0 na quarentena"
  WHEN alerta_catalogo_defasado = true THEN "🟡 Catálogo sem atualização há mais de 24h"
  WHEN alerta_novos_skus = true THEN "🟡 Volume atípico de SKUs novos"
  ELSE "✅ Sem alertas neste ciclo"
END
```
Exibir como **Scorecard** de texto, sem agregação numérica.

### 3.5 Configuração — Página 2: Categorias & Margem

#### Gráfico: Margem por Departamento
- **Data source**: `gold_kpi_categoria_raiz_atual`
- **Tipo**: Barras horizontais
- **Dimensão**: `nome_categoria_raiz`
- **Métrica**: `margem_pct_media_ativos` (Soma, formato `0.00"%"`)
- **Ordenação**: `margem_pct_media_ativos` DESC
- **Cor**: `#1F4E8C`
- **Linha de referência**: 0% (identifica margem negativa visualmente)

#### Gráfico: Distribuição por Faixa de Preço
- **Data source**: `gold_kpi_faixa_preco_atual`
- **Filtro fixo**: `nome_categoria_raiz = "TODO O CATÁLOGO"` (para visão global; ou remover filtro e usar `nome_categoria_raiz` como breakdown)
- **Tipo**: Barras verticais
- **Dimensão**: `faixa_preco` (ordenar por `faixa_ordem` ASC)
- **Métrica**: `qtd_skus_ativos` (Soma)
- **Cor**: `#1F4E8C`

#### Tabela: Performance por Categoria
- **Data source**: `gold_kpi_categoria_performance_atual`
- **Colunas visíveis** (nesta ordem):

| Coluna | Formato | Nota |
|---|---|---|
| `nome_categoria` | Texto | — |
| `nome_categoria_raiz` | Texto | Departamento pai |
| `classe_valor_catalogo` | Texto | A / B / C |
| `qtd_skus_ativos` | Número `#,##0` | — |
| `valor_lista_ativo` | Número `R$ #,##0` | — |
| `share_valor_pct` | Número `0.00"%"` | — |
| `margem_pct_media_ativos` | Número `0.00"%"` | Heatmap: verde > 20%, amarelo 0–20%, vermelho < 0% |
| `diagnostico_codigo` | Texto | Cor condicional (mapa acima) |
| `diagnostico_acao` | Texto | Ação recomendada |

- **Ordenação padrão**: `ranking_valor` ASC (= maior valor primeiro)
- **Linhas por página**: 20
- **Heatmap condicional** em `margem_pct_media_ativos`:
  - Regra no Looker: Estilo > Formatação condicional > Escala de cores
  - Min (vermelho `#C0392B`) = -10, Meio (amarelo `#F2B705`) = 0, Max (verde `#2E7D32`) = 40

### 3.6 Configuração — Página 3: Evolução Temporal

#### Gráfico: Evolução do Catálogo
- **Data source**: `gold_hist_catalogo_resumo`
- **Tipo**: Gráfico de linhas com 2 eixos Y (combo)
- **Dimensão temporal**: `ciclo_ts`
- **Eixo Y esquerdo**: `skus_ativos` (Soma) — linha `#1F4E8C`
- **Eixo Y direito**: `valor_lista_ativo` (Soma) — linha `#5B8FC9`
- **Marcadores de dados**: ativados
- **Suavização**: desativada (dados discretos por ciclo)

#### Gráfico: Evolução da Qualidade
- **Data source**: `gold_hist_qualidade_dados`
- **Tipo**: Gráfico de área empilhada
- **Dimensão temporal**: `ciclo_ts`
- **Breakdown**: `regra`
- **Métrica**: `qtd_rejeicoes` (Soma)
- **Cores**: paleta categórica (automática, sem necessidade de fixar — cada regra é uma série)

#### Gráfico: Share de Valor Top 5 Categorias
- **Data source**: `gold_hist_categoria_share` (se existir no SQL Server; senão, criar view/tabela)
- **Filtro**: Top 5 por `valor_lista_ativo` DESC no ciclo mais recente
- **Tipo**: Linhas
- **Dimensão temporal**: `ciclo_ts`
- **Breakdown**: `nome_categoria`
- **Métrica**: `share_valor_pct` (Soma)

### 3.7 Configuração — Página 4: Quarentena & Qualidade

#### Scorecards (3 cards)

| Card | Data Source | Campo | Formato |
|---|---|---|---|
| Taxa de Rejeição | `catalogo_resumo_atual` | `taxa_rejeicao_produtos_pct` | `0.00"%"` |
| SKUs Fora do Catálogo | `catalogo_resumo_atual` | `skus_sem_versao_valida` | `#,##0` |
| Rejeições 24h | `catalogo_resumo_atual` | `rejeicoes_janela` | `#,##0` |

#### Gráfico: Rejeições por Regra
- **Data source**: `gold_kpi_qualidade_dados_atual`
- **Tipo**: Barras horizontais
- **Dimensão**: `regra`
- **Métrica**: `qtd_rejeicoes` (Soma)
- **Cor**: `#1F4E8C`
- **Métrica secundária** (opcional): `qtd_chaves_sem_versao_valida` em `#C0392B`

#### Tabela: SKUs Rejeitados
- **Data source**: `gold_kpi_rejeitados_produtos_atual`
- **Colunas**:

| Coluna | Formato |
|---|---|
| `sku` | Texto |
| `preco_rejeitado` | Número `R$ #,##0.00` |
| `motivo_rejeicao` | Texto |
| `tem_versao_valida_na_silver` | Booleano (Sim/Não) |
| `preco_vigente_silver` | Número `R$ #,##0.00` |
| `data_rejeicao` | Data/hora |

- **Ordenação**: `data_rejeicao` DESC (mais recentes primeiro)
- **Formatação condicional**: `tem_versao_valida_na_silver` = false → fundo `#C0392B` com texto branco

### 3.8 Filtros Globais

| Filtro | Tipo | Aplica em | Nota |
|---|---|---|---|
| Período (`ciclo_ts`) | Intervalo de datas | Páginas 3 (histórico) | Não se aplica a `_atual` (já são ciclo corrente) |
| Departamento (`nome_categoria_raiz`) | Lista suspensa | Páginas 1, 2, 4 | Cross-filter entre `categoria_raiz`, `categoria_performance`, `faixa_preco` |
| Classe de Valor (`classe_valor_catalogo`) | Lista suspensa | Página 2 | Filtrar A/B/C |
| Diagnóstico (`diagnostico_codigo`) | Lista suspensa | Página 2 | Filtrar categorias com problema |

---

## CHECKLIST DE VALIDAÇÃO / AUDITORIA

Antes de apresentar ao cliente exigente, validar cada item:

### Números

| # | Check | Como validar | Criticidade |
|---|---|---|---|
| 1 | `total_skus` no card = soma de `skus_ativos` + `skus_inativos` | Scorecard `total_skus` = `skus_ativos` + `skus_inativos` na `catalogo_resumo_atual` | 🔴 Alta |
| 2 | `pct_skus_ativos` = `skus_ativos / total_skus × 100` | Calcular manualmente e comparar com o card | 🔴 Alta |
| 3 | Soma de `share_valor_pct` em `categoria_raiz_atual` ≈ 100% | Query SQL: `SELECT SUM(share_valor_pct) FROM gold_kpi_categoria_raiz_atual` | 🔴 Alta |
| 4 | Contagem de registros em `rejeitados_produtos_atual` = card "SKUs em Quarentena" | Query SQL: `SELECT COUNT(*) FROM gold_kpi_rejeitados_produtos_atual` | 🔴 Alta |
| 5 | `margem_pct_media_ativos` não é NULL quando `skus_ativos_com_custo_informado > 0` | Query SQL na `catalogo_resumo_atual` | 🟡 Média |
| 6 | Nenhum `diagnostico_codigo` fora do conjunto válido | Query: `SELECT DISTINCT diagnostico_codigo FROM gold_kpi_categoria_performance_atual` | 🟡 Média |

### Visuais

| # | Check | Como validar |
|---|---|---|
| 7 | Valores `%` não estão multiplicados por 100 duas vezes | Comparar card com valor bruto do SQL: `SELECT margem_pct_media_ativos FROM gold_kpi_catalogo_resumo_atual` → deve ser ~41.74, não ~4174 |
| 8 | Pareto de departamentos ordena DESC por valor | Olhar visualmente se a maior barra está no topo |
| 9 | Rosca de diagnósticos usa cores fixas corretas | Verde para SAUDÁVEL, vermelho para MARGEM_NEGATIVA |
| 10 | Filtro de período NÃO afeta cards da Página 1 (são `_atual`) | Mudar o filtro e confirmar que scorecards não mudam |
| 11 | Heatmap de margem mostra vermelho para negativos | Inserir dado de teste com margem < 0 e confirmar cor |

### Integridade

| # | Check | Como validar |
|---|---|---|
| 12 | `_atual` tem exatamente 1 `ciclo_id` | `SELECT DISTINCT ciclo_id FROM gold_kpi_catalogo_resumo_atual` → 1 linha |
| 13 | `gold_hist_*` acumula ciclos sem duplicata | `SELECT ciclo_id, COUNT(*) FROM gold_hist_catalogo_resumo GROUP BY ciclo_id HAVING COUNT(*) > 1` → 0 linhas |
| 14 | Campos de margem existem (post `preco_custo`) | `SELECT margem_pct_media_ativos FROM gold_kpi_catalogo_resumo_atual` → não nulo |

---

## APÊNDICE: CAMPOS DISPONÍVEIS POR DATA SOURCE

### `gold_kpi_catalogo_resumo_atual` (1 linha)

```
ciclo_id, ciclo_ts, janela_horas,
total_skus, skus_ativos, skus_inativos, pct_skus_ativos,
preco_medio_ativos, preco_mediano_ativos, preco_p90_ativos,
valor_lista_ativo, valor_lista_inativo,
total_categorias, categorias_raiz, categorias_com_sku_ativo, categorias_sem_sku_ativo,
pct_categorias_com_oferta,
skus_sem_categoria_valida, categorias_hierarquia_invalida,
pct_valor_top3_categorias, indice_hhi_valor,
skus_atualizados_janela, horas_desde_ultima_ingestao,
skus_sem_versao_valida, skus_ativos_preco_invalido, rejeicoes_janela,
taxa_rejeicao_produtos_pct,
delta_skus, delta_skus_ativos, delta_valor_lista_ativo,
delta_preco_medio_ativos_pct, delta_categorias_raiz,
alerta_novos_skus, alerta_categorias_raiz_mudou,
alerta_preco_ativo_invalido, alerta_catalogo_defasado,
-- Campos de margem (condicional a preco_custo existir):
skus_ativos_com_custo_informado, pct_skus_ativos_com_custo_informado,
custo_total_ativo, margem_valor_ativo_total, margem_pct_media_ativos,
skus_margem_negativa_ou_zero, alerta_margem_negativa,
delta_margem_valor_ativo_total, delta_margem_pct_media_ativos
```

### `gold_kpi_categoria_performance_atual` (1 por categoria)

```
ciclo_id, ciclo_ts,
id_categoria, nome_categoria, id_categoria_pai,
id_categoria_raiz, nome_categoria_raiz, nivel, hierarquia_status,
qtd_skus, qtd_skus_ativos, qtd_skus_inativos, pct_skus_ativos,
preco_medio_ativos, preco_mediano_ativos, preco_min_ativos, preco_max_ativos,
desvio_padrao_preco_ativos, valor_lista_ativo, valor_lista_inativo,
coef_variacao_preco, share_skus_ativos_pct, share_valor_pct, ranking_valor,
classe_valor_catalogo, diagnostico_codigo, diagnostico_acao,
-- Margem (condicional):
qtd_skus_ativos_com_custo, pct_skus_ativos_com_custo,
custo_total_ativo, valor_lista_ativo_com_custo, margem_valor_ativo,
margem_pct_media_ativos, ranking_margem
```

### `gold_kpi_categoria_raiz_atual` (1 por departamento)

```
ciclo_id, ciclo_ts,
id_categoria_raiz, nome_categoria_raiz,
qtd_categorias, qtd_subcategorias, profundidade_maxima,
qtd_skus, qtd_skus_ativos, qtd_skus_inativos, pct_skus_ativos,
categorias_sem_sku_ativo, pct_categorias_com_oferta,
preco_medio_ativos, preco_mediano_ativos, preco_p90_ativos,
valor_lista_ativo, valor_lista_inativo,
share_skus_ativos_pct, share_valor_pct, ranking_valor,
-- Margem (condicional):
qtd_skus_ativos_com_custo, pct_skus_ativos_com_custo,
custo_total_ativo, margem_valor_ativo, margem_pct_media_ativos, ranking_margem
```

### `gold_kpi_faixa_preco_atual` (departamento × faixa)

```
ciclo_id, ciclo_ts,
id_categoria_raiz, nome_categoria_raiz,
faixa_ordem, faixa_preco,
qtd_skus_ativos, pct_skus_no_escopo,
valor_lista_ativo, pct_valor_no_escopo
```

### `gold_kpi_qualidade_dados_atual` (tabela × regra)

```
ciclo_id, ciclo_ts,
tabela, regra,
qtd_rejeicoes, qtd_chaves_afetadas,
qtd_chaves_sem_versao_valida, qtd_rejeicoes_janela
```

### `gold_kpi_rejeitados_produtos_atual` (1 por SKU rejeitado)

```
ciclo_id, ciclo_ts,
sku, id_categoria,
preco_rejeitado, is_ativo, motivo_rejeicao,
arquivo_origem, data_ingestao, data_rejeicao,
preco_vigente_silver, tem_versao_valida_na_silver
```
