# 💰 Financial Ops Platform

A complete financial intelligence platform — cash flow analysis, income statement (P&L), banking analytics, and financial cycle diagnostics.

Built to turn raw ERP transaction data into actionable financial visibility — with sub-second response on 2,200+ records across 12 periods.

> ⚠️ This repository contains documentation and sanitized code samples only.  
> The production system runs on private infrastructure connected to a live ERP database.

---

## 📸 Screenshots

### Cash Flow Dashboard (DFC)
![Cash Flow Dashboard](screenshots/dfc-dashboard.png)

### Receipts & Payments Chart
![Receipts and Payments](screenshots/receipts-payments-chart.png)

### Hierarchical Cash Flow Table
![Hierarchical Table](screenshots/hierarchical-table.png)

---

## 🧠 System Architecture

```
ERP Database (PostgreSQL)
        │
        ├── Payments & Receivables ───────────────────┐
        ├── Bank transactions ────────────────────────┤
        ├── Cost categories ──────────────────────────┼──▶ Materialized Views (daily cache)
        └── Invoice records ──────────────────────────┘              │
                                                                      │  10–150x faster
                                                                      ▼
                                                             Python FastAPI Backend
                                                                      │
                                                    ┌─────────────────┼─────────────────┐
                                                    ▼                 ▼                 ▼
                                               DFC Engine       DRE Engine      Cycle Analytics
                                          (Cash Flow)          (P&L)          (PMR · PMP · CCE)
                                                    └─────────────────┼─────────────────┘
                                                                      ▼
                                                             Next.js Dashboard
                                                      (Charts · Hierarchy · Filters)
```

---

## 📁 Project Structure

```
financial-ops-platform/
├── python-backend/
│   ├── main.py                    # FastAPI app and routes
│   ├── services.py                # Business logic and SQL queries
│   ├── database.py                # PostgreSQL connection pool
│   └── refresh_materialized_views.py  # Daily cache refresh script
├── sql_scripts/
│   ├── 10_create_materialized_views.sql   # Initial MV setup
│   └── 11_refresh_materialized_views.sql  # Refresh script
├── app/                           # Next.js frontend
│   ├── components/
│   │   ├── DFCTableDiaria.tsx     # 4-level hierarchical table
│   │   ├── DateFilters.tsx        # Dynamic period filters
│   │   ├── EvolutionChart.tsx     # 6-month area chart
│   │   └── CycleMetrics.tsx       # PMR · PMP · CCE cards
│   ├── hooks/
│   │   ├── useDFC.ts              # Cash flow data hook
│   │   └── useDashboard.ts        # Dashboard metrics hook
│   └── utils/
│       └── formatters.ts          # BRL formatting · conditional colors
└── scripts/
    └── refresh_views.sh           # Cron wrapper with logging
```

---

## ⚙️ Core Logic

### 1. Materialized Views — Performance Architecture

Replaced slow views with a daily cache layer, achieving **10–150x query performance improvement**.

```sql
-- Create materialized view with unique index for concurrent refresh
CREATE MATERIALIZED VIEW mv_fluxo_pagamentos AS
SELECT
    origem_tabela,
    faturaduplicata,
    nrparcela,
    idempresa,
    dtvencimento,
    vlrliquido,
    categoria,
    grupo,
    atividade
FROM vw_fluxo_pagamentos;

-- Unique index required for CONCURRENTLY refresh (zero downtime)
CREATE UNIQUE INDEX idx_mv_fluxo_pag_unique
ON mv_fluxo_pagamentos(origem_tabela, faturaduplicata, nrparcela, idempresa);
```

**Performance comparison:**

| Operation | Slow View | Materialized View | Gain |
|---|---|---|---|
| Simple SELECT | 2–5s | 0.05–0.2s | ~25x |
| SELECT with JOIN | 5–15s | 0.1–0.5s | ~50x |
| Aggregations | 10–30s | 0.2–1s | ~100x |

**Automated daily refresh via cron:**
```bash
# Refresh at 00:30 daily — after ERP daily close
30 0 * * * /path/to/refresh_views.sh >> /logs/cron.log 2>&1
```

```python
# Concurrent refresh — queries keep running during update
def refresh_materialized_views():
    views = [
        "mv_fluxo_pagamentos",
        "mv_despesas_historico",
        "mv_recebimentos_historico",
        "mv_despesas_abertas"
    ]
    for view in views:
        execute_query(f"REFRESH MATERIALIZED VIEW CONCURRENTLY {view}")
        log(f"✅ {view} refreshed at {datetime.now()}")
```

### 2. Financial Cycle Analytics

Automatic calculation of PMR, PMP, and Cash Conversion Cycle with health status classification.

```python
def calcular_ciclo_financeiro(pmr: float, pmp: float) -> dict:
    ciclo_operacional = pmr + prazo_estoque     # Days tied up in operations
    ciclo_financeiro = ciclo_operacional - pmp  # Net cash conversion cycle

    # Status classification
    if ciclo_financeiro <= 0:
        status = "BOM"        # Company receives before it pays — ideal
    elif ciclo_financeiro <= 30:
        status = "ATENCAO"    # Short cycle — manageable
    else:
        status = "CRITICO"    # Long cycle — cash flow pressure

    return {
        "pmr": round(pmr, 1),
        "pmp": round(pmp, 1),
        "ciclo_operacional": round(ciclo_operacional, 1),
        "ciclo_financeiro": round(ciclo_financeiro, 1),
        "status": status
    }
```

### 3. 4-Level Hierarchical Data Model

```
Activities (RECEBIMENTOS / PAGAMENTOS)
└── Groups (RECEITAS OPERACIONAIS / ATIVIDADES OPERACIONAIS)
    └── Categories (FATURA / CARTÃO CRÉDITO / PIX / FORNECEDORES)
        └── Individual Entries (invoice-level records)
```

```typescript
// Recursive tree builder — handles arbitrary depth
function buildHierarchy(entries: Entry[]): HierarchyNode[] {
    const grouped = groupBy(entries, 'atividade');

    return Object.entries(grouped).map(([atividade, items]) => ({
        label: atividade,
        total: sumBy(items, 'vlrliquido'),
        periodos: aggregateByPeriod(items),
        children: buildGroupLevel(items),  // recurse
        expanded: false
    }));
}
```

### 4. Dual Synchronized Scroll

One of the UX challenges: a wide table with fixed columns and two scrollbars (top + table) that stay in sync.

```typescript
// Synchronized horizontal scroll — top bar + table body
const handleScroll = (source: 'top' | 'table') => (e: React.UIEvent) => {
    const scrollLeft = e.currentTarget.scrollLeft;

    if (source === 'top' && tableRef.current) {
        tableRef.current.scrollLeft = scrollLeft;
    }
    if (source === 'table' && topScrollRef.current) {
        topScrollRef.current.scrollLeft = scrollLeft;
    }
};
```

---

## 🔌 API Reference

```
GET  /api/dfc                          # Cash flow data (hierarchical)
GET  /api/dre                          # P&L income statement
GET  /api/dashboard/metricas           # PMR · PMP · cycle metrics
GET  /api/dashboard/evolucao           # 6-month evolution chart data
GET  /api/saldos-bancarios             # Bank balance summary
GET  /api/recebimentos-futuros         # Upcoming receivables
```

**Query parameters for `/api/dfc`:**

| Parameter | Type | Description |
|---|---|---|
| `dataInicio` | string | Start date (YYYY-MM-DD) |
| `dataFim` | string | End date (YYYY-MM-DD) |
| `tipoData` | string | `emissao` · `vencimento` · `baixa` |
| `agrupamento` | string | `diario` · `mensal` |

**Example response structure:**
```json
{
  "periodos": [{ "key": "2025-01-01", "label": "Jan/25" }],
  "recebimentos": {
    "data": [
      {
        "label": "RECEITAS OPERACIONAIS",
        "total": 7044521.21,
        "periodos": { "2025-01-01": 7044521.21 },
        "children": [...]
      }
    ],
    "totais": { "total": 73815846.80 }
  },
  "cicloFinanceiro": {
    "pmr": 51.8,
    "pmp": 29.4,
    "cicloOperacional": 51.8,
    "cicloFinanceiro": 22.4,
    "status": "BOM"
  },
  "metadata": { "totalRegistros": 2230, "periodos": 12 }
}
```

---

## 📊 Materialized Views Created

| Original View | Materialized View | Description |
|---|---|---|
| `vw_fluxo_pagamentos` | `mv_fluxo_pagamentos` | Payments — payables, orders, suppliers |
| `vw_despesas_historico` | `mv_despesas_historico` | Paid expenses (2024–2026) for PMP calc |
| `vw_recebimentos_historico` | `mv_recebimentos_historico` | Received payments (2024–2026) for PMR calc |
| `vw_despesas_abertas` | `mv_despesas_abertas` | Open payables for DFC projection |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 · TypeScript · Tailwind CSS · Recharts |
| Backend | Python · FastAPI · Uvicorn |
| Database | PostgreSQL · psycopg2 · Materialized Views · Connection pooling |
| Scheduling | Cron · Shell scripting · Structured logging |
| Deploy | Vercel (frontend) · Render (backend) |

---

## 🔐 Notes

- All credentials managed via environment variables — never committed
- Custom PostgreSQL functions (`f_dic_pes_nome`, `f_dic_sld_prd_produto`) abstracted in service layer
- Production financial data and company-specific identifiers removed from this public version
