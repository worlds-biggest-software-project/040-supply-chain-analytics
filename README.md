# Supply Chain Analytics

**Candidate #40** — An AI-native supply chain planning platform with modern demand forecasting, agentic disruption response, and supplier risk scoring for mid-market manufacturers and distributors.

## Market Opportunity

The supply chain analytics market is **$11 billion in 2025–2026**, projected to reach **$32–56 billion by 2032–2035 at 16–18% CAGR**. The market is dominated by Blue Yonder ($600M ARR), Kinaxis ($483M ARR), and SAP IBP, all with custom enterprise pricing and 18–24 month implementations.

**Market dynamics:**
- 87% of enterprises report using AI for demand forecasting (AllAboutAI, 2025)
- AI-driven inventory optimization reduces stockouts by ~28% on average
- Fivetran + dbt Labs merger creates an integrated ingestion-transformation powerhouse
- Mid-market accessibility gap: frePPLe is the only open-source option and it lacks modern ML/DL

## What This Platform Solves

An **AI-native supply chain planning engine** combining modern ML-based demand forecasting, agentic disruption response, and supplier risk scoring—designed for mid-market manufacturers and distributors (not enterprises).

**Core value proposition:**
- **Modern ML/DL forecasting**: LSTM, transformer temporal fusion models + external signals (weather, social trends)
- **Agentic disruption response**: AI agents monitor supplier news and supply conditions; autonomously generate re-sourcing recommendations
- **Supplier risk scoring**: LLM-parsed ESG disclosures, financial filings, geopolitical risk + structured on-time delivery metrics
- **Real-time S&OP narratives**: Auto-generated variance explanations from planning data (replaces manual slide building)
- **Self-hostable**: Full data sovereignty; no cloud-only lock-in
- **Affordable**: Free self-hosted; cloud tier from $500–$2K/month

## Competitive Differentiation

| Aspect | This Platform | Blue Yonder | Kinaxis | o9 | frePPLe |
|--------|---------------|-----------|---------|-----|---------|
| **ML/DL Forecasting** | Yes (LSTM, TFT) | Yes | Yes | Yes | Limited |
| **Agentic Disruption** | Yes | Partial | Partial | Partial | No |
| **Supplier Risk** | Yes (LLM+structured) | Partial | No | No | No |
| **Open Source** | Yes (MIT) | No | No | No | Yes (MIT) |
| **Self-Hostable** | Yes | No | No | No | Yes |
| **Price** | Free/Cloud | $100K+/yr | Custom | Custom | Free/custom |
| **Implementation** | Weeks | 18–24 mo | 12–18 mo | 3–6 mo | 4–12 wk |

## Key Features

### Must-Have (MVP)
- ML-based demand forecasting (at minimum gradient boosting; ideally transformer-based)
- Multi-location inventory visibility with safety stock calculation
- Supply plan generation from forecasts with constraint handling
- ERP data ingest via CSV upload and REST API
- Override workflow with audit trail
- Planning dashboard with KPIs (fill rate, inventory turns, MAPE, stockout rate)

### Should-Have (v1.1)
- Scenario modelling for supply disruptions and demand shocks
- Supplier performance tracking (on-time delivery, lead time variance)
- Explainability layer (SHAP values) for forecast drivers
- Supplier risk scoring from structured data
- Natural-language query interface for ad-hoc analysis

### Nice-to-Have (Backlog)
- LLM-generated S&OP narrative commentary
- Supplier risk from unstructured data (news, ESG, geopolitical)
- Agentic disruption response with auto-recommendations
- Multi-enterprise data sharing for trading partner collaboration
- Sustainability/emissions tracking integrated with planning

## Technology Stack

**Backend**: Python or Go, scikit-learn + LightGBM for ML  
**Forecasting**: LSTM/transformer models (PyTorch or TensorFlow)  
**Supplier Risk**: LLM integration for document parsing  
**ERP Integration**: REST APIs for SAP, Oracle, Microsoft  
**Frontend**: React, TypeScript  
**Licensing**: MIT or Apache 2.0 (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–4): Demand forecasting + inventory planning + basic dashboards
2. **Feature Expansion** (months 5–8): Supplier tracking, scenario modeling, explainability
3. **AI Features** (months 9–12): Supplier risk scoring, S&OP narratives, agentic disruption
4. **Monetization**: Open-source core + managed cloud tier (free–$2K/month), enterprise support ($5K+/month)

## Why This Matters

- **Mid-market pricing gap**: Blue Yonder and Kinaxis start at $100K+/year. frePPLe is the only open-source option but lacks modern ML. This platform bridges the gap.
- **Modern ML/DL forecasting is proven to outperform statistical methods**: LSTM and temporal fusion transformers consistently beat ARIMA and exponential smoothing. Accessibility to mid-market represents competitive advantage.
- **Supplier risk + disruption detection is nascent**: CSDDD/LkSG regulations are creating demand for supplier due diligence. No supply chain tool addresses this fully. AI-native approach would differentiate.
- **S&OP narrative generation is manual**: Supply chain teams spend 10+ hours/week building variance explanations. AI automation directly addresses this bottleneck
- **EU regulatory tailwind**: German Supply Chain Due Diligence Act (LkSG) and EU Corporate Sustainability Due Diligence Directive (CSDDD) create compliance demand for supplier analytics

## Success Metrics

- **Year 1**: 300+ active deployments, $250K ARR from cloud + support contracts
- **Year 2**: 1,000+ active deployments, $1.5M ARR; featured in G2 Leaders
- **Year 3**: 3,000+ active deployments, $4M+ ARR; adopted by 150+ mid-market manufacturers/distributors
