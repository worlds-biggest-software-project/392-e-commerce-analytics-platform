# E-Commerce Analytics Platform — Feature & Functionality Survey

> Candidate #392 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Triple Whale | SaaS | Commercial | https://www.triplewhale.com |
| Glew | SaaS | Commercial | https://glew.io |
| Mixpanel | SaaS | Commercial (free tier) | https://mixpanel.com |
| Improvado | SaaS | Commercial (enterprise) | https://improvado.io |
| Google Meridian | Open source | Apache 2.0 | https://github.com/google/meridian |

## Feature Analysis by Solution

### Triple Whale

**Core features**
- First-party pixel capturing purchase data and tying it back to specific ad clicks with high fidelity in a cookieless environment
- Attribution dashboard comparing multiple attribution models (first-click, last-click, linear, data-driven) side-by-side
- Blended ROAS and contribution margin dashboard across all paid channels: Meta, Google, TikTok, Pinterest
- Total Impact Model: probabilistic attribution blending pixel data with platform-reported data to correct for under-reporting
- Creative analytics: per-ad-creative performance ranked by ROAS, conversion volume, and thumb-stop rate

**Differentiating features**
- Used by more than 50,000 e-commerce brands as of 2026 — large installed base in DTC segment
- Triplewhale AI (Moby): conversational analytics interface answering natural-language questions about store performance
- Creative Cockpit: centralised library of all ad creatives with performance metrics attached, reducing creative testing time

**UX patterns**
- Summary dashboard with configurable metric tiles for daily P&L snapshot
- Channel attribution waterfall view with period-over-period comparison
- AI-generated daily email digest summarising store performance trends

**Integration points**
- Shopify (primary), WooCommerce, and BigCommerce platform connectors
- Meta Ads, Google Ads, TikTok Ads, Klaviyo, and major attribution sources
- Looker Studio and Google Sheets data export

**Known gaps**
- Primarily DTC-focused; limited support for B2B commerce or marketplace-channel attribution
- Inventory-performance analytics not included; demand-supply linkage requires separate tools
- Data model is proprietary; limited SQL access for custom queries

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Glew

**Core features**
- Customer journey analytics: stitched session-to-purchase paths across acquisition channels and visits
- Trend analysis, cohort analysis, retention curves, and multi-touch attribution reports in one platform
- Inventory performance module tracking sell-through rates, days-of-stock, and margin by product
- Customer segmentation: high-LTV, at-risk, and lapsed customer cohorts with exportable lists

**Differentiating features**
- Rare combination of demand-side (marketing attribution) and supply-side (inventory performance) analytics in one platform
- Finance data integration: pulls gross margin and cost-of-goods from accounting systems to produce true profitability reports

**UX patterns**
- Pre-built report library covering AOV trends, customer retention, and channel performance
- Automated weekly performance digest emailed to stakeholders
- Customer profile drill-down showing full purchase history and predicted LTV

**Integration points**
- Shopify, WooCommerce, BigCommerce, and Amazon Seller Central
- Google Ads, Facebook Ads, and email marketing platforms
- QuickBooks and Xero for margin data

**Known gaps**
- Less real-time than Triple Whale; data refresh can lag by hours for some sources
- User interface less modern than newer DTC analytics tools
- Limited creative-level analytics for paid media optimisation

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Mixpanel

**Core features**
- Behavioural event analytics with configurable funnels, retention curves, and cohort analysis
- Pre-built e-commerce reports: product conversion funnel, cart-abandonment analysis, and revenue attribution by acquisition source
- Boards: shared dashboards pinning key metrics with configurable date ranges and filters
- User-level event stream for debugging individual shopper journeys

**Differentiating features**
- Real-time data ingestion; events queryable within seconds of occurrence
- A/B test integration for connecting checkout funnel changes to conversion impacts
- Warehouse connector (Snowflake, BigQuery, Redshift) for bidirectional data exchange

**UX patterns**
- Drag-and-drop report builder requiring no SQL for standard analyses
- Signal detection surfacing metric anomalies without manual monitoring
- Slack and email alerts on metric threshold breaches

**Integration points**
- JavaScript, Python, Swift, Kotlin, and React Native SDKs for first-party event capture
- Segment and Rudderstack as CDP intermediaries
- Warehouse connectors for bidirectional sync

**Known gaps**
- Not purpose-built for e-commerce; inventory and supply chain analytics require separate tooling
- No native paid-media attribution; ad spend data requires manual integration
- Cost can escalate rapidly at high monthly tracked user volumes

**Licence / IP notes**
- Proprietary SaaS. No open-source components. SDKs are open-source (Apache 2.0).

---

### Improvado

**Core features**
- Marketing data aggregation from 500+ ad channels, CRMs, and e-commerce platforms into a unified data model
- Multi-touch attribution across paid, organic, email, and social channels
- Data transformation layer normalising inconsistent schemas from different ad platforms
- Pre-built dashboards for CMO-level channel performance reporting

**Differentiating features**
- Broadest connector library in the marketing data aggregation segment
- Enterprise-grade data governance: field-level access control and audit logging
- Custom metrics and KPI builder without requiring SQL expertise

**UX patterns**
- Template gallery with pre-built dashboards deployable to Looker Studio, Tableau, or Power BI
- Automated anomaly detection alerting on spend and performance outliers

**Integration points**
- Google Ads, Meta Ads, TikTok Ads, and 500+ other sources
- Snowflake, BigQuery, Redshift, and Azure Synapse as data warehouse destinations
- Looker, Tableau, Power BI, and Google Looker Studio as visualisation layers

**Known gaps**
- Primarily a marketing-data ETL tool; behavioural analytics and session replay require additional tools
- Expensive for smaller merchants; positioned at enterprise retailers with large ad budgets
- No built-in inventory or supply chain analytics

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-touch attribution across paid and organic channels with model comparison
- Conversion funnel analysis from traffic source to purchase, segmented by device and channel
- Customer cohort retention showing repeat-purchase rates by acquisition period
- Integration with Shopify, WooCommerce, and BigCommerce as primary commerce platforms
- First-party pixel or server-side event capture for cookieless attribution accuracy

### Differentiating Features
- Inventory performance analytics linking stock levels to marketing spend decisions
- Creative-level paid media analytics ranking ad assets by ROAS and engagement
- Probabilistic attribution models correcting for platform under-reporting in a cookieless environment
- AI conversational analytics interface for non-technical merchant queries
- True profitability reporting incorporating cost-of-goods and ad spend in a single margin view

### Underserved Areas / Opportunities
- Real-time inventory-to-marketing integration: automatically pausing ads for out-of-stock SKUs
- Cross-device identity resolution without third-party cookies using first-party login graphs
- Predictive demand forecasting at SKU level integrated with reorder alerting
- Marketplace attribution: unified view spanning owned store, Amazon, and social commerce with consistent margin calculation

### AI-Augmentation Candidates
- Natural-language store performance queries ("why did ROAS drop on Meta last week?")
- Automated creative performance analysis and underperforming-ad pausing recommendations
- Demand forecasting models trained on historical sell-through rates and external signals
- Personalised product recommendation engine fed by behavioural analytics data

## Legal & IP Summary

All purpose-built e-commerce analytics platforms in this space (Triple Whale, Glew, Improvado) are proprietary SaaS tools. Google Meridian, an open-source media mix modelling framework (Apache 2.0), provides a legally clear foundation for channel-level attribution modelling. Mixpanel's SDKs are Apache 2.0. The core analytical techniques (cohort analysis, multi-touch attribution modelling, funnel analysis) are established statistical methods with no patent encumbrances. A new entrant building on first-party event tracking, open-source transformation tools (dbt, Apache 2.0), and standard BI layers (Metabase, Apache Superset) faces no legal barriers. The primary competitive moat in this space is data network effects from the installed connector library and the quality of the probabilistic attribution model, neither of which is patentable.

## Recommended Feature Scope

**Must-have (MVP)**:
- First-party pixel or server-side event capture for cookieless conversion tracking
- Native connectors for Shopify, WooCommerce, and BigCommerce order and product data
- Multi-touch attribution dashboard with at least last-click, first-click, and linear models
- Conversion funnel analysis segmented by traffic source, device, and product category
- Customer cohort retention: repeat-purchase rate and LTV by acquisition month
- Core e-commerce metrics: GMV, AOV, conversion rate, and CAC by channel

**Should-have (v1.1)**:
- Creative-level paid media analytics linking ad assets to conversion outcomes
- Inventory performance module: sell-through rate, days-of-stock, and stockout alerting
- Customer segmentation: high-LTV, at-risk, and lapsed cohorts with exportable lists
- Data warehouse export (Snowflake, BigQuery) for custom SQL analysis
- Probabilistic attribution model correcting for platform under-reporting

**Nice-to-have (backlog)**:
- AI natural-language analytics interface for non-technical users
- Real-time inventory-to-ad-spend integration pausing campaigns for out-of-stock SKUs
- Predictive demand forecasting at SKU level with reorder point alerting
- Cross-device identity resolution using first-party login graph
- Marketplace attribution unifying owned store, Amazon, and social commerce in one model
