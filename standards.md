# Standards & API Reference

> Project: Retail Store Analytics · Generated: 2026-05-06

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Relevance: The baseline information security management standard for any platform handling retail operational data including POS transaction records, foot traffic logs, and staff scheduling data. Certification provides assurance to enterprise retail clients that data is managed and protected appropriately.

**ISO/IEC 27701:2019 — Privacy Information Management**
- URL: https://www.iso.org/standard/71670.html
- Relevance: Extension to ISO 27001 addressing privacy information management. Directly relevant for in-store analytics platforms capturing anonymised visitor data and processing employee data within GDPR and CCPA compliance frameworks.

**ISO 8583 — Financial Transaction Card Messages**
- URL: https://www.iso.org/standard/79451.html
- Relevance: Governs the message format for POS payment card transactions. A retail analytics platform ingesting real-time transaction data from POS systems may encounter ISO 8583-format message streams from payment terminals, particularly in enterprise implementations.

**ISO 31000:2018 — Risk Management**
- URL: https://www.iso.org/standard/65694.html
- Relevance: Provides a framework for risk management applicable when assessing operational risks that the analytics platform is designed to surface and mitigate (e.g. over/understaffing risk, conversion loss risk).

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Relevance: Foundational HTTP standard governing the REST API interfaces that all major retail analytics platforms (Placer.ai, V-Count BoostBI, Lightspeed, Square) use for data exchange. Any REST API built for this platform must conform.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Relevance: The authentication standard used by Square, Shopify, Lightspeed, and Placer.ai for delegated API access. The platform must implement OAuth 2.0 to authenticate against POS and analytics data sources on behalf of retailer accounts.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Relevance: Governs hypermedia linking in REST APIs, relevant for pagination and resource discovery in analytics data endpoints.

**W3C SPARQL / RDF (optional)**
- URL: https://www.w3.org/TR/sparql11-query/
- Relevance: If the platform adopts a semantic data model for retail product taxonomies or location data, SPARQL provides a standardised query mechanism. Lower-priority, but worth noting for future extensibility.

---

### Data Model & API Specifications

**ARTS Operational Data Model (NRF/ARTS)**
- URL: https://www.omg.org/retail-depository/arts-odm-73/
- Relevance: The Association for Retail Technology Standards (a division of NRF) maintains the ARTS Operational Data Model — the canonical retail industry data model covering transactions, products, customers, staff, and locations. Aligning the platform's data schema to ARTS ODM version 7.3 ensures interoperability with enterprise retail ERP and POS systems.

**ARTS IXRetail XML Schema**
- URL: https://xml.coverpages.org/ixRetail.html
- Relevance: ARTS XML (formerly IXRetail) builds on the ARTS Data Model to provide standard XML message sets for application-to-application integration within retail enterprises. Relevant for any EDI-style integration with legacy retail systems.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Relevance: The standard for documenting REST APIs. The platform's own API should be documented using OpenAPI 3.1 to enable automatic SDK generation and developer tooling integration.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12
- Relevance: Used for validating request and response payloads in the platform's REST API, and for defining the schema of event messages from sensor hardware integrations.

**Apache Avro / Parquet**
- URLs: https://avro.apache.org/ / https://parquet.apache.org/
- Relevance: When handling high-volume foot traffic event streams (sensor data at 10 samples/second per sensor, per store), columnar formats like Parquet and schema-defined serialisation like Avro are standard in data engineering pipelines (Apache Airflow + dbt contexts).

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) — see IETF section above**

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Relevance: Identity layer on top of OAuth 2.0; required if the platform supports single sign-on (SSO) for enterprise retail clients using Azure AD, Okta, or Google Workspace as identity providers.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Relevance: Defines the most critical API security risks. Any public API exposed by the platform must be assessed against OWASP API Security Top 10, particularly for broken object-level authorisation (BOLA) risks in multi-tenant store data access.

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- Relevance: Widely adopted US federal and enterprise security framework (Identify → Protect → Detect → Respond → Recover). Relevant for platform security posture documentation when selling to US enterprise retailers.

**NIST SP 800-213 — IoT Device Cybersecurity Guidance**
- URL: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-213A.pdf
- Relevance: Governs security requirements for IoT devices (people-counting sensors, WiFi access points used for analytics). Relevant when the platform connects to or manages sensor hardware.

---

### Privacy & Regulatory Frameworks

**GDPR (EU General Data Protection Regulation)**
- URL: https://gdpr.eu/
- Relevance: Applies to any in-store analytics deployment in the EU. People-counting systems that capture video must demonstrate anonymisation at source (no personally identifiable images transmitted or stored). GDPR's proportionality principle means camera-based counting must achieve the same outcome as 3D depth sensing without identifiable images. Consent and data retention policies must be explicitly defined.

**EU AI Act (Regulation 2024/1689)**
- URL: https://artificialintelligenceact.eu/
- Relevance: Enters full enforcement for Annex III high-risk systems on 2 August 2026. Retail video analytics using biometric identification or demographic inference (gender/age detection) may be classified as high-risk AI, requiring conformity assessments, technical documentation, CE marking, and EU database registration. Anonymised, non-biometric people counting is lower risk but must still comply with transparency obligations (Article 50) from August 2026.

**CCPA (California Consumer Privacy Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- Relevance: Applies to US retail deployments capturing consumer location or behavioural data. Relevant for Placer.ai-style mobile-panel data integrations and any WiFi-based visitor tracking that captures device identifiers.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Relevance: If the platform exposes retail analytics data to AI agents (e.g. a store manager asking questions in natural language via Claude or similar), an MCP server would allow AI assistants to query foot traffic, basket, and scheduling data programmatically. An MCP server exposing store metrics, trend queries, and scheduling recommendations would be a natural interface for an AI-native version of this platform.

---

## Similar Products — Developer Documentation & APIs

### Square POS API

- **Description:** Square provides a full-featured REST API for payments, inventory, orders, customers, and reporting. It is the most widely used POS API for SMB and mid-market retail, and a primary data source for basket and transaction analytics.
- **API Documentation:** https://developer.squareup.com/reference/square
- **SDKs/Libraries:** Python, Ruby, Node.js, PHP, Java, .NET, Go — https://developer.squareup.com/docs/sdks
- **Developer Guide:** https://developer.squareup.com/docs
- **Standards:** REST/JSON, OpenAPI; OAuth 2.0 for authentication
- **Authentication:** OAuth 2.0

---

### Shopify GraphQL Admin API

- **Description:** Shopify's primary developer API for accessing order, product, customer, and analytics data from Shopify stores. ShopifyQL is a commerce-specific query language layered on top of GraphQL for building custom analytics and reports.
- **API Documentation:** https://shopify.dev/docs/api
- **SDKs/Libraries:** Ruby, Node.js, PHP, Go, .NET — https://shopify.dev/docs/api
- **Developer Guide:** https://shopify.dev/docs/api (ShopifyQL reference: https://shopify.dev/docs/api/shopifyql)
- **Standards:** GraphQL, REST; ShopifyQL for analytics queries
- **Authentication:** OAuth 2.0

---

### Lightspeed Retail X-Series API

- **Description:** Lightspeed's REST API for its X-Series retail POS, enabling access to sales, inventory, customer, and employee data. Webhook support for real-time event-driven integration. Primary POS integration target for basket analysis.
- **API Documentation:** https://x-series-api.lightspeedhq.com/
- **SDKs/Libraries:** No official SDK; community Laravel package available — https://github.com/timothydc/laravel-lightspeed-retail-api
- **Developer Guide:** https://x-series-api.lightspeedhq.com/
- **Standards:** REST/JSON; webhook payloads in JSON over HTTP POST
- **Authentication:** OAuth 2.0; API team contact: x-series.api@lightspeedhq.com

---

### Placer.ai REST API (PAPI)

- **Description:** Placer.ai's API provides programmatic access to aggregated foot traffic data, demographic breakdowns, trade area polygons, and competitor benchmarking for retail locations. Designed for BI integration and custom analytics pipelines.
- **API Documentation:** https://docs.placer.ai/
- **SDKs/Libraries:** Language-agnostic REST API; multi-language guide available in developer hub
- **Developer Guide:** https://docs.placer.ai/reference/getting-started
- **Standards:** REST/JSON; automatic daily data exports available
- **Authentication:** API Key (token-based)

---

### Sensormatic ShopperTrak API

- **Description:** Two-tier API for ShopperTrak Managed Service clients: (1) Flash Traffic API providing 15-minute interval foot traffic counts for near-real-time POS and workforce integration; (2) Enterprise Daily Traffic API for previous-day store-level counts. Restricted to contracted clients.
- **API Documentation:** https://portal.shoppertrak.com/st_web/help/help_rta.html (client portal)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** Available through Sensormatic enterprise engagement
- **Standards:** REST (implied); JSON data format
- **Authentication:** Enterprise client credentials (not publicly documented)

---

### V-Count BoostBI Open API

- **Description:** V-Count's BoostBI platform exposes an open REST API allowing retailers to integrate people-counting data (foot traffic, occupancy, gender/age counts, zone metrics) with their existing BI tools and custom dashboards.
- **API Documentation:** Not publicly listed; available through V-Count client onboarding
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** Available through V-Count client onboarding
- **Standards:** REST/JSON
- **Authentication:** Not publicly documented

---

### Aislelabs Advanced Visualisation API

- **Description:** Aislelabs provides an API for embedding and accessing WiFi analytics data — dwell time, zone visits, occupancy, and visitor metrics — in external dashboards and applications. Designed for shopping centres and retailers integrating analytics into property management or tenant-facing portals.
- **API Documentation:** https://www.aislelabs.com/features/occupancy/ (feature overview; full docs via client portal)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** Available through Aislelabs client onboarding
- **Standards:** REST (implied)
- **Authentication:** Not publicly documented

---

## Notes

**Sensor hardware APIs are largely proprietary and client-gated.** The leading people-counting vendors (RetailNext, Sensormatic, V-Count) do not publish open developer documentation. Access requires enterprise contracts. An open-source retail analytics platform would need to design a generic sensor data ingestion interface (webhook receiver or polling agent) that can accept data in common formats (JSON over HTTP, CSV flat files) regardless of sensor vendor.

**ARTS ODM is the canonical retail data model** but is rarely implemented in SMB SaaS products. Aligning to ARTS terminology (Transaction, WorkPeriod, Workstation, RetailStore) would accelerate enterprise adoption.

**The EU AI Act deadline of August 2026** creates an immediate compliance opportunity: retailers using older camera-based analytics systems that capture identifiable video will need compliant replacements. A privacy-by-design, non-biometric analytics platform built for this moment has a narrow but real regulatory tailwind.

**MCP integration is an emerging opportunity.** No surveyed platform currently exposes retail analytics data via Model Context Protocol. An MCP server for store analytics would allow AI assistants to answer operational questions in natural language — a differentiator that does not require building a full NLP UI layer.
