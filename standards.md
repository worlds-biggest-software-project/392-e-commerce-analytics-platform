# Standards & API Reference

> Project: E-Commerce Analytics Platform · Generated: 2026-05-06

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
URL: https://www.iso.org/standard/27001
Defines requirements for establishing, implementing, maintaining, and continually improving an information security management system. Relevant for any analytics platform storing customer behavioural data at scale, particularly when handling purchase histories and PII.

**ISO/IEC 27701:2019 — Privacy Information Management**
URL: https://www.iso.org/standard/71670.html
Extension to ISO/IEC 27001 that specifies requirements for establishing a Privacy Information Management System (PIMS). Directly applicable when an analytics platform processes personal data (customer profiles, order histories, browsing behaviour) across multiple jurisdictions.

**ISO/TR 16320-1:2026 — Smart Contract-Based B2B Electronic Transaction Execution**
URL: https://www.iso.org/standard/76553.html
Provides a reference framework for smart contract-based B2B electronic transaction execution and verification. Relevant to analytics platforms that may incorporate blockchain-based transaction ledgers or need to verify cross-platform order provenance.

**ISO/IEC 20151 — Data Spaces (2026 revision)**
URL: https://internationaldataspaces.org/standardization-in-data-spaces-where-iso-iec-20151-stands-in-2026/
Governs the interoperability of data sharing between organisations in data spaces. Relevant for analytics platforms that participate in federated retail data ecosystems or multi-brand attribution networks.

**ISO/IEC 19583-26:2026 — Metadata Registries in W3C XML Schema**
URL: https://www.iso.org/standard/19583-26
Specifies how to represent metadata registry data models in W3C XML Schema, enabling machine-readable metadata exchange. Relevant for analytics platforms that expose or consume catalogue and product metadata from retail data registries.

---

### W3C & IETF Standards

**W3C Attribution Reporting API (WICG)**
URL: https://github.com/WICG/attribution-reporting-api
A privacy-preserving browser API for measuring ad conversions without third-party cookies. Event-level reports link ad interactions (clicks, views) to coarse conversion data using noise and delay to protect user privacy. Critical for any e-commerce analytics platform building cookieless attribution. Note: Google announced retirement of this API in Chrome following ecosystem feedback, but W3C community groups continue standardisation work.

**W3C Topics API (Privacy Sandbox)**
URL: https://privacysandbox.google.com
Enables interest-based cohort signals derived from browsing history without exposing individual sites visited. Relevant for platforms building audience segmentation on privacy-safe signals rather than third-party cookies.

**RFC 2706 — ECML v1: Field Names for E-Commerce**
URL: https://datatracker.ietf.org/doc/html/rfc2706
Defines the Electronic Commerce Modelling Language version 1, a standard set of field names allowing electronic wallets to auto-fill merchant checkout forms. Foundation for understanding standardised e-commerce data field naming conventions.

**RFC 3106 — ECML v1.1: Field Specifications for E-Commerce**
URL: https://datatracker.ietf.org/doc/html/rfc3106
Updates ECML v1 with additional field specifications. Relevant as a historical baseline for understanding how the industry attempted to standardise e-commerce data fields before modern JSON-based event schemas.

**RFC 7231 — HTTP/1.1 Semantics and Content**
URL: https://datatracker.ietf.org/doc/html/rfc7231
Defines HTTP methods, status codes, and headers. The foundational standard for all REST APIs that an analytics platform will expose or consume.

**RFC 8288 — Web Linking**
URL: https://datatracker.ietf.org/doc/html/rfc8288
Defines the `Link` header and link relations for hypermedia APIs. Relevant when designing paginated analytics API responses with HATEOAS-style navigation.

**RFC 6749 — OAuth 2.0 Authorization Framework**
URL: https://datatracker.ietf.org/doc/html/rfc6749
The standard authorisation framework used universally across analytics and commerce APIs. Analytics platforms integrating with Shopify, Meta Ads, Google Ads, and similar sources must implement OAuth 2.0 flows.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1.0**
URL: https://spec.openapis.org/oas/v3.1.0.html
The industry-standard format for describing RESTful APIs in machine-readable YAML or JSON. Fully compatible with JSON Schema. An analytics platform should publish its own API surface as an OpenAPI document to enable automated SDK generation and partner integrations.

**OpenAPI Specification v3.2.0**
URL: https://spec.openapis.org/oas/v3.2.0.html
The latest revision, adding refinements to JSON Schema alignment and webhook description. Recommended target version for new API designs.

**JSON Schema (Draft 2020-12)**
URL: https://json-schema.org/specification.html
The standard for describing the structure and validation rules of JSON data. Used for defining e-commerce event payloads (orders, products, sessions) and API request/response schemas. Natively supported in OpenAPI 3.1.

**Segment Spec (Twilio)**
URL: https://segment.com/docs/connections/spec/
A widely-adopted de-facto standard for analytics event schemas across web, mobile, and server-side contexts. Defines six canonical call types (Identify, Track, Page, Screen, Group, Alias) and a versioned e-commerce event spec (Order Completed, Product Viewed, Cart Viewed, Checkout Step Viewed, etc.). Platforms that conform to the Segment spec gain access to Segment's 400+ downstream integrations.

**Segment E-Commerce Spec v2**
URL: https://segment.com/docs/spec/ecommerce/v2/
A detailed specification for e-commerce event naming and property conventions within the Segment ecosystem. Covers the full shopper journey: product browsing, promotions, cart management, checkout steps, and order completion. Widely used as a reference model even by platforms not built on Segment.

**GA4 Measurement Protocol**
URL: https://developers.google.com/analytics/devguides/collection/protocol/ga4
Google's server-to-server event ingestion protocol for Google Analytics 4. Defines a standard HTTP POST API for sending e-commerce events (purchase, refund, view_item, add_to_cart, begin_checkout, etc.) with a structured items array. Maximum 25 events per request; events must be received within 48 hours for proper joining with client-side data.

**RudderStack Ecommerce Events Spec**
URL: https://www.rudderstack.com/docs/event-spec/ecommerce-events-spec/
An open-source alternative to the Segment spec, defining e-commerce event schemas for a customer data pipeline platform. Largely mirrors Segment's e-commerce spec and is a useful reference for implementing interoperable event models.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
URL: https://datatracker.ietf.org/doc/html/rfc6749
Industry-standard authorisation framework. Required for integrating with Shopify Admin API, Meta Ads API, Google Ads API, and other commerce and advertising platforms. Analytics platforms must implement appropriate flows (authorization code for user-facing connectors, client credentials for server-to-server).

**OpenID Connect 1.0**
URL: https://openid.net/connect/
Authentication layer built on top of OAuth 2.0. Enables single sign-on for analytics platform users and supports federated identity for enterprise customers. Widely supported by identity providers including Okta, Auth0, and Google Workspace.

**FAPI 2.0 Security Profile**
URL: https://openid.net/specs/fapi-security-profile-2_0-final.html
The Financial-grade API security profile, an extension of OAuth 2.0 designed for high-security API access. Relevant if an analytics platform processes payment or financial data requiring stronger token binding and sender-constrained access tokens.

**PCI DSS v4.0 (Payment Card Industry Data Security Standard)**
URL: https://www.pcisecuritystandards.org/
Mandatory compliance standard for platforms that process, store, or transmit payment card data. Requirements 6.4.3 and 11.6.1 (effective 31 March 2025) directly target e-commerce: script authorisation on payment pages and change-and-tamper detection for HTTP headers and script content. Analytics platforms must not capture raw payment card numbers; they must operate upstream of the payment capture step.

**OWASP Top 10 (2021)**
URL: https://owasp.org/www-project-top-ten/
The authoritative reference for web application security risks. Particularly relevant for analytics platforms exposing public APIs and embedding tracking scripts on merchant storefronts: broken access control, injection, insecure design, and security misconfiguration are top risks in this context.

**GDPR (General Data Protection Regulation)**
URL: https://gdpr.eu/
EU law governing the processing of personal data. Analytics platforms tracking EU shoppers must implement: lawful basis (typically consent) before firing analytics cookies, data minimisation, right to erasure, and data processing agreements with merchant customers. Failure to obtain consent before firing analytics pixels violates GDPR; fines reached €1.2 billion in 2024.

**CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)**
URL: https://cppa.ca.gov/
California law requiring transparency about data collection and honouring "Do Not Sell or Share" requests. New CPRA regulations effective January 1, 2026 require comprehensive privacy risk assessments before processing that presents significant risk to consumer privacy. Analytics platforms must support opt-out mechanisms and data deletion requests.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant for AI-native analytics platforms that expose data query capabilities to LLM-based agents. An e-commerce analytics MCP server could expose tools such as:
- `query_sales_metrics` — retrieve revenue, order volume, and conversion data for a date range
- `get_funnel_report` — return funnel drop-off analysis by traffic source
- `get_product_performance` — return sell-through, margin, and view-to-purchase rates by product

**MCP Specification**
URL: https://modelcontextprotocol.io/specification
Defines the protocol for connecting AI assistants to data sources and tools via a standardised server interface. An analytics platform exposing an MCP server would allow merchants to query their analytics data through natural language via Claude, Cursor, and other MCP-compatible AI clients.

---

## Similar Products — Developer Documentation & APIs

### Google Analytics 4 (GA4)

- **Description:** Google's flagship analytics platform with built-in e-commerce event tracking and a server-to-server Measurement Protocol.
- **API Documentation:** https://developers.google.com/analytics/devguides/collection/protocol/ga4
- **SDKs/Libraries:** gtag.js (web), Firebase SDK (mobile), Google Analytics Data API client libraries (Python, Node.js, Java, Go, PHP, Ruby, .NET) — https://developers.google.com/analytics/devguides/reporting/data/v1/libraries
- **Developer Guide:** https://developers.google.com/analytics/devguides/collection/ga4
- **Standards:** REST/JSON, Measurement Protocol (proprietary HTTP POST), OpenAPI-described Data API
- **Authentication:** API key (Measurement Protocol), OAuth 2.0 / Service Account (Data Reporting API)

---

### Segment (Twilio)

- **Description:** Customer data platform and analytics event routing infrastructure. Defines the most widely adopted e-commerce event specification in the industry.
- **API Documentation:** https://docs.segmentapis.com/ (Public API) and https://segment.com/docs/connections/sources/catalog/libraries/server/http-api/ (HTTP Tracking API)
- **SDKs/Libraries:** Analytics.js (web), analytics-node, analytics-python, analytics-java, analytics-go, analytics-ios, analytics-android — https://segment.com/docs/connections/sources/
- **Developer Guide:** https://segment.com/docs/
- **Standards:** REST/JSON; proprietary event spec (Identify/Track/Page/Screen/Group/Alias); batch endpoint accepts up to 500 KB per request, 32 KB per event
- **Authentication:** Write Key (source-level API key); Public API uses OAuth 2.0

---

### Shopify Admin API

- **Description:** Shopify's primary developer API for accessing orders, products, customers, inventory, and analytics data from Shopify stores.
- **API Documentation:** https://shopify.dev/docs/api/admin-graphql/latest (GraphQL) and https://shopify.dev/docs/api/admin-rest (REST, deprecated for new apps)
- **SDKs/Libraries:** shopify-api-node, shopify-api-ruby, shopify-api-php, shopify-api-go — https://shopify.dev/docs/api
- **Developer Guide:** https://shopify.dev/docs/apps/getting-started
- **Standards:** GraphQL (primary); REST/JSON (legacy); ShopifyQL for analytics queries (https://shopify.dev/docs/api/shopifyql)
- **Authentication:** OAuth 2.0 (public apps); Admin API access tokens (custom apps)

---

### Mixpanel

- **Description:** Product analytics platform with event-based tracking, funnel analysis, retention cohorts, and user journey visualisation. Widely used for e-commerce behavioural analytics.
- **API Documentation:** https://developer.mixpanel.com/reference/overview
- **SDKs/Libraries:** mixpanel-js, mixpanel-python, mixpanel-node, mixpanel-ruby, mixpanel-java, mixpanel-swift, mixpanel-android — https://docs.mixpanel.com/docs/tracking-methods/sdks
- **Developer Guide:** https://docs.mixpanel.com/
- **Standards:** REST/JSON; proprietary event ingestion API; export API returns newline-delimited JSON
- **Authentication:** Project token (client-side); Service Account + secret (server-side and export APIs)

---

### Amplitude

- **Description:** Product analytics and behavioural intelligence platform. Offers event tracking, cohort analysis, and predictive analytics for e-commerce and SaaS use cases.
- **API Documentation:** https://amplitude.com/docs (HTTP API v2, Batch API, Identify API, Revenue API)
- **SDKs/Libraries:** @amplitude/analytics-browser, @amplitude/analytics-node, amplitude-python, amplitude-android, amplitude-ios — https://amplitude.com/docs/sdks
- **Developer Guide:** https://amplitude.com/docs/get-started
- **Standards:** REST/JSON; Batch API supports up to 2,000 events per request; SDK API surface closely mirrors Segment's
- **Authentication:** API key (event ingestion); secret key (management and export APIs)

---

### Klaviyo

- **Description:** Marketing automation and analytics platform purpose-built for e-commerce. Provides metrics, reporting, and event APIs tuned to email/SMS campaign performance and customer lifecycle analytics.
- **API Documentation:** https://developers.klaviyo.com/en (REST API, current revision 2025-04-15)
- **SDKs/Libraries:** klaviyo-python, klaviyo-php, klaviyo-ruby, klaviyo-node — https://developers.klaviyo.com/en/docs/sdk_overview
- **Developer Guide:** https://developers.klaviyo.com/en/docs/getting_started
- **Standards:** REST/JSON; OpenAPI-described; versioned by date header (`revision: YYYY-MM-DD`)
- **Authentication:** `Authorization: Klaviyo-API-Key <key>` header

---

### Triple Whale

- **Description:** E-commerce analytics operating system for Shopify brands. Provides first-party attribution, profitability tracking, creative performance, and customer lifetime value analytics, with a Data-In API for custom data ingestion.
- **API Documentation:** https://www.triplewhale.com/api
- **SDKs/Libraries:** JavaScript pixel SDK (first-party tracking); REST API for data ingestion and export
- **Developer Guide:** https://www.triplewhale.com/api
- **Standards:** REST/JSON; proprietary first-party pixel spec; webhook-based event delivery
- **Authentication:** API key

---

### Looker (Google Cloud)

- **Description:** Business intelligence platform with an API-first architecture. Used by enterprise e-commerce operators to build embedded analytics dashboards and expose consistent metrics via LookML semantic models.
- **API Documentation:** https://docs.cloud.google.com/looker/docs/reference/looker-api/latest/overview
- **SDKs/Libraries:** looker-sdk (Python, Node.js, Ruby, Go, Kotlin, Swift) — https://github.com/looker-open-source/sdk-codegen
- **Developer Guide:** https://docs.cloud.google.com/looker/docs
- **Standards:** REST/JSON; Open SQL Interface (JDBC); OpenAPI-described; supports Looker Conversational Analytics API (GA 2026)
- **Authentication:** OAuth 2.0 / API3 credentials (client ID + secret)

---

## Notes

**Cookieless tracking transition:** The industry is mid-transition away from third-party cookies toward first-party pixel data, server-side tracking, and privacy-preserving APIs. Analytics platforms should plan for a dual-mode architecture that works with both legacy cookie-based signals and first-party or server-side alternatives.

**Segment spec as de-facto standard:** While not formally governed by an SDO, the Segment e-commerce event spec (v2) functions as the closest thing to an industry-standard event schema. Platforms conforming to it gain broad ecosystem compatibility.

**GDPR consent requirements (2026):** European data protection authorities have confirmed that legitimate interest does not justify analytics or marketing cookies; explicit opt-in consent is required. Analytics platforms must design consent-aware data collection from the ground up.

**PCI DSS scope management:** Analytics platforms must be carefully scoped to avoid touching payment card data. Server-side event collection should be implemented downstream of payment processors, and scripts on checkout pages fall under PCI DSS Requirements 6.4.3 and 11.6.1 as of March 2025.

**ShopifyQL:** Shopify's proprietary analytics query language (ShopifyQL) is queryable via the GraphQL Admin API and represents an emerging pattern of commerce-platform-native analytics query interfaces that may influence future standards.
