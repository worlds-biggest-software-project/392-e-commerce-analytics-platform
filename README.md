# E-Commerce Analytics Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A unified, AI-native analytics platform for online retailers — connecting shopper behaviour, marketing attribution, conversion funnels, and inventory performance in a single queryable model.

Online retailers generate rich behavioural data across product discovery, cart, checkout, and post-purchase stages, but most analytics setups stop at page-view aggregates or fragment the picture across siloed tools. This project consolidates those signals into one platform so merchants can connect early browse sessions to delayed conversions, evaluate channel quality by repeat-purchase patterns, and detect when a product is over-advertised while running out of stock.

---

## Why E-Commerce Analytics Platform?

- **Demand and supply data live in different tools.** Triple Whale and Improvado focus on marketing attribution; inventory-performance analytics is "not included" or absent, forcing merchants to buy a second platform for stockout and sell-through visibility.
- **Cookieless attribution is a moving target.** Browser privacy changes have degraded third-party cookie tracking; first-party server-side capture is now required, and 65.7% of marketing leaders cite data integration as the primary barrier to effective measurement.
- **Incumbents are proprietary and expensive.** Triple Whale, Glew, and Improvado are closed SaaS with proprietary data models and limited SQL access; Improvado is positioned at enterprise retailers with large ad budgets.
- **Mixpanel-class behavioural tools are not e-commerce-aware.** They lack native paid-media attribution and inventory analytics, and costs escalate rapidly at high tracked-user volumes.
- **No legal moat blocks an open alternative.** Core techniques (cohort analysis, multi-touch attribution, funnel analysis) are unpatented; open-source building blocks (Google Meridian, dbt, Metabase, Superset) provide a clean foundation.

---

## Key Features

### Attribution and Conversion

- Multi-touch attribution dashboard with last-click, first-click, linear, and probabilistic models
- Conversion funnel analysis segmented by traffic source, device, and product category
- Blended ROAS and contribution margin across paid channels
- First-party pixel and server-side event capture for cookieless tracking

### Customer and Cohort Analytics

- Customer cohort retention: repeat-purchase rate and LTV by acquisition month
- Customer segmentation for high-LTV, at-risk, and lapsed cohorts with exportable lists
- Stitched session-to-purchase journey paths across visits and channels
- Core commerce metrics: GMV, AOV, conversion rate, and CAC by channel

### Inventory and Supply-Side

- Inventory performance module tracking sell-through rate, days-of-stock, and margin by product
- Stockout alerting tied to active marketing campaigns
- Real-time inventory-to-ad-spend integration to pause campaigns for out-of-stock SKUs
- Predictive demand forecasting at SKU level with reorder point alerting

### Creative and Channel Insights

- Creative-level paid media analytics ranking ad assets by ROAS and engagement
- Probabilistic attribution correcting for platform under-reporting
- Marketplace attribution unifying owned store, Amazon, and social commerce
- Native connectors for Shopify, WooCommerce, and BigCommerce

### Data Access and Extensibility

- Data warehouse export to Snowflake and BigQuery for custom SQL analysis
- Integration with Segment and Rudderstack as CDP intermediaries
- BI layer compatibility with Looker, Metabase, and Superset

---

## AI-Native Advantage

A natural-language analytics interface lets non-technical merchants ask questions like "why did ROAS drop on Meta last week?" without writing SQL. AI-driven creative analysis surfaces underperforming ads and recommends pauses, while SKU-level demand forecasting models trained on historical sell-through rates and external signals power reorder alerting. Personalised product recommendations are fed directly by the behavioural analytics layer, closing the loop between measurement and merchandising.

---

## Tech Stack & Deployment

The platform is designed for self-hosted deployment on cloud data warehouses (Snowflake, BigQuery) with dbt as the transformation layer and Apache Kafka for real-time inventory-change streaming. Event ingestion uses first-party server-side capture compatible with Segment and Rudderstack. Forecasting models are built in Python with scikit-learn and Prophet. Channel attribution leverages open-source media mix modelling frameworks such as Google Meridian (Apache 2.0) and Meta Robyn. Connectors for Shopify, BigCommerce, WooCommerce, Amazon Seller Central, and major ad networks are exposed via Airbyte and platform-native APIs.

---

## Market Context

The global e-commerce analytics market is moving toward USD 28.64 billion in 2026. Triple Whale alone serves more than 50,000 e-commerce brands, primarily DTC merchants on Shopify. Primary buyers span DTC operators, multi-channel retailers selling on owned stores plus Amazon and social commerce, and enterprise retail marketing teams currently served by Improvado-class ETL tooling.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
